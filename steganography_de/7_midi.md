---
title: MIDI-Dateien
category: Steganografie
subcategory: Audiodaten
order: 007
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 19.5 Kb]({% link /assets/steganodotnet5_src.zip %})

## Worum geht es?

Die meisten MIDI Nachrichten sind h&ouml;rbar, aber manche steuern nur Einstellungen des MIDI Ger&auml;tes.
Dieser Artikel besch&auml;ftigt sich damit, die Nachricht &quot;Program Change&quot;, die Klangfarbe des Instruments festlegt,
zum Verstecken kurzer Texte zu missbrauchen.

## Kurzer MIDI &Uuml;berblick

Eine MIDI Datei enth&auml;lt Ereignisse. Jedes Ereignis besteht aus seiner Zeit, einem Nachrichtentyp, und einer
bestimmten Menge von Parametern (Datenbytes). Acht Typen sind m&ouml;glich:

<table class="border" cellspacing="0">
<tr><td class="border">Typ</td><td class="border">Name</td><td class="border">Daten</td><td class="border">Bedeutung</td></tr>
<tr><td class="border">80</td><td class="border">Note Off</td><td class="border">2 Bytes (Note, Velocity)</td><td class="border">Ein Ton wird losgelassen</td></tr>
<tr><td class="border">90</td><td class="border">Note On</td><td class="border">2 Bytes (Note, Velocity)</td><td class="border">Ein Ton wird gespielt</td></tr>
<tr><td class="border">A0</td><td class="border">After Touch</td><td class="border">2 Bytes (Note, Pressure)</td><td class="border">Der Druck auf eine Taste &auml;ndert sich zwischen 90 and 80</td></tr>
<tr><td class="border">B0</td><td class="border">Control Change</td><td class="border">2 Bytes (Control, Value)</td><td class="border">Eine Ger&auml;teabh&auml;ngige Einstellung wird ge&auml;ndert</td></tr>
<tr><td class="border">C0</td><td class="border">Program change</td><td class="border">1 Byte (Program Number)</td><td class="border">Eine andere Klangfarbe wird gew&auml;hlt</td></tr>
<tr><td class="border">D0</td><td class="border">Channel Pressure</td><td class="border">1 Byte (Pressure)</td><td class="border">After Touch f&uuml;r einen ganzen Kanal; f&uuml;r Ger&auml;te ohne einzelne Sensoren an jeder Taste</td></tr>
<tr><td class="border">E0</td><td class="border">Pitch Wheel</td><td class="border">2 Bytes (combined to a 14-bit value)</td><td class="border">Das Pitch Wheel wird verstellt</td></tr>
<tr><td class="border">F0</td><td class="border">System Exclusive</td><td class="border">all Bytes to next 0xF7</td><td class="border">Ger&auml;teabh&auml;ngige Nachricht</td></tr>
</table>

Die unteren vier Bits sind f&uuml;r die Kanalnummer reserviert.
Wenn man ein mittleres C auf Kanal 5 spielt, sendet der Sequencer (z.B. MIDI Keyboard) so eine Nachricht:
02 94 3C B7<br/>
Zwei Einheiten vom Anfang, Taste auf Kanal 4 (von 0 an gez&auml;hlt), Note 60 (mittleres C), Anschlagsgeschwindigkeit 92.

Wenn man die Klangfarbe auf &quot;Piano&quot; stellt, bevor man anf&auml;ngt zu spielen,
sendet er so eine Nachricht:
00 C4 00<br/>
Vor dem Anfang, Programmwechsel auf Kanal 5, neues Program ist Nummer 0.

Wenn der Sequencer die aufgezeichneten Nachrichten speichert, setzt er einen Header an den Anfang
der Datei, und einen an den Anfang jedes Tracks.
Jeder Header enth&auml;lt zwei Felder f&uuml;r Typ und L&auml;nge.


<pre>
struct ChunkHeader {
        char[] type; //char[4], MThd ot MTrk
        Int32 length;
}
</pre>

Der Typ kann &quot;MThd&quot; f&uuml;r einen Datei-Header sein, oder &quot;MTrk&quot; f&uuml;r einen Track-Header.
Ein Datei-Header sieht so aus:

<img src="/images/steganodotnet51.png" width="511" height="104" alt="" border="0">

