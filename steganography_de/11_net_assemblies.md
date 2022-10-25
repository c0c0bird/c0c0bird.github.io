---
title: Spaß auf dem Stack
category: Steganografie
subcategory: .NET Assemblies / IL Code
order: 11
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 13.0 Kb]({% link /assets/steganodotnet7_src.zip %})

## Worum geht es?


Im [letzten Artikel]({% link steganography_de/10_net_assemblies.md %}) wurden nur
`void`-Methoden verwendet, was die L&auml;nge der versteckten Nachricht stark eingeschr&auml;nkt hat.
Dieser Artikel erweitert das Programm:

* Alle Methoden mit den R&uuml;ckgabetypen `void`, `bool`, `int32` oder `string` k&ouml;nnen verwendet werden.
* Ein Schl&uuml;ssel definiert, wie die Nachricht getarnt wird.


Der Unterschied zwischen `void`- und nicht-`void`-Methoden ist nicht sehr gro&szlig;.
Am Ende einer nicht-`void`-Methode ist der Stack nicht leer, er enth&auml;lt einen Wert des deklarierten Typs.
Das heisst, diese Anwendung muss den Namen des R&uuml;ckgabetyps aus der Methodensignatur lesen,
und eine zus&auml;tzliche lokale Variable deklarieren. In der Zeile vor dem ersten "ret" muss sie den
Stack-Inhalt (der den R&uuml;ckgabewert enh&auml;lt, und sonst nichts) in dieser Variablen speichern,
die Zeilen mit den versteckten Bytes einf&uuml;gen, und dann den Wert zur&uuml;ck auf den Stack laden.


Schau Dir als Beispiel diese `int`-Methode an:
<pre>
private int intTest(){
        int a = 1;
        return a;
}
</pre>

Der C# compiler &uuml;bersetzt sie so:

<pre>
.method private hidebysig instance int32
        intTest() cil managed
{
        // Code size       8 (0x8)
        .maxstack  1
        .locals init ([0] int32 a,
                        [1] int32 CS$00000003$00000000)
        IL_0000:  ldc.i4.1
        IL_0001:  stloc.0
        IL_0002:  ldloc.0
        IL_0003:  stloc.1
        IL_0004:  br.s       IL_0006

        IL_0006:  ldloc.1
        IL_0007:  ret
} // end of method Form1::intTest
</pre>


        Der Compiler hat eine zweite Variable erzeugt, um den R&uuml;ckgabewert abzulegen.
        Am Ende der Methode wird dieser Wert auf den Stack gelegt, das ist schon alles.
        Also wird nichts kaputt gehen, wenn wir ein paar Zeilen zwischen `IL_0006`
        und `IL_0007` schreiben, und anschlie&szlig;end den Stack aufr&auml;umen
        bevor wir wieder den R&uuml;ckgabewert laden:

<pre>
.method private hidebysig instance int32
        intTest() cil managed
{
                // Code size       8 (0x8)
        <b>.maxstack 2</b> //Stack-Gr&ouml;&szlig;e anpassen
        .locals init ([0] int32 a,
                                [1] int32 CS$00000003$00000000)

        .locals init (int32 myvalue)
        IL_0000:  ldc.i4.1
        IL_0001:  stloc.0
        IL_0002:  ldloc.0
        IL_0003:  stloc.1
        IL_0004:  br.s       IL_0006

        IL_0006:  ldloc.1

        <b>.locals init (int32 returnvalue)</b> //Variable hinzuf&uuml;gen
        <b>stloc returnvalue</b> //Rückgabewert zwischenspeichern
        <b>ldstr "DEBUG - current value is: {0}"</b> //etwas das wie alter Debug-Code aussieht
        <b>ldc.i4 111</b> //Heir ist unser versteckter Wert
        <b>box [mscorlib]System.Int32
        call void [mscorlib]System.Console::WriteLine(string,
        object)

        ldloc returnvalue</b> //Rückgabewert dorthin zurücklegen, wo er her kam
        IL_0007:  ret
} // end of method Form1::intTest
</pre>


Jetzt kann ILAsm den Code re-compilieren. Wenn man ihn wieder decompiliert, kann man sehen,
dass ILAsm die Variablendeklarationen optimiert hat:


