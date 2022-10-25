---
title: Einstieg
category: Steganografie
subcategory: Bilder
order: 001
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 9.43 Kb]({% link /assets/steganodotnet_src.zip %})

## Worum geht es?

Dieser Artikel versucht zu erkl&auml;ren, wie beliebige Daten im Rauschen von Bildern versteckt werden k&ouml;nnen.
Es gibt nur einen sicheren Ort f&uuml;r private Daten: Den Ort, an dem niemand danach sucht.<br>
Eine mit Algorithmen wie PGP verschl&uuml;sselte Datei ist nicht lesbar, aber jeder sieht
sofort, dass etwas versteckt wird. W&auml;re es nicht gut, wenn  jeder Deine verschl&uuml;sselten
Dateien &ouml;ffnen k&ouml;nnte, und anstelle der Geheimnisse unscharfe Fotos vom letzten
Urlaub sehen w&uuml;rde? Kaum jemand w&uuml;rde nach Teilen verschl&uuml;sselter Nachrichten
in den Pixel-Werten suchen.

## Wie es funktioniert

Um zu sehen, wie die Anwendung arbeitet, solltest Du [den Quellcode anschauen]({% link /assets/steganodotnet_src.zip %}).

Allgemein dargestellt werden Nachrichten so versteckt:

![How to embed](/images/steganodotnet11.gif)

Und so wieder extrahiert:

![How to extract](/images/steganodotnet12.gif)

Zuerst schreibt das Programm die L&auml;nge der Nachricht ins erste Pixel.
Wir brauchen diesen Wert sp&auml;ter, um die Nachricht wieder zu extrahieren.

<pre>messageLength = (Int32)messageStream.Length;

//hier werden eventuelle Konflikte abgefangen
//...
//L&auml;nge der Nachricht ins erste Pixel schreiben
int colorValue = messageLength;
int red = colorValue &gt;&gt; 2;
colorValue -= red &lt;&lt; 2;
int green = colorValue &gt;&gt; 1;
int blue = colorValue - (green &lt;&lt; 1);
pixelColor = Color.FromArgb(red, green, blue);
bitmap.SetPixel(0,0, pixelColor);
</pre>

Anschlie&szlig;end liest es ein Byte aus dem Schl&uuml;ssel-Stream, um die Position des n&auml;chsten
zu verwendenden Pixels zu berechnen:

<pre>//mit dem zweiten Pixel anfangen
Point pixelPosition = new Point(1,0);

//F&uuml;r jedes Byte der Nachricht
for(int messageIndex=0; messageIndex&lt;messageLength; messageIndex++){
        //repeat the key, if it is shorter than the message
        if(keyStream.Position == keyStream.Length){
        keyStream.Seek(0, SeekOrigin.Begin);
}
//N&auml;chste Pixel-Anzahl aus dem Schl&uuml;ssel lesen, "1" f&uuml;r 0 verwenden
currentStepWidth = keyStream.ReadByte() + 1;

//Zeile wechseln, falls aktuelle Schrittweite den Rand &uuml;berschreitet
while(currentStepWidth &gt; bitmapWidth){
        currentStepWidth -= bitmapWidth;
        pixelPosition.Y++;
}

//Position horizontal versetzen
if((bitmapWidth - pixelPosition.X) &lt; currentStepWidth){
        pixelPosition.X = currentStepWidth - (bitmapWidth - pixelPosition.X);
        pixelPosition.Y++;
}else{
        pixelPosition.X += currentStepWidth;
}</pre>

Jetzt lesen wir das Pixel und ersetzen eine Farb-Komponente mit dem Nachrichten-Byte
(oder alle Komponenten, wenn schwarz/weisses Rauschen entstehen soll):

<pre> //Farbe des "sauberen" Pixels lesen
 pixelColor = bitmap.GetPixel(pixelPosition.X, pixelPosition.Y);

 //Um etwas Verwirrung hinzuzuf&uuml;gen, wird das Byte mit der aktuellen Schrittweite kombiniert
 int currentByte = messageStream.ReadByte() ^ currentKeyByte;

 if(useGrayscale){
  pixelColor = Color.FromArgb(currentByte, currentByte, currentByte);
 }else{
  //Eine Farb-Komponente ersetzen
  SetColorComponent(ref pixelColor, currentColorComponent, currentByte);
  //F&uuml;r das n&auml;chste Byte eine andere Komponente verwenden
  currentColorComponent = (currentColorComponent==2) ? 0 : (currentColorComponent+1);
 }
} //Ende "for" - weiter zum n&auml;chsten Byte</pre>

Wenn das Programm eine versteckte Nachricht ausliest, liest es die L&auml;nge der Nachricht und
die Farb-Komponenten, anstatt sie in die Pixel zu schreiben. So wird die L&auml;nge aus dem ersten Pixel gelesen:

<pre>pixelColor = bitmap.GetPixel(0,0);
messageLength = (pixelColor.R &lt;&lt; 2) + (pixelColor.G &lt;&lt; 1) + pixelColor.B;
messageStream = new MemoryStream(messageLength);</pre>

Die Pixel-Koordinaten werden genauso bestimmt wie bereits beschrieben.
Anschlie&szlig;end wird das versteckte Byte aus dem Farbwert gelesen:

<pre> //Farbe des ge&auml;nderten Pixels lesen
 pixelColor = bitmap.GetPixel(pixelPosition.X, pixelPosition.Y);
 //Das ersteckte Nachrichten-Byte aus der Farbe extrahieren
 byte foundByte = (byte)(currentKeyByte ^ GetColorComponent(pixelColor, currentColorComponent));
 messageStream.WriteByte(foundByte);
 //F&uuml;r das n&auml;chste Byte eine andere Komponente verwenden
  currentColorComponent = (currentColorComponent==2) ? 0 : (currentColorComponent+1);
} //Ende "for" - weiter zum n&auml;chsten Byte</pre>