Nach dem Datei-Header muss der Header des ersten Tracks folgen. Ein typischer Track-Header sieht so aus:

<img src="/images/steganodotnet52.gif" width="171" height="91" alt="" border="0">

Die L&auml;nge des Track-Headers gibt die Anzahl der Bytes bis zum n&auml;chsten Track-Header an.
Diese Bytes sind System- und MIDI-Nachrichten. System-Nachrichten haben den Typ 0xFF, ein Subtyp-Byte,
und ein Length-Byte. Die L&auml;nge gibt die Anzahl der Datenbytes an:

<img src="/images/steganodotnet53.gif" width="461" height="96" alt="" border="0">

Normalerweise beginnt eine Datei mit einer Reihe Non-MIDI Nachrichten, gefolgt von Control Change Nachrichten,
und den Program Change und Note On/Off Nachrichten:

<img src="/images/steganodotnet54.gif" width="493" height="58" alt="" border="0">

Das Ende jedes Tracks wird von einem End Of Track Ereignis markiert:

<img src="/images/steganodotnet55.gif" width="185" height="54" alt="" border="0">

Wenn Du alles &uuml;ber die MIDI Spezifikation wissen m&auml;chtest, empfehle ich das
<a href="http://www.borg.com/~jglatt/" target="_blank">MIDI Technical Fanatic&acute;s Brainwashing Center</a>.

## Stille Verstecke

Was passiert, wenn ein paar Program Change Nachrichten aufeinander folgen, ohne eine Note On/Off Nachricht dazwischen?
Das MIDI-Ger&auml;t wechselt von einer Einstellung zur n&auml;chsten, erreicht die Letzte und spielt erst dann den n&auml;chsten
Ton. Das Program Change selbst h&ouml;rt man nicht, man h&ouml;rt nur die T&ouml;ne, die in der aktuellen Klangfarbe gespielt werden.
Das heisst, wir k&ouml;nnen ein Program Change VOR einem anderen Program Change verstecken, und niemand wird es h&ouml;ren.

Das Datenbyte, das die Programmnummer enth&auml;lt, kann ein Wert zwischen 0 und 127 sein.
Bit #7 von jedem Oktett ist als a Start Of Message Flag reserviert. Bei allen Typen ist Bit #7 auf 1 gesetzt,
alle anderen Bytes verwenden nur die Bits #0 bis #6. Felder mit variabler L&auml;nge (Zeitfelder und Parameter von SysEx Nachrichten)
brauchen kein eigenes L&auml;ngnefeld, weil sie mit dem ersten Byte >127 aufh&ouml;ren (dieses muss der Anfang der n&auml;chsten
Nachricht sein).
Die Program Change Nachrichten sind also geeignet, um eine kurze Nachricht zu verstecken, aber die Bytes eines Unicode Textes
k&ouml;nnen >127 sein. Also m&uuml;ssen wir die Bytes teilen. Uns stehen mehr als genug Bits zu Verf&uuml;gung, um ein halbes Byte
in einem Program Change zu verstecken. Bytes zu spalten ist einfach:

<pre>
private byte[] SplitByte(byte b){
        byte[] parts = new byte[2];
        parts[0] = (byte)(b &gt;&gt; 4); //h&ouml;here H&auml;lfte in die Niedrigere schieben
        parts[1] = (byte)((byte)(b &lt;&lt; 4) &gt;&gt; 4); //H&ouml;here H&auml;lfte rausschieben, zur&uuml;ck schieben
        return parts;
}
</pre>

Wir m&uuml;ssen nur die MIDI Datei durchs&acute;uchen, bis wir ein Program Change Ereignis erreichen,
eine Kopie dieses Ereignisses einf&uuml;gen, bei der die Programmnummer unser Halbbyte enth&auml;lt,
und dann das n&auml;chste Program Change Ereignis suchen, um das n&auml;chste Halbbyte zu verstecken.
Eine durchschnittliche MIDI Datei enth&auml;lt weniger Program Change Ereignisse als ein durchschnittlicher
Satz Buchstaben, darum m&uuml;ssen wir mehrere falsche Ereignisse vor dem Original einf&uuml;gen.

Bevor wir anfangen, definieren wir erstmal ein paar Strukturen, die einiges erleichtern.