<pre>
    .method private hidebysig instance int32
            intTest() cil managed
    {
      // Code size       36 (0x24)
      .maxstack  2
      <b>.locals init (int32 V_0,</b> //ILAsm hat die lokalen Variablen zusammengefa&szlig;t !
               <b>int32 V_1,
               int32 V_2,
               int32 V_3)</b>
      IL_0000:  ldc.i4.1
      IL_0001:  stloc.0
      IL_0002:  ldloc.0
      IL_0003:  stloc.1
      IL_0004:  br.s       IL_0006

      IL_0006:  ldloc.1
      IL_0007:  stloc      V_3
      IL_000b:  ldstr      "DEBUG - current value is: {0}"
      IL_0010:  ldc.i4     0x6f
      IL_0015:  box        [mscorlib]System.Int32
      IL_001a:  call       void [mscorlib]System.Console::WriteLine(string,
                                                                    object)
      IL_001f:  ldloc      V_3
      IL_0023:  ret
    } // end of method Form1::intTest
</pre>


ILAsm r&auml;umt meine Zeilen auf, ist das nicht nett? Nein, das ist &uuml;berhaupt nicht nett, denn wir
k&ouml;nnen uns nicht darauf verlassen, dass unsere eingef&uuml;gten Zeilen noch vorhanden sind,
nachdem der IL Code compiliert und wieder decompiliert wurde.
Das heisst, was immer wir einf&uuml;gen, um Teile der geheimen Nachricht zu verstecken, muss einen Sinn ergeben.
Eine zus&auml;tzliche `.maxlength`-Zeile wird verschwinden, genau wie eine
`.locals init`-Zeile mit Variablen, die nie verwendet werden.
Bitte denke an diesen Effekt, wenn Du Dir neue Byte-Verkleidungen ausdenkst!


## Einen Schl&uuml;ssel-Stream verwenden

Im letzten Artikel wurden immer diese zwei Zeilen verwendet, um einen `int32` zu verstecken:
<pre>
<b>ldc.i4 65;</b>
stloc myvalue
</pre>

Wie Du schon oben gesehen hast, k&ouml;nnen diese Zeilen die gleichen Daten verstecken:
<pre>
ldstr      "DEBUG - current value is: {0}"
<b>ldc.i4     65</b>
box        [mscorlib]System.Int32
call       void [mscorlib]System.Console::WriteLine(string, object)
</pre>


Es gibt hunderte solcher Bl&ouml;cke, welche Variantion sollen wir verwenden?
Wir werden alle verwenden, oder - um das Beispiel einfach zu lassen - diese zwei Variationen.
Der Benutzer kann eine beliebige Datei angeben, und f&uuml;r jeden Vier-Byte-Block liest die
Anwendung ein Byte aus dieser Datei: Wenn es eine gerade Zahl ist, wird die erste Variation
verwendet, sonst die Zweite.


<pre>
private bool ProcessMethodHide(String[] lines, ref int indexLines,
                                                                Stream message, Stream key){
        //...
        //Zeilen f&uuml; [bytesPerMethod] Bytes aus dem Nachrichten-Stream einf&uuml;gen
        //jeweils 4 bytes zu einem Int32 kombinieren
        int keyValue; //aktueller Wert aus dem Schl&uuml;ssel-Stream
        for(int n=0; n&lt;bytesPerMethod; n+=4){
                isMessageComplete = GetNextMessageValue(message, out currentMessageValue);

                //n&auml;chstes Bytes aus dem Schl&uuml;ssel lesen
                if( (keyValue=key.ReadByte()) < 0){
                        key.Seek(0, SeekOrigin.Begin);
                        keyValue=key.ReadByte();
                }

                if(keyValue % 2 == 0){
                        //der Schl&uuml;ssel ist gerade - erste Variation verwenden
                        writer.WriteLine("ldc.i4 "+currentMessageValue.ToString());
                        writer.WriteLine("stloc myvalue");
                }else{
                  //der Schl&uuml;ssel ist ungerade - zweite Variation verwenden
                  writer.WriteLine("ldstr \"DEBUG - current value is: {0}\"");
                  writer.WriteLine("ldc.i4 "+currentMessageValue.ToString());
                  writer.WriteLine("box [mscorlib]System.Int32");
                  writer.WriteLine("call void [mscorlib]System.Console::WriteLine(string, ");
                  writer.WriteLine( "object)" ); //ILDAsm f&uuml; hier einen Zeilenumbruch ein
                }
        }
        //...
}
</pre>


