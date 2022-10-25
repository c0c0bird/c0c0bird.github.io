---
title: AVI-Dateien
category: Steganografie
subcategory: Bilder
order: 4
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 27.2 Kb]({% link /assets/steganodotnet4_src.zip %})

## Worum geht es?

Der Video Stream in einem AVI-Film ist nichts weiter als eine Folge von Bitmaps.
In diesem Artikel geht es darum, diese Bitmaps zu extrahieren und den Stream anschlie&szlig;end wieder
zusammen zu setzen, um auch im Video eine Nachricht verstecken zu k&ouml;nnen.
<b>Falls Du schon wei&szlig;t, wie man AVI-Videos bearbeitet, solltest Du diese Seite &uuml;berspringen,
es ist reine Zeitverschwendung.</b>

## Den Video Stream lesen

Die Windows AVI Library ist ein Satz von Funktionen in avifil32.dll. Bevor der verwendet werden kann,
muss er mit <code>AVIFileInit</code> initialisiert werden.
<code>AVIFileOpen</code> &ouml;ffnet eine Datei, <code>AVIFileGetStream</code> findet den
Video Stream. Jede dieser Funktionen belegt Speicher, der am Ende wieder freigegeben werden muss.


<pre>
//AVI Library initialisieren
[DllImport("avifil32.dll")]
public static extern void AVIFileInit();

//Eine AVI Datei &ouml;ffnen
[DllImport("avifil32.dll", PreserveSig=true)]
public static extern int AVIFileOpen(
        ref int ppfile,
        String szFile,
        int uMode,
        int pclsidHandler);

//einen Stream in einer offenen AVI Datei holen
[DllImport("avifil32.dll")]
public static extern int AVIFileGetStream(
        int pfile,
        out IntPtr ppavi,
        int fccType,
        int lParam);

//Einen offenen AVI Stream freigeben
[DllImport("avifil32.dll")]
public static extern int AVIStreamRelease(IntPtr aviStream);

//Eine offene AVI Datei freigeben
[DllImport("avifil32.dll")]
public static extern int AVIFileRelease(int pfile);

//AVI Library schlie&szlig;en
[DllImport("avifil32.dll")]
public static extern void AVIFileExit();
</pre>

Jetzt k&ouml;nnen wir eine AVI Datei &ouml;ffnen und den Video Stream finden.
AVI Dateien enthalten mehrere Streams von vier verschiedenen Typen (Video, Audio, Midi und Text).
Normalerweise existiert nur ein Stream von jedem Typ, und wir sind nur am Video Stream interessiert.


<pre>
private int aviFile = 0;
private IntPtr aviStream;

public void Open(string fileName) {
        AVIFileInit(); //Intitialize AVI library

        //Datei &ouml;ffnen
        int result = AVIFileOpen(
                ref aviFile,
                fileName,
                OF_SHARE_DENY_WRITE, 0);

        //Video Stream holen
        result = AVIFileGetStream(
                aviFile,
                out aviStream,
                streamtypeVIDEO, 0);
}
</pre>

Bevor wir die Frames auslesen k&ouml;nnen, m&uuml;ssen wir wissen, was genau wir lesen wollen:
- Wo beginnt der erste Frame?
- Wie viele Frames sind vorhanden?
- Welche H&ouml;he/Breite haben die Bilder?
Die AVI Library enth&auml;lt Funktionen f&uuml;r jede Frage.


<pre>
//Startposition eines Streams ermitteln
[DllImport("avifil32.dll", PreserveSig=true)]
public static extern int AVIStreamStart(int pavi);

//Anzahl der Frames in einem Stream ermitteln
[DllImport("avifil32.dll", PreserveSig=true)]
public static extern int AVIStreamLength(int pavi);

//Header-Infos &uuml;ber einen offenen Stream abrufen
[DllImport("avifil32.dll")]
public static extern int AVIStreamInfo(
        int pAVIStream,
        ref AVISTREAMINFO psi,
        int lSize);
</pre>

Mit diesen Funktionen k&ouml;nnen wir eine <code>BITMAPINFOHEADER</code> Struktur f&uuml;llen.
Um Bilder zu extrahieren, brauchen wir noch drei weitere Funktionen.

<pre>
//Pointer auf ein GETFRAME Objekt holen (gibt bei Fehlern 0 zur&uuml;ck)
[DllImport("avifil32.dll")]
public static extern int AVIStreamGetFrameOpen(
        IntPtr pAVIStream,
        ref BITMAPINFOHEADER bih);

