---
title: Mehrere Trägerbilder
category: Steganografie
subcategory: Bilder
order: 2
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 19.5 Kb]({% link /assets/steganodotnet2_src.zip %})

## Worum geht es?

Dieser Artikel erweitert die Anwendung aus [Teil 1]({% link steganography_de/1_intro.md %})
um das Aufteilen der Nachricht auf mehrere Tr&auml;ger-Bitmaps.

## Die Information &uuml;ber mehrere Bilder verteilen

Wenn Du richtig paranoid bist, wirst Du Deiner Mailbox nicht vertrauen. Nat&uuml;rlich wirst Du auch
nicht dem Brieftr&auml;ger vertrauen. Das bedeutet, dass Du es nicht wagst, die Tr&auml;ger-Bitmap in
einem St&uuml;ck zu verschicken.
Das ist kein Problem mehr, denn Du kannst <i>eine</i> Nachricht in <i>mehreren</i> Bitmaps verstecken,
als w&uuml;rde man einen Text &uuml;ber mehrere Seiten schreiben. Auf diese Anwendung bezogen heisst es,
die Pixel &uuml;ber mehrere Bilder zu verteilen.

So wie du einen Papierbrief zerschneiden und in mehrere hohle Baustämme stecken kannst...

![multiple sheets of paper](/images/steganodotnet23.png)

... kannst du die Bits in mehreren Grafiken verstecken:

![multiple images](/images/steganodotnet24.png)

Du kannst jedes Bild in einer separaten eMail verschicken, in verschiedenen Postf&auml;chern hinterlegen,
oder auf verschiedenen Festplatten speichern.
Die Oberfl&auml;che erlaubt es, mehrere Tr&auml;ger-Bitmaps genauso wie Schl&uuml;ssel-Dateien auszuw&auml;hlen.
Die Auswahl wird in einem Array vom Typ <code>CarrierImage</code> gespeichert.

<pre>
public struct CarrierImage{
        //Name des "sauberen" Bildes
        public String sourceFileName;
        //Dateiname f&uuml;r das neue Bild
        public String resultFileName;
        //Breite * H&ouml;he
        public long countPixels;
        //Farbiges (false) oder monochromes (true) Rauschen in diesem Bild erzeugen
        public bool useGrayscale;
        //Anzahl der Bytes, die in diesem Bild versteckt werden - wird von HideOrExtract() gesetzt
        public long messageBytesToHide;

        public CarrierImage(String sourceFileName, String resultFileName, long countPixels, bool useGrayscale){
                this.sourceFileName = sourceFileName;
                this.resultFileName = resultFileName;
                this.countPixels = countPixels;
                this.useGrayscale = useGrayscale;
                this.messageBytesToHide = 0;
        }
}
</pre>

Gr&ouml;&szlig;ere Bilder k&ouml;nnen mehr Bytes (in mehr Pixeln) fassen als kleinere Bilder.
Diese Beispiel-Anwendung verwendet die einfachste m&ouml;gliche Verteilung:

<pre>
//Anzahl der Bytes berechnen, f&uuml;r die dieses Bild verwendet wird
for(int n=0; n&lt;imageFiles.Length; n++){
        //pixels = Anzahl der Pixel im Bild / Anzahl verf&uuml;gbarer Pixel insgesamt
        float pixels = (float)imageFiles[n].countPixels / (float)countPixels;
        //Das Bild wird (L&auml;nge der Nachricht * pixels) Bytes verstecken
        imageFiles[n].messageBytesToHide = (long)Math.Ceiling( (float)messageLength * pixels );
}
</pre>

Jetzt beginnen wir mit der ersten Tr&auml;ger-Bitmap, laufen durch die Nachricht, verstecken eine
bestimmte Anzahl an Bytes, wechseln zur zweiten Tr&auml;ger-Bitmap, und immer so weiter.

<pre>
//Aktuelle Position im Tr&auml;ger-Bitmap
//Start bei 1, weil (0,0) die L&auml;nge der Nachricht enth&auml;lt
Point pixelPosition = new Point(1,0);

//Anzahl der Bytes, die in diesem Bild bereits versteckt wurden
int countBytesInCurrentImage = 0;

//Index des aktuell verwendeten Bildes
int indexBitmaps = 0;

//Nachricht durchlaufen und jedes Byte verstecken
for(int messageIndex=0; messageIndex&lt;messageLength; messageIndex++){
        //Position des n&auml;chsten Pixels berechnen
        //...

        //Weiter zur n&auml;chsten Bitmap
        if(countBytesInCurrentImage == imageFiles[indexBitmaps].messageBytesToHide){
                indexBitmaps++;
                pixelPosition.Y = 0;
                countBytesInCurrentImage = 0;
                bitmapWidth = bitmaps[indexBitmaps].Width-1;
                bitmapHeight = bitmaps[indexBitmaps].Height-1;
                if(pixelPosition.X > bitmapWidth){ pixelPosition.X = 0; }
        }

        //Ein Byte verstecken oder extrahieren
        //...

        countBytesInCurrentImage++;
}
</pre>

Zum Schluss m&uuml;ssen wir die neuen Bilder speichern. Jedes Bild kann in einem anderen Format
gespeichert werden (bmp, tif oder png). Das neue Format hat nichts mit dem Format der Original-Datei zu tun.
Das heisst, Du kannst eine BMP-, zwei PNG und eine TIFF-Datei als Tr&auml;ger-Bilder ausw&auml;hlen, und
die Ergebnisse in drei TIFF- und einer PNG-Datei speichern.

<pre>
//...
        for(indexBitmaps=0; indexBitmaps&lt;bitmaps.Length; indexBitmaps++){
                if( ! extract ){ //Extrations-Modus &auml;ndert keine Bilder
                        //Entstandenes Bild speichern und Original schlie&szlig;en
                        SaveBitmap( bitmaps[indexBitmaps], imageFiles[indexBitmaps].resultFileName );
                }
        }
//...

private static void SaveBitmap(Bitmap bitmap, String fileName){
        String fileNameLower = fileName.ToLower();

        //Format anhand der Erweiterung bestimmen
        System.Drawing.Imaging.ImageFormat format = System.Drawing.Imaging.ImageFormat.Bmp;
        if((fileNameLower.EndsWith("tif"))||(fileNameLower.EndsWith("tiff"))){
                format = System.Drawing.Imaging.ImageFormat.Tiff;
        }else if(fileNameLower.EndsWith("png")){
                format = System.Drawing.Imaging.ImageFormat.Png;
        }

        //Bitmap kopieren
        Image img = new Bitmap(bitmap);

        //Datei schlie&szlig;en
        bitmap.Dispose();
        //Neue Bitmap speichern
        img.Save(fileName, format);
        img.Dispose();
}
</pre>