Bei der ersten Variation m&uuml;ssen wird die versteckte Konstante in der ersten Zeile suchen,
bei der zweiten Variation m&uuml;ssen wir sie aus der zweiten Zeile lesen.
Beim Auslesen der versteckten Nachricht m&uuml;ssen wir also die erste Zeile &uuml;berspringen,
wenn das Schl&uuml;ssel-Byte ungerade ist:


<pre>
private bool ProcessMethodExtract(String[] lines, ref int indexLines,
                                                                        Stream message, Stream key){
        //[bytesPerMethod] Bytes in den Nachrichten-Stream lesen
        //wenn [bytesPerMethod]==0 ist, wurde es noch nicht gelesen
        for(int n=0; (n&lt;bytesPerMethod)||(bytesPerMethod==0); n+=4){

        if(bytesPerMethod > 0){
                //n&auml;chstes Bytes aus dem Schl&uuml;ssel lesen
                if( (keyValue=key.ReadByte()) < 0){
                        key.Seek(0, SeekOrigin.Begin);
                        keyValue=key.ReadByte();
                }

                if(keyValue % 2 == 1){
                        //ldc.i4 steht in der zweiten Zeile des versteckten Blocks
                        indexLines++;
                }
        }

        //ILDAsm setzt Zeilennummern - Anfange der Anweisung finden
        indexValue = lines[indexLines].IndexOf("ldc.i4");
        if(indexValue >= 0){

        //...
</pre>


Jetzt k&ouml;nnen wir Daten verstecken und extrahieren, aber viele re-compilierte
Assemblies st&uuml;rzen mit einer `InvalidProgramException` ab.
Das kommt daher, dass die zweite Variation zwei Werte auf den Stack legt:


<pre>
        <b>.maxstack 1</b> //eine kleine Methode verwendet nur eine Variable auf einmal
        ...
        <b>ldstr      "DEBUG - current value is: {0}"</b>
        <b>ldc.i4     0x6f</b> //wir versuchen, einen zweiten Wert auf den Stack zu laden
        box        [mscorlib]System.Int32
        call       void [mscorlib]System.Console::WriteLine(string,
                                                        object)
</pre>


Deshalb m&uuml;ssen wir sicherstellen, dass der `.maxstack`-Wert in jeder
Methode mindestens 2 ist.
Die `.maxstack`-Zeile ist eine von denen, die wir bisher unbeachtet kopiert haben:


<pre>
        CopyBlock(lines, startIndex, endIndex);

        //...

        private void CopyBlock(String[] lines, int start, int end){
                String[] buffer = new String[end-start];
                Array.Copy(lines, start, buffer, 0, buffer.Length);
                writer.WriteLine(String.Join(writer.NewLine, buffer));
        }
</pre>


Jetzt m&uuml;ssen wir die `.maxstack`-Zeilen finden und anpassen, sonst w&uuml;rden
wir Assemblies zerst&ouml;ren, die Methoden mit einem maxstack von 1 enthalten.
Diese Zeilen k&ouml;nnen wir nicht mit `Array.IndexOf(".maxstack 1")` finden, weil die
exakte Zeile unbekannt ist - denk nur mal an die Zeilennummern, Tabs und Leerzeichen, die ILDAsm
in jede Zeile einf&uuml;gt. Also werden wir die Methoden Zeile f&uuml;r Zeile kopieren:


<pre>
private void CopyBlockAdjustStack(String[] lines, int start, int end){
        for(int n=start; n&lt;end; n++){

                if(lines[n].IndexOf(".maxstack ")&gt;0){
                        //Stack-Gr&ouml;&szlig;e lesen
                        int indexStart = lines[n].IndexOf(".maxstack ");
                        int maxStack = int.Parse( lines[n].Substring(indexStart+10).Trim() );
                        //maxstack muss 2 oder gr&ouml;&szlig;er sein
                        if(maxStack < 2){
                                lines[n] = ".maxstack 2";
                        }
                }

                writer.WriteLine(lines[n]);
        }
}
</pre>

## R&uuml;ckgabewerte behandeln


Der R&uuml;ckgabetyp einer Methode wird im Header deklariert, das heisst wir m&uuml;ssen ihn
lesen und zwischenspeichern, sobald wir eine neue Methde betreten:


<pre>
private String GetReturnType(String line){
        String returnType = null;
        if(line.IndexOf(" void ") > 0){ returnType = "void"; }
        else if(line.IndexOf(" bool ") > 0){ returnType = "bool"; }
        else if(line.IndexOf(" int32 ") > 0){ returnType = "int32"; }
        else if(line.IndexOf(" string ") > 0){ returnType = "string"; }
        return returnType;
}

private bool ProcessMethodHide(String[] lines, ref int indexLines,
                                                                Stream message, Stream key){

        //..

        //R&uuml;ckgabewert der aktuellen Methode lesen
        <b>String returnType = GetReturnType(lines[indexLines]);</b>

        if(returnType != null){
                //void/bool/int32/string-Methode gefunden

                //...

                //Position des letzten ".locals init" und ersten "ret" suchen
                positionInitLocals = positionRet = 0;
                SeekLastLocalsInit(lines, ref indexLines,
                                                        ref positionInitLocals, ref positionRet);

                //...

                //Rest der Methode bis zur Zeile vor "ret" kopieren
                CopyBlockAdjustStack(lines, indexLines, positionRet);

                //n&auml;chste Zeile ist "ret" - auf dem Stack kann nichts kaputt gehen
                indexLines = positionRet;

                <b>if(returnType != "void"){
                        //not a void method - store the return value
                        writer.Write(writer.NewLine);
                        writer.WriteLine(".locals init ("+returnType+" returnvalue)");
                        writer.WriteLine("stloc returnvalue");
                }</b>

                //Zeile f&uuml;r [bytesPerMethod] Bytes vom Nachrichten-Stream einf&uuml;gen
                //4 Bytes zu einem Int32 kombinieren
                int keyValue;
                for(int n=0; n&lt;bytesPerMethod; n+=4){
                        //...
                }
                //...

                <b>if(returnType != "void"){
                        //keine void-Methode - R&uuml;ckgabewert zur&uuml;ck auf den Stack laden
                        writer.WriteLine("ldloc returnvalue");
                }</b>

                //...

        } //else diese Methode auslassen
}
</pre>


Wir m&uuml;ssen die Zeile `ldloc returnvalue` beim Extrahieren nur auslassen.


<pre>
private bool ProcessMethodExtract(String[] lines, ref int indexLines,
                                                                        Stream message, Stream key){
        bool isMessageComplete = false;
        int positionRet,                                //index der "ret"-Zeile
                positionStartOfMethodLine;        //index der ersten Zeile

        String returnType = GetReturnType(lines[indexLines]);
        int keyValue = 0;

        if(returnType != null){
                //void/bool/int32/string-gethode gefunden
                //ein Teil der Nachricht ist hier versteckt

                //...

                //Position des "ret" suchen
                positionRet = SeekRet(lines, ref indexLines);

                if(bytesPerMethod == 0){
                        //zwei Zeilen zur&uuml;ck gehen - dort haben wir "ldc.i4 "+bytesPerMethod eingef&uuml;gt
                        indexLines = positionRet - 2;
                }else{
                        //[linesPerMethod] Zeilen pro erwartetem Nachrichten-Byte zur&uuml;ck gehen
                        //dort haben wir "ldc.i4 "+currentByte eingef&uuml;gt
                        linesPerMethod = GetLinesPerMethod(key);
                        indexLines = positionRet - linesPerMethod;
                }

                <b>if(returnType != "void"){
                        indexLines--; //die Zeile "ldloc returnvalue" &uuml;berspringen
                }</b>

                //...
        }
}
</pre>


Jetzt k&ouml;nnen wir eine Schl&uuml;ssel-Datei anwenden, und die meisten Methoden ausnutzen.
Wenn Du mehr Methoden verwenden m&ouml;chtest, brauchst Du nur die Methode `GetReturnType` anzupassen.
Mehr Variationen von Dummy-Code einzuf&uuml;gen ist etwas aufw&auml;ndiger, Du musst
`ProcessMethodHide`, `ProcessMethodExtract` und `GetLinesPerMethod`
anpassen - und denke daran, gegebenenfalls den `.maxstack` zu vergr&ouml;&szlig;ern.