//Pointer auf ein DIB holen (gibt bei Fehlern 0 zur&uuml;ck)
[DllImport("avifil32.dll")]
public static extern int AVIStreamGetFrame(
        int pGetFrameObj,
        int lPos);

//GETFRAME Object freigeben
[DllImport("avifil32.dll")]
public static extern int AVIStreamGetFrameClose(int pGetFrameObj);
</pre>

Endlich k&ouml;nnen wir die Frames entpacken...

<pre>
//Startposition und Anzahl der Frames holen
int firstFrame = AVIStreamStart(aviStream.ToInt32());
int countFrames = AVIStreamLength(aviStream.ToInt32());

//Header-Inforamtionen holen
AVISTREAMINFO streamInfo = new AVISTREAMINFO();
result = AVIStreamInfo(aviStream.ToInt32(), ref streamInfo, Marshal.SizeOf(streamInfo));

//Header f&uuml;r die Bitmaps zusammensetzen
BITMAPINFOHEADER bih = new BITMAPINFOHEADER();
bih.biBitCount = 24;
bih.biCompression = 0;
bih.biHeight = (Int32)streamInfo.rcFrame.bottom;
bih.biWidth = (Int32)streamInfo.rcFrame.right;
bih.biPlanes = 1;
bih.biSize = (UInt32)Marshal.SizeOf(bih);

//Entpacken von DIBs (device independend bitmaps) vorbereiten
int getFrameObject = AVIStreamGetFrameOpen(aviStream, ref bih);

...

//Den Frame an einer bestimmten Position exportieren
public void ExportBitmap(int position, String dstFileName){
        //Frame dekomprimieren und Pointer zum DIB zur&uuml;ckgeben
        int pDib = Avi.AVIStreamGetFrame(getFrameObject, firstFrame + position);

        //Bitmap-Header in eine verwaltete Struktur kopieren
        BITMAPINFOHEADER bih = new BITMAPINFOHEADER();
        bih = (BITMAPINFOHEADER)Marshal.PtrToStructure(new IntPtr(pDib), bih.GetType());

        //Das Bild kopieren
        byte[] bitmapData = new byte[bih.biSizeImage];
        int address = pDib + Marshal.SizeOf(bih);
        for(int offset=0; offset&lt;bitmapData.Length; offset++){
                bitmapData[offset] = Marshal.ReadByte(new IntPtr(address));
                address++;
        }

        //Bitmap-Details kopieren
        byte[] bitmapInfo = new byte[Marshal.SizeOf(bih)];
        IntPtr ptr;
        ptr = Marshal.AllocHGlobal(bitmapInfo.Length);
        Marshal.StructureToPtr(bih, ptr, false);
        address = ptr.ToInt32();
        for(int offset=0; offset&lt;bitmapInfo.Length; offset++){
                bitmapInfo[offset] = Marshal.ReadByte(new IntPtr(address));
                address++;
        }
</pre>

...und in einer Bitmap-Datei speichern.

<pre>
        //Header aufbauen
        Avi.BITMAPFILEHEADER bfh = new Avi.BITMAPFILEHEADER();
        bfh.bfType = Avi.BMP_MAGIC_COOKIE;
        bfh.bfSize = (Int32)(55 + bih.biSizeImage); //Gr&ouml;&szlig;e der gespeicherten Datei
        bfh.bfOffBits = Marshal.SizeOf(bih) + Marshal.SizeOf(bfh);

        //Zieldatei erstellen oder &uuml;berschreiben
        FileStream fs = new FileStream(dstFileName, System.IO.FileMode.Create);
        BinaryWriter bw = new BinaryWriter(fs);

        //Header schreiben
        bw.Write(bfh.bfType);
        bw.Write(bfh.bfSize);
        bw.Write(bfh.bfReserved1);
        bw.Write(bfh.bfReserved2);
        bw.Write(bfh.bfOffBits);
        //Details schreiben
        bw.Write(bitmapInfo);
        //Write bitmap data
        bw.Write(bitmapData);
        bw.Close();
        fs.Close();
} //end of ExportBitmap
</pre>