<pre>
/// &lt;summary&gt;Header einer MIDI Datei (MThd)&lt;/summary&gt;
public struct MidiFileHeader {
        /// &lt;summary&gt;char[4] - muss "MThd" (Dateianfang) sein&lt;/summary&gt;
        public char[] HeaderType;
        ///&lt;summary&gt;L&auml;nge der Header-Daten - muss 6 sein.
        ///Dieser Wert in ein Int32 in Big Endian Format (umgekehrte Byte-Reihenfolge)&lt;/summary&gt;
        public byte[] DataLength;
        /// &lt;summary&gt;Format der Datei
        /// 0 (ein Track)
        /// 1 (mehrere simultane Tracks)
        /// 2 (mehrere unabh&auml;ngige Tracks)&lt;/summary&gt;
        public Int16 FileType;
        /// &lt;summary&gt;Anzahl der Tracks&lt;/summary&gt;
        public Int16 CountTracks;
        /// &lt;summary&gt;Einheiten pro Viertelnote&lt;/summary&gt;
        public Int16 Division;
}

/// &lt;summary&gt;Header eines MIDI Tracks (MTrk)&lt;/summary&gt;
public struct MidiTrackHeader {
        /// &lt;summary&gt;char[4] - muss "MTrk" (beginning of track) sein&lt;/summary&gt;
        public char[] HeaderType;
        ///&lt;summary&gt;L&auml;nge in Bytes aller Nachrichten im Track
        ///Dieser Wert wird in Big Endian Format gespeichert&lt;/summary&gt;
        public Int32 DataLength;
}

/// &lt;summary&gt;Zeit, Typ und Parameter eines Ereignisses&lt;/summary&gt;
public struct MidiMessage {
        /// &lt;summary&gt;Delta Zeit - Feld varaibler L&auml;nge&lt;/summary&gt;
        public byte[] Time;
        /// &lt;summary&gt;//H&ouml;here 4 Bits: Type, Niedrigere 4 Bits: Channel&lt;/summary&gt;
        public byte MessageType;
        /// &lt;summary&gt;Ein oder zwei Datenbytes
        /// SysEx (F0) Nachrichten k&ouml;nnen mehr Datenbytes haben,  aber wir brauchen sie nicht&lt;/summary&gt;
        public byte[] MessageData;

        /// &lt;summary&gt;ERstellt eine neue Message aus einer Vorlage&lt;/summary&gt;
        /// &lt;param name="template"&gt;Vorlage f&uuml;r Zeit und Typ&lt;/param&gt;
        /// &lt;param name="messageData"&gt;Wert f&uuml;r die Datenbytes&lt;/param&gt;
        public MidiMessage(MidiMessage template, byte[] messageData){
                Time = template.Time;
                MessageType = template.MessageType;
                MessageData = messageData;
        }
}
</pre>

Jetzt k&ouml;nnen wir anfangen, die MIDI Datei zu lesen. Alle Sicherheits-Checks &uuml;ber Dateigr&ouml;&szlig;e etc.
sind ausgelassen, Du kannst sie im <a href="../steganodotnet5_src.zip">vollst&auml;ndigen Quellcode</a> nachlesen.

