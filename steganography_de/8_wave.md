---
title: Wave-Dateien
category: Steganografie
subcategory: Audiodaten
order: 8
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 20.8 Kb]({% link /assets/steganodotnet8_src.zip %})

## Worum geht es?


Nachdem wir nun in Bitmaps, MIDI Tracks und .NET Assemblies Daten versteckt haben,
vermisst Du vielleicht ein wichtiges Dateiformat. Vielleicht vermisst Du die Dateien,
die sehr viele Bytes verstecken k&ouml;nnen, ohne gr&ouml;&szlig;er zu werden, und sich
in wenigen Sekunden erstellt lassen, so dass man keine Originaldateien auf der Festplatte
speichern muss. Es ist an der Zeit, Wave Audio zur Liste hinzuzuf&uuml;gen.


Dieser Artikel verwendet Code aus
<a href="http://www.codeproject.com/cs/media/cswavrec.asp" target="_blank">A full-duplex audio player in C# using the waveIn/waveOut APIs</a>.

## Das Wave Dateiformat


Hast Du schon einmal eine Wave Date im HEX Editor angeschaut?
Sie beginnt so, und geht mit unlesbaren Bin&auml;rdaten weiter:


<img src="/images/steganodotnet86.gif" width="523" height="35">


Jede RIFF Datei beginnt mit dem Text &quot;RIFF&quot;, gefolgt von der
<code>Int32</code> L&auml;nge der gesamten Datei:


<img src="/images/steganodotnet81.gif" width="184" height="36">

Die n&auml;chsten Felder sagen, dass diese RIFF Datei Wave-Daten enth&auml;lt, und &ouml;ffnen den Format-Chunk:

<img src="/images/steganodotnet82.gif" width="184" height="35">

Die L&auml;nge des folgenden Format-Chunks muss f&uuml;r PCM Dateien 16 sein:

<img src="/images/steganodotnet83.gif" width="140" height="35">

Jetzt wird mit einer <code>WAVEFORMATEX</code> Struktur das Format angegeben:

<img src="/images/steganodotnet84.gif" width="356" height="83">


Nach dem Format-Chunk k&ouml;nnen noch Extra-Informationen stehen.
Der interessante Teil beginnt mit dem <code>data</code> Chunk.


<img src="/images/steganodotnet85.gif" width="198" height="38">


Der Daten-Chunk enth&auml;lt alle Wave Samples. Dass hei&szlig;t, der Rest der Datei
besteht aus reinen Audio-Daten. Kleine &Auml;nderungen k&ouml;nnen eventuell h&ouml;rbar sein,
aber nicht die Datei zerst&ouml;ren.


## Die Nachricht verstecken


Eine Nachricht in Wave Samples zu verstecken funktioniert fast genauso wie in den Pixeln
einer Bitmap. Wieder verwenden wir einen Schl&uuml;ssel-Stream, um eine Anzahl von Tr&auml;gereinheiten
(Samples/Pixel) zu &uuml;berspringen, greifen eine Tr&auml;gereinheit heraus, setzen ein Bit der
Nachricht in ihr niedrigstes Bit, und schreiben die ge&auml;nderte Einheit in den Ziel-Stream.
Nachdem die ganze Nachricht so versteckt wurde, kopieren wir den Rest des Tr&auml;ger-Streams.


<pre>
public void Hide(Stream messageStream, Stream keyStream){

        byte[] waveBuffer = new byte[bytesPerSample];
        byte message, bit, waveByte;
        int messageBuffer; //receives the next byte of the message or -1
        int keyByte; //distance of the next carrier sample

        //Schleife &uuml;ber die Nachricht, jedes Byte verstecken
        while( (messageBuffer=messageStream.ReadByte()) >= 0 ){
                //read one byte of the message stream
                message = (byte)messageBuffer;

                //f&uuml;r jedes Bit in [message]
                for(int bitIndex=0; bitIndex&lt;8; bitIndex++){

                        //ein Byte vom Schl&uuml;ssel-Stream lesen
                        keyByte = GetKeyValue(keyStream);

                        //[keyByte] Samples &uuml;berspringen
                        for(int n=0; n&lt;keyByte-1; n++){
                                //ein Sample aus dem sauberen Stream in den Tr&auml;ger-Stream kopieren
                                sourceStream.Copy(
                                        waveBuffer, 0,
                                        waveBuffer.Length, destinationStream);
                        }

                        //ein Sample aus dem Wave-Stream lesen
                        sourceStream.Read(waveBuffer, 0, waveBuffer.Length);
                        waveByte = waveBuffer[bytesPerSample-1];

                        //n&auml;chstes Bit des aktuellen Nachrichten-Bytes holen...
                        bit = (byte)(((message & (byte)(1 &lt;&lt; bitIndex)) &gt; 0) ? 1 : 0);

                        //...und ins letzte Bit des Samples schreiben
                        if((bit == 1) && ((waveByte % 2) == 0)){
                                waveByte += 1;
                        }else if((bit == 0) && ((waveByte % 2) == 1)){
                                waveByte -= 1;
                        }

                        waveBuffer[bytesPerSample-1] = waveByte;

                        //Ergebnis in den Ziel-Stream schreiben
                        destinationStream.Write(waveBuffer, 0, bytesPerSample);
                }
        }

        //Rest der Wave unver&auml;ndert kopieren
        //...
}
</pre>