Die Anwendung kann die extrahierten Bitmaps genauso verwenden wie jedes andere Bild.
Wenn eine Tr&auml;ger-Datei ein AVI Video ist, wird der erste Frame in eine tempor&auml;re Datei extrahiert,
ge&ouml;ffnet und ein Teil der Nachricht darin versteckt. Anschlie&szlig;end wird die ge&auml;nderte Bitmap
in einen neuen Stream geschrieben und mit dem n&auml;chsten Frame weiter gearbeitet.
Nach dem letzten Frame schlie&szlig;t die Anwendung beide Video Dateien, l&ouml;scht die tempor&auml;ren
Bitmap Dateien, und macht mit der n&auml;chsten Tr&auml;ger-Datei weiter.


## Einen Video Stream schreiben

Wenn die Anwendung eine AVI Tr&auml;ger-Datei &ouml;ffnet, erstellt sie auch eine weitere AVI Datei
f&uuml;r die resultierenden Bitmaps.
Der neue Video Stream muss die gleiche H&ouml;he/Breite und Frame Rate haben wie das Original,
darum k&ouml;nnen wir ihn nicht gleich in der <code>Open()</code> Methode anlegen.
Wenn die erste Bitmap extrahiert wurde wissen wir das Format der Frames und k&ouml;nnen den Video Stream erstellen.
Die Funktionen zum Anlegen von Streams und Hinzuf&uuml;gen von Frames sind
<code>AVIFileCreateStream</code>, <code>AVIStreamSetFormat</code> und <code>AVIStreamWrite</code>:

<pre>
//Neuen Strean in einer vorhandenen AVI Datei erstellen
[DllImport("avifil32.dll")]
public static extern int AVIFileCreateStream(
        int pfile,
        out IntPtr ppavi,
        ref AVISTREAMINFO ptr_streaminfo);

//Format eines neuen Stram festlegen
[DllImport("avifil32.dll")]
public static extern int AVIStreamSetFormat(
        IntPtr aviStream, Int32 lPos,
        ref BITMAPINFOHEADER lpFormat, Int32 cbFormat);

//Einen Frame in einen Stream schreiben
[DllImport("avifil32.dll")]
public static extern int AVIStreamWrite(
        IntPtr aviStream, Int32 lStart, Int32 lSamples,
        IntPtr lpBuffer, Int32 cbBuffer, Int32 dwFlags,
        Int32 dummy1, Int32 dummy2);
</pre>

Jetzt k&ouml;nnen wir einen Stream erstellen...

<pre>
//Neuen Viedo Stream anlegen
private void CreateStream() {
        //Eigenschaften des Streams festlegen
        AVISTREAMINFO strhdr = new AVISTREAMINFO();
        strhdr.fccType = this.fccType; //mmioStringToFOURCC("vids", 0)
        strhdr.fccHandler = this.fccHandler; //"Microsoft Video 1"
        strhdr.dwScale = 1;
        strhdr.dwRate = frameRate;
        strhdr.dwSuggestedBufferSize = (UInt32)(height * stride);
        //H&ouml;hste Qualit&auml;t verwenden! Kompression zerst&ouml;rt die versteckte Nachricht.
        strhdr.dwQuality = 10000;
        strhdr.rcFrame.bottom = (UInt32)height;
        strhdr.rcFrame.right = (UInt32)width;
        strhdr.szName = new UInt16[64];

        //Den Stream erstellen
        int result = AVIFileCreateStream(aviFile, out aviStream, ref strhdr);

        //Format festlegen
        BITMAPINFOHEADER bi = new BITMAPINFOHEADER();
        bi.biSize      = (UInt32)Marshal.SizeOf(bi);
        bi.biWidth     = (Int32)width;
        bi.biHeight    = (Int32)height;
        bi.biPlanes    = 1;
        bi.biBitCount  = 24;
        bi.biSizeImage = (UInt32)(this.stride * this.height);

        //Format zuweisen
        result = Avi.AVIStreamSetFormat(aviStream, 0, ref bi, Marshal.SizeOf(bi));
}
</pre>

...und Video Frames schreiben.

<pre>
//Leere AVI Datei anlegen
public void Open(string fileName, UInt32 frameRate) {
        this.frameRate = frameRate;

        Avi.AVIFileInit();

        int hr = Avi.AVIFileOpen(
                ref aviFile, fileName,
                OF_WRITE | OF_CREATE, 0);
}