<pre>
/// &lt;summary&gt;MIDI Datei lesen und Nachricht verstekcken bzw. auslesen&lt;/summary&gt;
/// &lt;param name="srcFileName"&gt;Name der "sauberen" MIDI Datei&lt;/param&gt;
/// &lt;param name="dstFileName"&gt;Name der Zieldatei&lt;/param&gt;
/// &lt;param name="secretMessage"&gt;Die geheime Nachricht,
///                oder ein leerer Stream f&uuml;r die extrahierte Nachricht&lt;/param&gt;
/// &lt;param name="key"&gt;Das Schl&uuml;sselmuster legt fest, welche ProgChg Ereignisse ausgelassen werden&lt;/param&gt;
/// &lt;param name="extract"&gt;true: Eine Nachricht aus [srcFileName] extrahieren;
///                false: Eine Nachricht in [srcFileName] verstecken&lt;/param&gt;
public void HideOrExtract(String srcFileName, String dstFileName,
                          Stream secretMessage, Stream key, bool extract){

        //Quelldatei &ouml;ffnen
        FileStream srcFile = new FileStream(srcFileName, FileMode.Open);
        srcReader = new BinaryReader(srcFile);
        //Stream f&uuml;r die resultierende MIDI Datei initialisieren
        dstWriter = null;
        if(dstFileName != null){
                FileStream dstFile = new FileStream(dstFileName, FileMode.Create);
                dstWriter = new BinaryWriter(dstFile);
        }

        //Wenn das Flag true ist, wird der Rest der Queldatei unver&auml;ndert kopiert
        bool isMessageComplete = false;
        //Enth&auml;lt die gerade bearbeitete Nachricht
        MidiMessage midiMessage = new MidiMessage();

        //Date-Header lesen

        MidiFileHeader header = new MidiFileHeader();

        //Typ lesen
        header.HeaderType = CopyChars(4);
        header.DataLength = new byte[4];
        header.DataLength = CopyBytes(4);

        //Typ pr&uuml;fen
        if((new String(header.HeaderType) != "MThd")
                ||(header.DataLength[3] != 6)){
                MessageBox.Show("Keine Standard-MIDI Datei!");
                srcReader.Close();
                dstWriter.Close();
                return;
        }

        //Es ist eine Standard-MIDI Datei - Rest des Headers lesen

        //Diese Werte sind Int16, in umgekehrter Byte-Reihenfolge
        header.FileType = (Int16)(CopyByte()*16 + CopyByte());
        header.CountTracks = (Int16)(CopyByte()*16 + CopyByte());
        header.Division = (Int16)(CopyByte()*16 + CopyByte());
</pre>

Damit haben wir den Datei-Header &uuml;berwunden und erwarten den ersten Track-Header.
Es ist an der Zeit, das erste Paar von Halbbytes zu lesen, und dann in den Track einzutauchen.

<pre>
//Erstes geheimes Byte lesen, oder das Byte zum Extrahieren zur&uuml;cksetzen
byte[] currentMessageByte = extract
        ? new byte[2]{0,0}
        : SplitByte((byte)secretMessage.ReadByte());
//Index f&uuml;r das currentMessageByte Array initialisieren
byte currentMessageByteIndex = 0;

//Z&auml;hler f&uuml;r die zu Track hinzugef&uuml;gten Bytes initialisieren
Int32 countBytesAdded = 0;

//Erster Byte aus dem Schl&uuml;ssel lesen (0 wenn kein Schl&uuml;ssel verwendet wird)
int countIgnoreMessages = GetKeyByte(key);

//F&uuml;r alle Tracks
for(int track=0; track&lt;header.CountTracks; track++){

        if(srcReader.BaseStream.Position == srcReader.BaseStream.Length){
                break; //keine weiteren Tracks vorhanden
        }

        //Track-Header lesen

        MidiTrackHeader th = new MidiTrackHeader();
        th.HeaderType = CopyChars(4);
        if(new String(th.HeaderType) != "MTrk"){
                //Kein Standard-Track - n&auml;chsten Track suchen
                while(srcReader.BaseStream.Position+4 &lt; srcReader.BaseStream.Length){
                        th.HeaderType = CopyChars(4);
                        if(new String(th.HeaderType) == "MTrk"){
                                break; //Standard-Track gefunden
                        }
                }
        }

        //Position des L&auml;ngenfeldes merken
        //Sp&auml;ter muss hier der Wert ge&auml;ndert werden,
        //weil die L&auml;nge des Tracks sich &auml;ndern wird
        int trackLengthPosition = (dstWriter == null) ? 0
                : (int)dstWriter.BaseStream.Position;

        //L&auml;ngenfeld lesen und zu Int32 konvertieren
        //srcReader.ReadInt32() gibt wegen der Big Endian Reihenfolge
        //einen falschen Wert zur&uuml;ck,

        byte[] trackLength = new byte[4];
        trackLength = CopyBytes(4);

        th.DataLength = trackLength[0] &lt;&lt; 24;
        th.DataLength += trackLength[1] &lt;&lt; 16;
        th.DataLength += trackLength[2] &lt;&lt; 8;
        th.DataLength += trackLength[3];
</pre>

Der Header ist geschafft, weiter gehts mit den Nachrichten.
Normalerweise enthalten die ersten Nachrichten Non-MIDI Information wie Songname und -Text.
Wir k&ouml;nnen sie in die Zeildatei kopieren, ohne uns mit dem Inhalt aufzuhalten.

<pre>
bool isEndOfTrack = false; //Track f&auml;ngt erst an
countBytesAdded = 0; //noch keine Bytes hinzugef&uuml;gt
while( ! isEndOfTrack){

        /* Nachrichten lesen
         * 1. Feld: Zeit - variable L&auml;nge
         * 2. feld: Typ und Kanal - 1 Byte
         *    Untere vier Bits enthalten den Kanal (0-15),
         *    obere vier Bits en Typ (8-F)
         * 3. und 4. Feld: Parameter - je 1 Byte */

        ReadMidiMessageHeader(ref midiMessage);

        if(midiMessage.MessageType == 0xFF){ //Non-MIDI Ereignis
                if(dstWriter != null){
                        dstWriter.Write(midiMessage.Time);
                        dstWriter.Write(midiMessage.MessageType);
                }
                byte name = CopyByte();
                int length = (int)CopyVariableLengthValue();
                CopyBytes(length);

                if((name == 0x2F)&amp;&amp;(length == 0)){ // End Of Track
                        isEndOfTrack = true;
                }
        }
</pre>

Die MIDI Nachrichten sind interessanter. Wir m&uuml;ssen die Kanalnummer (untere vier Bits) entfernen,
um den Nachrichtentyp zu erhalten. Dann k&ouml;nnen wir pr&uuml;fen, ob wir ein Program Change gefunden haben.

<pre>
else{
        //Untere vier Bits zur&uuml;cksetzen, um die Kanalnummer zu entfernen
        byte cleanMessageType = (byte)(((byte)(midiMessage.MessageType &gt;&gt; 4)) &lt;&lt; 4);

        if((cleanMessageType != 0xC0)&amp;&amp;(dstWriter != null)){
                //Kein "program change" - Kopieren
                dstWriter.Write(midiMessage.Time);
                dstWriter.Write(midiMessage.MessageType);
        }

        switch(cleanMessageType){
                case 0x80: //Note Off - Note und Velocity folgen
                case 0x90: //Note On - Note und Velocity folgen
                case 0xA0: //After Touch - Note und Pressure folgen
                case 0xB0: //Control Change - Control und Value folgen
                case 0xD0: //Channel Pressure - Value folgt
                case 0xE0:{ //Pitch Wheel - 14-Bit-Wert folgt
                        CopyBytes(2); //Datenbytes kopieren
                        break;
                }
                case 0xF0: { //SysEx - keine L&auml;nge, bis zum End-Tag 0xF7 lesen
                        byte b=0;
                        while(b != 0xF7){
                                b = CopyByte();
                        }
                        break;
                }
                case 0xC0:{ //Program Change - Programmnummer folgt
</pre>

Wir haben eine Program Change Nachricht gefunden. Anh&auml;ngig von der Anzahl aller Program Change Nachrichten
m&uuml;ssen wir ein oder mehrere 4-Bit-Pakete hier verstecken (&quot;Block Gr&ouml;sse&quot;).
Um die Nachricht sp&auml;ter zu extrahieren, m&uuml;ssen wir diese Block Gr&ouml;sse kennen,
darum werden wir sie als Erstes verstecken, und entsprechend als Erstes extrahieren.

<pre>
                //Programmnummer lesen
                midiMessage.MessageData = srcReader.ReadBytes(1);

                if( ! isHalfBytesPerMidiMessageFinshed){
                        //Die Anzahl von Halbbytes pro MIDI Nachricht wurde
                        //noch nicht geschrieben/gelesen - Jetzt erledigen
                        if(extract){
                                //Block Gr&ouml;sse lesen
                                halfBytesPerMidiMessage = midiMessage.MessageData[0];
                                countBytesAdded -= midiMessage.Time.Length + 2;

                                //N&auml;chste Nachricht lesen
                                ReadMidiMessageHeader(ref midiMessage);
                                //Get program number
                                midiMessage.MessageData = srcReader.ReadBytes(1);

                        }else{
                                //Block Gr&ouml;sse schreiben
                                MidiMessage msg = new MidiMessage(midiMessage,
                                       new byte[1]{halfBytesPerMidiMessage});
                                WriteMidiMessage(msg);
                                countBytesAdded += midiMessage.Time.Length + 2;
                        }
                        isHalfBytesPerMidiMessageFinshed = true;
                }

                //Einen Block von 4-Bit-Paketen verstecken
                //und dahinter das originale Program Change
                ProcessMidiMessage(midiMessage, secretMessage, key, extract,
                        ref isMessageComplete, ref countIgnoreMessages,
                        ref currentMessageByte, ref currentMessageByteIndex,
                        ref countBytesAdded);

                break;
        } //Ende &quot;case&quot;
}}} //Ende &quot;switch&quot;, &quot;else&quot;, &quot;while&quot;
</pre>

Haben wir etwas vergessen? Ja, wir haben Nachrichten zum Track hinzugef&uuml;gt,
also stimmt die im Header angegebene L&auml;nge nicht mehr.
Wir m&uuml;ssen zum Header zur&uuml;ckkehren und das alte L&auml;ngenfeld &uuml;berschreiben.

<pre>
                if(dstWriter != null){
                        //L&auml;ngenfeld im Track-Header &auml;ndern
                        th.DataLength += countBytesAdded;
                        trackLength = IntToArray(th.DataLength);
                        dstWriter.Seek(trackLengthPosition, SeekOrigin.Begin);
                        dstWriter.Write(trackLength);
                        dstWriter.Seek(0, SeekOrigin.End);
                }

        }//Ende &quot;for&quot; &uuml;ber die Tracks
} //Ende der Methode
</pre>