## Die Nachricht auslesen


Wieder verwenden wir den Schl&uuml;ssel-Stream, um die richtigen Samples zu finden, genauso wie
vorher beim Verstecken der Nachricht.
Dann lesen wir das letzte Bit des jeweiligen Samples und schieben es ins aktuelle Byte der Nachricht.
Wenn das Byte vollst&auml;ndig ist, schreiben wir es in den Nachrichten-Stream, und machen
mit dem N&auml;chsten weiter.


<pre lang=cs>
public void Extract(Stream messageStream, Stream keyStream){

        byte[] waveBuffer = new byte[bytesPerSample];
        byte message, bit, waveByte;
        int messageLength = 0; //expected length of the message
        int keyByte; //distance of the next carrier sample

        while( (messageLength==0 || messageStream.Length&lt;messageLength) ){
                //Nachrichten-Byte zur&uuml;cksetzen
                message = 0;

                //f&uuml;r jedes Bit in [message]
                for(int bitIndex=0; bitIndex&lt;8; bitIndex++){

                        //ein Byte vom Schl&uuml;ssel-Stream lesen
                        keyByte = GetKeyValue(keyStream);

                        //[keyByte] Samples auslassen
                        for(int n=0; n&lt;keyByte; n++){
                                //ein Sample aus dem Wave-Stream lesen
                                sourceStream.Read(waveBuffer, 0, waveBuffer.Length);
                        }
                        waveByte = waveBuffer[bytesPerSample-1];

                        //letztes Bit des Samples holen...
                        bit = (byte)(((waveByte % 2) == 0) ? 0 : 1);

                        //...und ins Nachrichten-Byte schreiben
                        message += (byte)(bit &lt;&lt; bitIndex);
                }

                //rekonstruiertes Byte zur Nachricht hinzuf&uuml;gen
                messageStream.WriteByte(message);

                if(messageLength==0 && messageStream.Length==4){
                        //die ersten 4 Bytes enthalten die L&auml;nge der Nachricht
                        //...
                }
        }
}
</pre>

## Einen Klang aufzeichnen


Die originalen, sauberen Tr&auml;gerdateien aufzuheben, kann gef&auml;hrlich sein.
Jemand der eine Tr&auml;gerdatei mit geheimer Nachricht darin hat, und es schafft, die
Original-Datei ohne Nachricht zu bekommen, kann einfach die beiden Dateien vergleichen,
die Abst&auml;nde zwischen jeweils zwei unterschiedlichen Samples z&auml;hlen,
und so schnell den Schl&uuml;ssel rekonstruieren.



Deshalb m&uuml;ssen wir die sauberen Tr&auml;gerdateien l&ouml;schen und zerst&ouml;ren nachdem
sie einmal verwendet wurden, oder einen Sound <i>on the fly</i> aufzeichnen.
Dank
<a href="http://www.codeproject.com/cs/media/cswavrec.asp" target="_blank">Ianier Munoz' WaveInRecorder</a>
ist es kein Problem, Wave-Daten aufzunehmen und eine Nachricht darin zu verstecken, bevor irgendetwas
auf der Festplatte gespeichert wird. Es gibt keine Original-Datei, um die wir uns Sorgen machen m&uuml;ssten.
Im Hauptformular kann der Benutzer w&auml;hlen, ob er eine vorhandene Wave-Datei verwenden, oder hier und jetzt
einen Sound aufzeichnen m&ouml;chte.
Wenn er einen einzigartigen, nicht reproduzierbaren Sound aufnehmen will, kann er ein Mikrofon
anschlie&szlig;en und sprechen/spielen/... was immer ihm einf&auml;llt:


<pre>
if(rdoSrcFile.Checked){
        //eine .wav Datei als Tr&auml;ger verwenden
        //beschwer dich nachher nicht, du wurdest schlie&szlig;lich gewarnt
        sourceStream = new FileStream(txtSrcFile.Text, FileMode.Open);
}else{
        //einen Tr&auml;ger-Klang aufzeichnen
        frmRecorder recorder = new frmRecorder(countSamplesRequired);
        recorder.ShowDialog(this);
        sourceStream = recorder.RecordedStream;
}
</pre>


<code>frmRecorder</code> ist eine kleine Oberfl&auml;che f&uuml;r den WaveIn Recorder,
die die aufgenommenen Samples mitz&auml;hlt und einen Stop-Button aktiviert, sobald der
Sound lang genug ist um die angegebene Nachricht zu verstecken.


<img src="/images/steganodotnet87.png" width="314" height="116">


Der neue Sound wird in einem <code>MemoryStream</code> abgelegt und an
<code>WaveUtility</code> weitergereicht. Von da an ist es egal, wo der Stream her kam,
<code>WaveUtility</code> macht keinen Unterschied zwischen aus Dateien gelesenen und <i>on the fly</i>
aufgezeichneten Kl&auml;ngen.


<pre>
WaveUtility utility = new WaveUtility(sourceStream, destinationStream);
utility.Hide(messageStream, keyStream);
</pre>

