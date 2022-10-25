---
title: Vorbereitung
category: Steganografie
subcategory: .NET Assemblies / IL Code
order: 10
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 11.8 Kb]({% link /assets/steganodotnet6_src.zip %})

## Worum geht es?

Jede Anwendung enth&auml;lt viele Zeilen, die den Stack leer hinterlassen. Nach diesen Zeilen
kann jeder Code eingef&uuml;gt werden, sofern er den Stack wieder leer zur&uuml;ckl&auml;sst.
Man kann ein paar Werte auf den Stack laden und wieder entfernen, ohne den Programmfluss zu st&ouml;ren.


## Stille Verstecke finden

Werfen wir mal einen Blick auf den IL Assembler Language Code eines Assemblies.
Jede Methode enth&auml;lt Zeilen, die etwas auf den Stack schreiben oder herunter lesen.
Wir k&ouml;nnen nicht immer vorhersagen, was genau bei welcher Zeile auf dem Stack liegt,
darum sollten wir nichts zwischen zwei Zeilen &auml;ndern. Aber es gibt einige Zeilen, von denen
man genau wei&szlig;, was auf dem Stack liegen muss.


Jede Methode enth&auml;lt mindestens eine <code>ret</code> Anweisung.
Wenn die Laufzeitumgebung ein <code>ret</code> erreicht, muss der Stack den R&uuml;ckgabewert und
sonst nichts enthalten. Das hei&szlig;t, vor einer <code>ret</code> Anweisung in einer Methode, die einen
<code>Int32</code> zur&uuml;ckgibt, enth&auml;lt der Stack genau einen <code>Int32</code> Wert.
Wir k&ouml;nnten ihn in einer lokalen Variable speichern, zus&auml;tzlichen Code einf&uuml;gen, der einen leeren Stack hinterl&auml;sst,
und dann den R&uuml;ckgabewert zur&uuml;ck auf den Stack schreiben. Zur Laufzeit w&uuml;rde es niemand bemerken.
Es gibt noch viel mehr solcher Zeilen, zum Beispiel die schlie&szlig;enden Klammern eines <code>.try {</code> und <code>.catch {</code>
Blocks (definitiv leerer Stack!) oder Methodeaufrufe (nur ein Wert von bekanntem Typ auf dem Stack!).
Um dieses Beispiel einfach zu halten, werden wir uns auf <code>void</code>-Methoden konzentrieren
und alle anderen ignorieren.
Wenn eine <code>void</code>-Methode verlassen wird, muss der Stack leer sein, so dass wir uns nicht
mit R&uuml;ckgabewerten aufhalten m&uuml;ssen.

Hier ist der IL Assembler Language Code einer typischen <code>void Dispose()</code> Methode:

<pre>
.method family hidebysig virtual instance void
            Dispose(bool disposing) cil managed
    {
      // Code size       39 (0x27)
      .maxstack  2
      IL_0000:  ldarg.1
      IL_0001:  brfalse.s  IL_0016

      IL_0003:  ldarg.0
      IL_0004:  ldfld      class [System]System.ComponentModel.Container
                  PictureKey.frmMain::components
      IL_0009:  brfalse.s  IL_0016

      IL_000b:  ldarg.0
      IL_000c:  ldfld      class [System]System.ComponentModel.Container
                  PictureKey.frmMain::components
      IL_0011:  callvirt   instance void [System]System.ComponentModel
                  .Container::Dispose()
      IL_0016:  ldarg.0
      IL_0017:  ldarg.1
      IL_0018:  call       instance void [System.Windows.Forms]System
                  .Windows.Forms.Form::Dispose(bool)
      IL_0026:  ret
    }
</pre>

Was wird also passieren, wenn wir eine neue lokale Variable einf&uuml;gen und eine Konstante darin speichern,
kurz bevor die Methode verlassen wird? Ja, nichts wird passieren, au&szlig;er vielleicht einer minimalen Verz&ouml;gerung.

<pre>
.method family hidebysig virtual instance void
            Dispose(bool disposing) cil managed
    {
      // Code size       39 (0x27)
      .maxstack  2
      .locals init (int32 V_0) //neue lokale Variable deklarieren

          ...

      IL_001d:  ldc.i4     0x74007a //Eine int32 Konstante laden
      IL_0022:  stloc      V_0 //Die Konstante in der Variablen speichern
      IL_0026:  ret
    }
</pre>

In C# w&uuml;rde diese Methode so aussehen:

<pre lang=cs>
//Original
protected override void Dispose( bool disposing ) {
        if( disposing ) {
                if (components != null) {
                        components.Dispose();
                }
        }
        base.Dispose( disposing );
}

//Version mit versteckter Variable
protected override void Dispose( bool disposing ) {
        <b>int myvalue = 0;</b>
        if( disposing ) {
                if (components != null) {
                        components.Dispose();
                }
        }
        base.Dispose( disposing );
        <b>myvalue = 0x74007a;</b>
}
</pre>

Wir haben gerade vier Bytes in einer Anwendung versteckt!
Die IL Datei wird sich wieder fehlerfrei kompilieren lassen, und wenn jemand das neue Assembly
disassembliert kann er den Wert 0x74007a wiederfinden.

## Einen geheimen Wert tarnen

Um Leuten, die eine Anwendung disassemblieren und nach nutzlosen Variablen suchen, die Arbeit
zu erschweren, kann man die versteckten Werte als vergessene Debug-Ausgabe tarnen:

<pre>
ldstr bytearray(65 00) //Ein "A" laden...
stloc mystringvalue    //...und wegspeichern
.maxstack  2           //Stackgr&ouml;sse setzen, um Laufzeitfehler auszuschlie&szlig;en
ldstr "DEBUG - current value is: {0}"
ldloc mystringvalue    //vergessenen Debug-Outout vort&auml;uschen
call void [mscorlib]System.Console::WriteLine(string, object)
</pre>

Um auch in Konsolenanwendungen unsichtbar zu bleiben, sollten wir die Variablen besser als Operationen tarnen.
Wir k&ouml;nnten mehr lokale/statische/Instanz-Variablen einf&uuml;gen, damit es so aussieht, als w&uuml;rden die
Werde an anderer Stelle gebraucht werden:

<pre>
.maxstack  2  //Stack-Gr&ouml;sse anpassen
ldc.i4 65     //"A" laden
ldloc myintvalue //noch eine Variable laden - die Deklaraion steht irgendwo weiter oben
add           //65 + myintvalue
stsfld int32 NameSpace.ClassName::mystaticvalue //Ergebnis vom Stack entfernen
</pre>

Dieses Beispiel soll demonstrieren, wie Informationen allgemein versteckt werden k&ouml;nnen,
darum werden wir nur diese Variante verwenden:

<pre>
ldc.i4 65;
stloc myvalue
</pre>

Man muss nicht f&uuml;r jedes Byte der Nachricht zwei Zeilen einf&uuml;gen.
Wir k&ouml;nnen bis zu vier Bytes in einen Int32-Wert stecken, und so nur eine halbe Zeile pro
verstecktem Byte einf&uuml;gen. Aber zuerst m&uuml;ssen wir wissen, wo genau wir dieses einf&uuml;gen.

## Ein Disassembly analysieren

Bevor man eine IL Datei bearbeiten kann, wird ILDAsm.exe aufgerufen, um sie aus dem kompilierten Assembly zu erstellen.
Sp&auml;ter rufen wir ILAsm.exe auf, um die Datei zu re-assemblieren. Der interessante Teil spielt sich dazwischen ab:
Wir m&uuml;ssen die Zeilen des IL Assembler Language Codes durchlaufen, die <code>void</code>-Methoden finden,
dann ihre jeweils letzte <code>.locals init</code> Zeile, und eine <code>ret</code>-Zeile.
Eine Nachricht kann mehr 4-Byte-Bl&ouml;cke enthalten als <code>void</code>-Methoden vorhanden sind,
darum m&uuml;ssen wir die Methoden z&auml;hlen und die Anzahl der Bytes berechnen, die in jeder davon versteckt werden.
Die Methode <code>Analyse</code> sammelt Namespaces, Klassen und <code>void</code>-Methoden:

<pre lang=cs>
/// &lt;summary&gt;Namespaces, Klassen und Methoden mit R&uuml;ckgabetyp &quot;void&quot; auflisten&lt;/summary&gt;
/// &lt;param name="fileName"&gt;Name der IL Datei&lt;/param&gt;
/// &lt;param name="namespaces"&gt;Gibt die Namen gefundener Namespaces zur&uuml;ck&lt;/param&gt;
/// &lt;param name="classes"&gt;Gibt die Namen gefundener Klassen zur&uuml;ck&lt;/param&gt;
/// &lt;param name="voidMethods"&gt;Gibt die ersten Zeilen aller Methoden-Signaturen zur&uuml;ck&lt;/param&gt;
public void Analyse(String fileName,
        out ArrayList namespaces, out ArrayList classes, out ArrayList voidMethods){

        //R&uuml;ckgabelisten initialisieren
        namespaces = new ArrayList(); classes = new ArrayList(); voidMethods = new ArrayList();
        //Anfang der aktuellen Methode, oder null bei nicht-void Methoden
        String currentMethod = String.Empty;

        //IL Datei zeilenweise lesen
        String[] lines = ReadFile(fileName);

        //F&uuml;r alle Zeilen der Datei: Listen f&uuml;llen
        for(int indexLines=0; indexLines&lt;lines.Length; indexLines++){
                if(lines[indexLines].IndexOf(".namespace ") &gt; 0){
                        //Namespace gefunden!
                        namespaces.Add( ProcessNamespace(lines[indexLines]) );
                }
                else if(lines[indexLines].IndexOf(".class ") &gt; 0){
                        //Klassen gefunden!
                        classes.Add( ProcessClass(lines, ref indexLines) );
                }
                else if(lines[indexLines].IndexOf(".method ") &gt; 0){
                        //Methode gefunden!
                        currentMethod = ProcessMethod(lines, ref indexLines);
                        if(currentMethod != null){
                                //Methode gibt void zur&uuml;ck - auflisten
                                voidMethods.Add(currentMethod);
                        }
                }
        }
}
</pre>

Mit der Anzahl verwendbarer Methoden k&ouml;nnen wir jetzt die Anzahl versteckter Bytes pro Methode berechnen:

<pre>
//L&auml;nge des Unicode-Strings + 1- Position f&uuml;r die L&auml;nge (wird wie immer mit der Nachricht versteckt)
float messageLength = txtMessage.Text.Length*2 +1;
//Bytes pro Methode
int bytesPerMethod = (int)Math.Ceiling( (messageLength / (float)voidMethods.Count));
</pre>

Endlich k&ouml;nnen wir anfangen. Die Methode <code>HideOrExtract</code> verwendet den Wert von
<code>bytesPerMethod</code>, um die Zeilen f&uuml;r einen oder mehrere 4-Byte-Bl&ouml;cke &uuml;ber den
<code>ret</code>-Anweisungen einzuf&uuml;gen.

<pre lang=cs>
/// &lt;summary&gt;Versteckt oder extrahiert eine Nachricht in/aus einer IL Datei&lt;/summary&gt;
/// &lt;param name="fileNameIn"&gt;Name der IL Datei&lt;/param&gt;
/// &lt;param name="fileNameOut"&gt;Name f&uuml;r die Ausgabedatei - ignoriert, wenn [hide]==false&lt;/param&gt;
/// &lt;param name="message"&gt;Nachricht zum Verstecken, oder leerer Stream f&uuml;r die extrahierte Nachricht&lt;/param&gt;
/// &lt;param name="hide"&gt;true: [message] verstecken; false: eine Nachricht auslesen&lt;/param&gt;
private void HideOrExtract(String fileNameIn, String fileNameOut, Stream message, bool hide){
        if(hide){
                //Zieldatei &ouml;ffnen
                FileStream streamOut = new FileStream(fileNameOut, FileMode.Create);
                writer = new StreamWriter(streamOut);
        }else{
                //Anzahl der Bytes pro Methode ist noch unbekannt
                //und wird der erste ausgelesene Wert sein
                bytesPerMethod = 0;
        }

        //Quelldatei lesen
        String[] lines = ReadFile(fileNameIn);
        //nein, wir sind noch nicht fertig
        bool isMessageComplete = false;

        //F&uuml;r alle Zeilen
        for(int indexLines=0; indexLines&lt;lines.Length; indexLines++){

                if(lines[indexLines].IndexOf(".method ") &gt; 0){
                        //Methode gefunden!
                        if(hide){
                                //einen Block von Bytes verstecken
                                isMessageComplete = ProcessMethodHide(lines, ref indexLines, message);
                        }else{
                                //Alle in dieser Methode versteckten Bytes auslesen
                                isMessageComplete = ProcessMethodExtract(lines, ref indexLines, message);
                        }
                }else if(hide){
                        //Die Zeile geh&ouml;rt nicht zu einer verwendbaren Methode - einfach kopieren
                        writer.WriteLine(lines[indexLines]);
                }

                if(isMessageComplete){
                        break; //Nichts mehr zu tun
                }
        }

        //Zieldatei schlie&szlig;en
        if(writer != null){ writer.Close(); }
}
</pre>

## Die Nachricht verstecken

Die Methode <code>ProcessMethodHide</code> kopiert die Signatur der Methode und pr&uuml;ft, ob der
R&uuml;ckgabetyp <code>void</code> ist. Dnach wird die letzte <code>.locals init</code> Zeile gesucht.
Wird kein <code>.locals init</code> gefunden, dann wird die zus&auml;tzliche Variable am Anfange der
Methode eingef&uuml;gt. Die versteckte Variable muss die letzte Variable sein, die in der Methode
deklariert wird, weil Compiler die IL Assembler Language ausgeben f&uuml;r lokale Variablen oft
Slot Nummern anstelle von Namen verwenden. Stell Dir nur mal so eine Katastrophe vor:


<pre>
//Ein C# Compiler hat diesen Code produziert, der 5+2 addiert
//Original C# code:
//int x = 5; int y = 2;
//mystaticval = x+y;

.locals init ([0] int32 x, [1] int32 y)
IL_0000:  ldc.i4.5
IL_0001:  stloc.0
IL_0002:  ldc.i4.2
IL_0003:  stloc.1
IL_0004:  ldloc.0
IL_0005:  ldloc.1
IL_0006:  add
IL_0007:  stsfld     int32 Demo.Form1::mystaticval
IL_000c:  ret
</pre>

W&uuml;rden wir eine Deklaration am Anfang der Methode einf&uuml;gen,
k&ouml;nnten wir den Code nicht re-assemblieren, da Slot 0 bereits von <code>myvalue</code> verwendet wird:

<pre>
.locals init (int32 myvalue)
.locals init ([0] int32 x, [1] int32 y) //Fehler!
IL_0000:  ldc.i4.5
IL_0001:  stloc.0
...
</pre>

Darum muss die zus&auml;tzliche lokale Variable nach dem letzten vorhandenen <code>.locals init</code>
initialisiert werden. <code>ProcessMethodHide</code> f&uuml;gt diese neue Variable ein, springt zur ersten
<code>ret</code>-Anweisung und f&uuml;gt  ldc.i4/stloc Paare ein.
Der erste Wert, der so versteckt wird, ist die Gr&ouml;sse des Nachrichten-Streams - die auslesende Methode
braucht diesen Wert, um zu wissen wann sie aufh&ouml;ren muss.
Der letzte Wert, der in der ersten Methode versteckt wird, ist die Anzahl von Nachrichten-Bytes pro Methode.
Dieser muss direkt &uuml;ber der <code>ret</code>-Zeile stehen, da die auslesende Methode
ihn finden muss, ohne zu wissen wie viele Zeilen sie zur&uuml;ck springen muss (weil das von gerade diesem Wert abh&auml;ngt).

<pre lang=cs>
/// &lt;summary&gt;Versteckt ein oder mehrere Bytes des Nachrichten-Streams in der IL Datei&lt;/summary&gt;
/// &lt;param name="lines"&gt;Zeilen der IL Datei&lt;/param&gt;
/// &lt;param name="indexLines"&gt;Aktueller Index in [lines]&lt;/param&gt;
/// &lt;param name="message"&gt;Stream der die Nachricht enth&auml;lt&lt;/param&gt;
/// &lt;returns&gt;true: letztes Byte wurde versteckt; false: noch mehr Nachrichten-Bytes warten&lt;/returns&gt;
private bool ProcessMethodHide(String[] lines, ref int indexLines, Stream message){
        bool isMessageComplete = false;
        int currentMessageValue,        //n&auml;chstes Byte zum Verstecken
                positionInitLocals,                //Index der letzten ".locals init"-Zeile
                positionRet,                        //Index der "ret"-Zeile
                positionStartOfMethodLine; //Index der ersten Zeile der Methode

        writer.WriteLine(lines[indexLines]); //copy first line

        //Ignorieren, wenn keine "void"-Methode
        if(lines[indexLines].IndexOf(" void ") > 0){
                //"void"-Methode gefunden
                //Der Stack wird am Ende leer sein,
                //also k&ouml;nnen wir (fast) alles M&ouml;gliche einf&uuml;gen

                indexLines++; //N&auml;chste Zeile
                //Anfang des Methoden-Blocks suchen, alle ausgelassenen Zeilen kopieren
                int oldIndex = indexLines;
                SeekStartOfBlock(lines, ref indexLines);
                CopyBlock(lines, oldIndex, indexLines);

                //Jetzt sind wir bei der &ouml;ffnenden Klammer der Methode
                positionStartOfMethodLine = indexLines;
                //Zur ersten Zeile der Methode gehen
                indexLines++;
                //get position of last ".locals init" and first "ret"
                positionInitLocals = positionRet = 0;
                SeekLastLocalsInit(lines, ref indexLines, ref positionInitLocals, ref positionRet);

                if(positionInitLocals == 0){
                        //kein .locals - Zeile am Anfang er Methode einf&uuml;gen
                        positionInitLocals = positionStartOfMethodLine;
                }

                //Von Anfang bis letztem .locals kopieren, oder nichts (wenn kein .locals gefunden)
                CopyBlock(lines, positionStartOfMethodLine, positionInitLocals+1);
                indexLines = positionInitLocals+1;
                //lokale Variable einf&uuml;gen
                writer.Write(writer.NewLine);
                writer.WriteLine(".locals init (int32 myvalue)");
                //Rest der Methode bis zur Zeile vor "ret" kopieren
                CopyBlock(lines, indexLines, positionRet);

                //N&auml;chste Zeile ist "ret" - auf dem Stack kann nichts kaputtgehen
                indexLines = positionRet;

                //ldc/stloc Paare einf&uuml;gen f&uuml;r [bytesPerMethod] Bytes aus dem Message Stream
                //4 Bytes zu einem Int32 kombinieren
                for(int n=0; n&lt;bytesPerMethod; n+=4){
                        isMessageComplete = GetNextMessageValue(message, out currentMessageValue);
                        writer.WriteLine("ldc.i4 "+currentMessageValue.ToString());
                        writer.WriteLine("stloc myvalue");
                }

                //bytesPerMethod muss der letzte Wert in der ersten Methode sein
                if(! isBytesPerMethodWritten){
                        writer.WriteLine("ldc.i4 "+bytesPerMethod.ToString());
                        writer.WriteLine("stloc myvalue");
                        isBytesPerMethodWritten = true;
                }

                //Aktuelle Zeile kopieren
                writer.WriteLine(lines[indexLines]);

                if(isMessageComplete){
                        //Nichts mehr gelesen, die Nachricht ist vollst&auml;ndig
                        //Rest der Quelldatei kopieren
                        indexLines++;
                        CopyBlock(lines, indexLines, lines.Length-1);
                }
        }
        return isMessageComplete;
}
</pre>

## Versteckte Werte auslesen

Die Methode <code>ProcessMethodExtract</code> sucht die erste <code>ret</code>-Zeile.
Wenn die Anzahl der pro Methode versteckten Bytes noch unbekannt ist, wird zwei Zeile zur&uuml;ck gesprungen,
wo die Anzahl aus der <code>ldc.i4</code>-Zeile gelesen wird (die als letzter Wert in die erste Methode eingef&uuml;gt wurde).
Andernfalls wird zwei Zeilen pro erwartetem ldc.i4/stloc Paar zur&uuml;ck gesprungen,
von wo dann die 4-Byte-Bl&ouml;cke extrahiert und in den Nachrichten-Stream geschrieben werden.
Wenn kein <code>ldc.i4</code> gefunden wird, wo eines sein sollte, wird das Programm eine Exception.<br>
Der zweite ausgelesene Wert ist die L&auml;nge der folgenden Nachricht.
Wenn der Nachrichten-Stream diese erwartete L&auml;nge erreicht hat, wird das <code>isMessageComplete</code>
Flag gesetzt, <code>HideOrExtract</code> wird verlassen und die extrahierte Nachricht angezeigt.
Auslesen funktioniert genauso wie Verstecken in umgekehrter Richtung.


## Kein Schl&uuml;ssel ?!

Bestimmt ist Dir aufgefallen, dass diese Anwendung keinen Schl&uuml;ssel verwendet, um die Nachricht zu verteilen.
Ein durchschnittliches Assembly enth&auml;lt weniger <code>void</code>-Methoden als ein durchschnittlicher Satz,
ein Verteilungsschl&uuml;ssel wie er auf den letzten Seite verwendet wurde, w&uuml;rde hier nur dazu f&uuml;hren,
dass massenweise Nonsense-Zeilen in wenige Methoden geschrieben werden, was allzu offensichtlich w&auml;re.
[Im nächsten Artikel]({% link steganography_de/11_net_assemblies.md %}) werden alle Methoden (nicht nur voids) verwendet,
so dass ein Verteilungsschl&uuml;ssel wieder Sinn macht.