Jetzt ist es aber wirklich Zeit, auf den Punkt zu kommen und die geheime Nachricht zu verstecken.
Die Methode <code>ProcessMidiMessage</code> entscheidet nur, ob versteckt oder extrahiert wird, und ruft
<code>ProcessMidiMessageH</code> oder <code>ProcessMidiMessageE</code> auf.
<code>ProcessMidiMessageH</code> versteckt mehrere Bl&ouml;cke und kopiert dann das Original MIDI Ereignis:


<pre>
...

//So viele 4-Bit-Pakete wie angegeben verstecken
for(int n=0; n&lt;halfBytesPerMidiMessage; n++){
        //Neue Nachricht mit gleichem Inhalt aber leerem Datenbyte erstellen
        MidiMessage msg = new MidiMessage(midiMessage,
                new byte[midiMessage.MessageData.Length]);

        //Neue Nachricht f&uuml;llen und in Zieldatei schreiben
        isMessageComplete = HideHalfByte(msg, secretMessage,
                ref currentMessageByte, ref currentMessageByteIndex, ref countBytesAdded);

        if(isMessageComplete){ break; }
}

...

//Original Nachricht kopieren
WriteMidiMessage(midiMessage);

...

private bool HideHalfByte(MidiMessage midiMessage, Stream secretMessage,
                ref byte[] currentMessageByte, ref byte currentMessageByteIndex,
                ref int countBytesAdded){

        bool returnValue = false;
        //Aktuelles Nachrichten-Byte ins Datenbyte der MIDI Nachricht setzen
        midiMessage.MessageData[0] = currentMessageByte[currentMessageByteIndex];
        //Nachricht in Zieldatei schreiben
        WriteMidiMessage(midiMessage);
        //Hinzugef&uuml;gte Bytes z&auml;hlen
        countBytesAdded += midiMessage.Time.Length + 1 + midiMessage.MessageData.Length;

        //Weiter mit dem  n&auml;chsten Halbbyte

        currentMessageByteIndex++;

        if(currentMessageByteIndex == 2){
                int nextValue = secretMessage.ReadByte();
                if(nextValue &lt; 0){
                        returnValue = true;
                }else{
                        currentMessageByte = SplitByte( (byte)nextValue );
                        currentMessageByteIndex = 0;
                }
        }

        return returnValue; //true wenn die Nachricht vollst&auml;ndig versteckt ist
}
</pre>