//Ein Bild zum Stream hinzuf&uuml;gen - wenn erstes Bild: Stream erstellen
public void AddFrame(Bitmap bmp) {
        BitmapData bmpDat = bmp.LockBits(
                new Rectangle(0, 0, bmp.Width, bmp.Height),
                ImageLockMode.ReadOnly,PixelFormat.Format24bppRgb);

        //Es ist der erste Frame - Gr&ouml;&szlig;e festlegen und Stream erstellen
        if (this.countFrames == 0) {
                this.stride = (UInt32)bmpDat.Stride;
                this.width = bmp.Width;
                this.height = bmp.Height;
                CreateStream(); //a method to create a new video stream
        }

        //Bitmap hinzuf&uuml;gen
        int result = AVIStreamWrite(aviStream,
                countFrames, 1,
                bmpDat.Scan0, //pointer to the beginning of the image data
                (Int32) (stride  * height),
                0, 0, 0);

        bmp.UnlockBits(bmpDat);
        this.countFrames ++;
}
</pre>

Mehr brauchen wir nicht, um Video Stream zu lesen und zu schreiben. Non-Video Streams und Kompression
sind erstmal uninteressant, weil Kompression die versteckte Nachricht durch Farbver&auml;nderungen
zerst&ouml;ren w&uuml;rde, und Sound die Dateien noch gr&ouml;sser machen w&uuml;rde - unkomprimierte
AVI Dateien sind gross genug ;-)

## &Auml;nderungen in <code>CryptUtility</code>

Die Methode <code>HideOrExtract()</code> hat bisher alle Tr&auml;ger-Bilder auf einmal geladen.
Das war von Anfang an nicht gerade perfekt, und ist ab jetzt unbrauchbar, da alle AVIs entpackt werden m&uuml;ssten.
Ab jetzt l&auml;dt <code>HideOrExtract()</code> nur eine Bitmap, und schlie&szlig;t sie wieder bevor das n&auml;chste
Bild geladen wird. Das aktuell verwendete Tr&auml;ger-Bild - einfache Bitmap oder extrahierter AVI Frame -
wird in einer <code>BitmapInfo</code> Struktur gespeichert.

<pre>
public struct BitmapInfo{
        //Unkomprimiertes Bild
        public Bitmap bitmap;

        //Position des Frames im AVI Stream, -1 f&uuml;r Non-AVI Bitmaps
        public int aviPosition;
        //Anzahl der Frames im AVI Stream,  0 f&uuml;r Non-AVI Bitmaps
        public int aviCountFrames;

        //Pfad und Name der Bitmap-Datei
        //Diese Datei wird sp&auml;ter gel&ouml;scht, wenn aviPosition 0 oder gr&ouml;&szlig;er ist
        public String sourceFileName;
        //Anzahl der Bytes, die in diesem Bild versteckt werden
        public long messageBytesToHide;

        public void LoadBitmap(String fileName){
                bitmap = new Bitmap(fileName);
                sourceFileName = fileName;
        }
}
</pre>

Jedesmal wenn <code>MovePixelPosition</code> die Position ins n&auml;chste Tr&auml;ger-Bitmap versetzt,
pr&uuml;ft die Methode das Feld <code>aviPosition</code>. Wenn <code>aviPosition</code> &lt; 0 ist, wird
die Bitmap gespeichert und geschlossen. Wenn <code>aviPosition</code> 0 oder gr&ouml;&szlig;er, ist es ein
Frame in einem AVI Stream. Dieser wird nicht in einer Datei gespeichert, sondern zum ge&ouml;ffneten AVI Stream hinzugef&uuml;gt.
Falls die Bitmap ein einfaches Bild oder der letzte Frame eines AVI Streams war, &ouml;ffnet die Methode
das n&auml;chste Tr&auml;ger-Bild. Gibt es weitere Frames im AVI Stream, wird die n&auml;chste Bitmap
exportiert und ge&ouml;ffnet.


## Hinweise

Das Project enth&auml;lt drei neue Klassen:
- <code>AviReader</code> &ouml;ffnet AVI Dateien und kopiert Frames in Bitmap Dateien.
- <code>AviWriter</code> erstellt neue AVI Dateien und kombiniert Bitmaps zu einem Video Stream.
- <code>Avi</code> enth&auml;lt alle Deklarationen und Struktur-Definitionen.


## Danke


Danke an <a href="http://www.shrinkwrapvb.com" target="_blank">Shrinkwrap Visual Basic</a>,
        f&uuml;r das &quot;Visual Basic AVIFile Tutorial&quot;, speziell f&uuml;r das Beispiel
        zum kopieren von DIBs.


Danke an Rene N., der eine <code>AviWriter</code>-Klasse in
        <a href="news://microsoft.public.dotnet.languages.csharp">microsoft.public.dotnet.languages.csharp</a> gepostet hat.