Mehr braucht man nicht, um Informationen in einer MIDI Datei zu verstecken. Gar nicht so schwer, oder?
<code>ProcessMidiMessageE</code> dreht den Prozess um:

<pre>
...

for(int n=0; n&lt;halfBytesPerMidiMessage; n++){

        ExtractHalfByte(midiMessage, secretMessage,
                        ref currentMessageByte, ref currentMessageByteIndex,
                        ref countBytesAdded);

        if((secretMessage.Length==8)&amp;&amp;(secretMessageLength==0)){
                //er geheime Nachrichten-Stream enthielt dieL&auml;nge er Nachricht
                //in den ersten 8 Bytes - Entfernen.
                secretMessage.Seek(0, SeekOrigin.Begin);
                byte[] bytes = new byte[8];
                secretMessage.Read(bytes, 0, 8);
                secretMessageLength = ArrayToInt(bytes);
                secretMessage.SetLength(0);
        }
        else if((secretMessageLength &gt; 0)&amp;&amp;(secretMessage.Length==secretMessageLength)){
                //Ale Bytes ausgelesen - weitere Program Change Nachrichten ignorieren
                isMessageComplete = true;
                break;
        }

        if((n+1)&lt;halfBytesPerMidiMessage){
                //Weitere versteckte Pakete folgen - n&auml;chsten Header lesen
                ReadMidiMessageHeader(ref midiMessage);
                midiMessage.MessageData = srcReader.ReadBytes(1);
        }
}

...

private void ExtractHalfByte(MidiMessage midiMessage, Stream secretMessage,
                ref byte[] currentMessageByte, ref byte currentMessageByteIndex,
                ref int countBytesAdded){

        //Gefundenes Halbbyte kopieren
        currentMessageByte[currentMessageByteIndex] = midiMessage.MessageData[0];

        //Entfernte (&quot;negativ hinzugef&uuml;gte&quot;) Bytes z&auml;hlen: Zeit, Typ, Parameter
        countBytesAdded -= midiMessage.Time.Length + 1 + midiMessage.MessageData.Length;


        //Weiter zum n&auml;chsten Halbbyte
        currentMessageByteIndex++;
        if(currentMessageByteIndex == 2){
                //Extrahierts Bytes schreiben
                byte completeMessageByte = (byte)((currentMessageByte[0]&lt;&lt;4) + currentMessageByte[1]);
                secretMessage.WriteByte(completeMessageByte);

                currentMessageByte[0]=0;
                currentMessageByte[1]=0;
                currentMessageByteIndex = 0;
        }
}
</pre>

## Konvertierung zwischen Big-Endian und Little-Endian

Wahrscheinlich hast Du die Methoden <code>IntToArray</code> und <code>ArrayToInt</code> bemerkt.
Diese Beiden konvertieren Integers zwischen dem von C# verwendeten Little-Endian Format und den
Big-Endian Byte Arrays, die wir brauchen um MIDI Dateien zu lesen/schreiben. Zum Beispiel ist in einer MIDI Datei
der Int16-Wert 12345 als &quot;0x30 0x39&quot; abgelegt. Das h&ouml;here Byte steht links vom niedrigeren Byte!
C# erwartet das h&ouml;here Byte rechts vom Niedrigeren, es speichert Integers von niedrig nach hoch.
Darum kann man hier keine Funktionen wie <code>BinaryReader.ReadInt16</code> verwenden.
Verwendbar sind <code>ReadChars</code> und <code>ReadBytes</code>, aber alles andere w&uuml;rde
die Byte-Reihenfolge umdrehen. Kein Problem, wir k&ouml;nnen Integer-Werte
Byte f&uuml;r Byte lesen, und dann alle Bytes in eine Integer-Variable schieben:

<pre>
public static byte[] IntToArray(Int64 val){
        //64 bits f&uuml;r den Int64 initialisieren
        byte[] bytes = new byte[8];
        for(int n=0; n&lt;8; n++){
                //Den Int64 nach rechts schieben und das niedrigste Byte abschneiden
                bytes[n] = (byte)(val &gt;&gt; (n*8));
        }
        return bytes;
}

public Int64 ArrayToInt(byte[] bytes){
        //Ein Little-Endian Int64 initialisieren
        Int64 result = 0;
        for(int n=0; n&lt;bytes.Length; n++){
                //Die Bytes in die Int64-Variable schieben
                result += (bytes[n] &lt;&lt; (n*8));
        }
        return result;
}
</pre>

