---
title: Analoge Musik-Cassetten
category: Steganografie
subcategory: Audiodaten
order: 9
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 35.5 Kb]({% link /assets/steganodotnet15_src.zip %})

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode und SoX Binaries - 514 Kb]({% link /assets/steganodotnet15_src_and_sox.zip %})

## Worum geht es?

<img src="/images/stations.jpg" width=600>

Digitale information ist verlustfrei. Man kann eine Datei von Festlpatte auf Floppy auf CD auf Flash Stick auf ... kopieren
und es wird nach wie vor die gleiche Datei sein.
Aber wenn man analoge Medien verwendet, wie gutes altes Tape, kann man sich auf das Gegenteil verlassen:
Die vom Medium gelesenen Daten werden definitiv anders sein, als das, was du vorher darauf geschrieben hast.


Dennoch gibt es viele Arten, geheime Information in Klang auf einer Audio-Cassette zu verstecken.
Eine recht einfache davon m&ouml;chte ich dir erkl&auml;ren. Alles was du tun musst, bevor du es ausprobieren kannst,
ist, einen alten Cassetten-Rekorder zu holen, den Mikrofon-Anschluss mit dem Kopfh&ouml;rer-Anschluss (oder Line Out)
deines Computers, und den Mikrofon-Anschluss (oder Line In) des Computers mit dem Kopfh&ouml;rer-Anschluss des
Rekorders zu verbinden.


Dieser Artikel verwendet Code aus <A href="http://www.codeproject.com/cs/media/cswavrec.asp" target="_blank">A full-duplex audio player in C# using the waveIn/waveOut APIs</A>,
und erfordert <A href="http://sox.sourceforge.net" target="_blank">Sound Exchange (SoX)</A>.

## Die Idee

Nachdem wir einen Klang auf eine Audio Cassette gespielt, und wieder in eine Wave-Datei zur&uuml;ck-aufgezeichnet haben, k&ouml;nnen
wir uns nicht darauf verlassen, dass die bin&auml;ren Daten dieselben oder auch nur &auml;hnlich sind.
Darum reicht es nicht mehr aus, nur ein paar Bits zu &auml;ndern.
Wir m&uuml;ssen den Klang so ver&auml;ndern, dass wir es wiedererkennen k&ouml;nnen, sogar hinter massenweise Rauschen und zuf&auml;lligen Ver&auml;nderungen.


Deine erste Idee k&ouml;nnte sein, sehr kurze Piepser alle <I>n</I> Sekunden einzuf&uuml;gen, wobei <I>n</I> ein Byte der geheimen Nachricht ist.
Der Empf&auml;nger kann die T&ouml;ne mit einem Band-Pass-Filter isolieren, die Intervalle in einen Stream schreiben,
und aus diesem einfach die Nachricht lesen. Aber wer das versucht, dem wird schnell das Band ausgehen:


<PRE>A = 65
B = 66
C = 67
ABC = 65 + 66 + 67 = 198</PRE>


Wenn wir eine wiedererkennbare Frequenz in Intervallen, die f&uuml;r unsere Bytes stehen, einf&uuml;gen w&uuml;rden,
br&auml;uchten wir 198 Sekunden allein f&uuml;r eine Kurznachricht wie <I>ABC</I>.


Deine zweite Idee k&ouml;nnte sein, jedes geheime Byte in oberews und unteres Halbbyte zu teilen, so dass das maximale Intervall
zwischen zwei Piepsern 15 betr&auml;gt, also kein Byte mehr als 30 Sekunden auf der Cassette belegt.
Das Zeichen "z", zum Beispiel, der mit dem ersten Versuch 122 Sekunden verbrauchen w&uuml;rde,
braucht so nur noch 17 Sekunden:


<PRE lang=text>z = <B>122</B> = 1111010
half bytes = <B>7</B> (0111) and <B>10</B> (1010)</PRE>
<IMG height=80 src="/images/seconds.png" width=600>


Das ist genau das, was diese Anwendung macht. Sie erm&ouml;glicht es, die Tr&auml;ger-Wave nach einer unbenutzten oder relativ
leisen Frequenz abzusuchen, und f&uuml;gt extrem kurze, kaum h&ouml;rbare T&ouml;ne in genau dieser Frequenz ein.
Man kann das Ergebnis abspielen, und mit dem Cassetten-Rekorder aufzeichnen.
Um die so versteckte Nachricht auszulesen, spielt man einfach die Cassette ab, nimmt die Musik mit einem
Audio Recorder-Programm auf (z.B. GoldWave etc.), und entfernt die Stille von Anfang und Ende.
Danach kann man die Wave-Datei mit dieser Anwendung &ouml;ffnen, die vorher zum Verstecken verwendete Frequenz eingeben,
und zuschauen, w&auml;hrend der Band-Pass-Filter die Piepser isoliert und die Nachricht rekonstruiert.

## Wie es funktioniert

<H4>Verstecken einer Nachricht</H4>

Zum Verstecken einer Nachricht geh&ouml;ren f&uuml;nf Schritte:

* Eine Wave-Datei ausw&auml;hlen, und die Nachricht eingeben.
* Pr&uuml;fen, ob die Nachricht in den Sound hinein passt, notfalls k&uuml;rzen.
* Eine Frequenz raten, die nur in geringer Lautst&auml;rke oder gar nicht vorkommt.<br/>
         Falls "Check sound" eine Warnung ausgibt, die Schwellenlautst&auml;rke oder Frequenz erh&ouml;hen, bis die Warnung verschwindet.
* Den neuen Sound schreiben.
* Den neuen Sound abspielen und aufs Band aufzeichnen.


<img src="/images/screenHide.png" width=600>


Schritt drei und vier erkl&auml;ren sich nicht unbedingt von selbst.
Fangen wir mit Schritt drei an: Eine Frequenz auf Existenz und Lautst&auml;rke pr&uuml;fen.
Der Benutzer r&auml;t eine Frequenz, und klickt auf "Check sound".
Um zu &uuml;berpr&uuml;fen, ob die h&ouml;chste Amplitude einer solchen Frequenz niedriger als der ausgew&auml;hlte Wert ist
(was bedeutet, dass wir die gew&auml;hlte Kombination verwenden k&ouml;nnen),
m&uuml;ssen wir zuerst die Frequenz mit einem Band-Pass-Filter isolieren. SoX erledigt das f&uuml;r uns.
Anschlie&szlig;end vergleichen wir die Samples des Ergebnisses mit der ausgew&auml;hlten maximalen Amplitude
(nennen wir es doch Lautst&auml;rke, in diesem Fall ist es fast das gleiche), und z&auml;hlen die Samples, die zu laut sind.

<PRE>//filter for frequency

String outFileName = waveUtility.FindFrequency(
       Path.GetTempPath(),
       soxPath, //contains a path like C:\somewhere\sox.exe
       (int)numFrequency.Value);

//Let another utility read the result file

WaveUtility filterUtility = new WaveUtility(outFileName);
WaveSound waveSound = filterUtility.WaveSound;

//filter for volume, check what is left of the sound

long countLoudSamples = 0;
short minVolumne = (short)(numHideVolume.Value - 100);
short[] samples = waveSound.Samples;
for (int n = 0; n &lt; samples.Length; n++) {
    if (Math.Abs(samples[n]) &gt; minVolumne) {
       countLoudSamples++;
    }
}

if (countLoudSamples &gt; 0) {
   MessageBox.Show(String.Format("The Frequency might be" +
     " a bad choice, because there are already {0} " +
     "too loud samples in the sound.", countLoudSamples));
   errorProvider.SetError(numHideFrequency,
     "Frequency not fitting, oder selected volume too low.");
} else {
   errorProvider.SetError(numHideFrequency, String.Empty);
}</PRE>


Fallses dich interessiert, wie <CODE>waveUtility.FindFrequency</CODE> arbeitet:
Es kombiniert die Parameter zu einem String, ruft damit SoX auf, und liest die Fehlerausgabe (falls es &Auml;rger gibt).

<PRE >/// &lt;summary&gt;Let "Sound Exchange" perform a band pass filter on the sound&lt;/summary&gt;
/// &lt;param name="tempDirectory"&gt;Path of the directory for temporary files&lt;/param&gt;
/// &lt;param name="soxPath"&gt;Path and Name of sox.exe&lt;/param&gt;
/// &lt;param name="frequency"&gt;Frequency that may pass the filter&lt;/param&gt;
/// &lt;returns&gt;Path of the output file&lt;/returns&gt;
public String FindFrequency(String tempDirectory, String soxPath, int frequency)
{
    String inFileName = Path.Combine(tempDirectory, "in.wav");
    String outFileName = Path.Combine(tempDirectory, "out.wav");
    int fileLength = this.WaveSound.SaveToFile(inFileName);

    String soxArguments = String.Format(
              "-t wav \"{0}\" -t .wav -c 1 -s -w \"{1}\" band {2} 10",
              inFileName,
              outFileName,
              frequency);

    RunSox(soxPath, soxArguments);
    return outFileName;
}

/// &lt;summary&gt;Let "Sound Exchange" convert the sound&lt;/summary&gt;
/// &lt;param name="soxPath"&gt;Path and Name of sox.exe&lt;/param&gt;
/// &lt;param name="soxArguments"&gt;Arguments for sox.exe&lt;/param&gt;
private void RunSox(String soxPath, String soxArguments)
{
    ProcessStartInfo startInfo = new ProcessStartInfo(
                soxPath,
                soxArguments);
    startInfo.RedirectStandardError = true;
    startInfo.UseShellExecute = false;
    Process sox = Process.Start(startInfo);

    StreamReader errorReader = sox.StandardError;
    String errors = errorReader.ReadLine();
    if (errors != null) {
        throw new ApplicationException("sox failed: " + errors);
    }

    sox.WaitForExit(10000);
}</PRE>


Haben wir erst einmal eine Frequenz un Amplitude gefunden, die im Tr&auml;ger-Klang eindeutig sein wird,
so dass sie nicht beim Extrahieren mit unschuldigen Samples verwechselt werden kann,
k&ouml;nnen wir den geheimen Stream verstecken.

<PRE >/// &lt;summary&gt;Hide a message in the wave&lt;/summary&gt;
/// &lt;param name="message"&gt;Stream containing the message&lt;/param&gt;
/// &lt;param name="frequencyHz"&gt;Frequency of the beeps,
///        which will be inserted into the sound&lt;/param&gt;
/// &lt;param name="volumne"&gt;Maximum sample value of the beeps&lt;/param&gt;
public void Hide(Stream message, int frequencyHz, int volumne)
{
    Stream preparedMessage = PrepareMessage(message);
    int messageByte;
    int offset = 0;
    while ((messageByte = preparedMessage.ReadByte()) &gt; -1) {
        offset += messageByte;
        InsertBeep(offset, frequencyHz, volumne);
    }
}</PRE>


Dieses kurze Code-St&uuml;ck ruft zwei wichtige Methoden auf, und zwar <CODE>PrepareMessage</CODE> und <CODE>InsertBeep</CODE>.
<CODE>PrepareMessage</CODE> nimmt einen Stream mit der geheimen Nachricht an, und spaltet jedes Byte in zwei Halbbytes.
Das Intervall zwischen zwei <I>Beeps</I> darf nicht Null Sekunden lang sein, aber viele Halbbytes werden Null sein,
darum wird jeder Wert im vorbereiteten Stream <CODE>Halbbyte + 1</CODE> betragen.
Sp&auml;ter muss die Lesemethode 1 von jedem Intervall abziehen, so dass wir wieder die alten Halbbytes bekommen.


<PRE>/// &lt;summary&gt;Split the bytes of a message into four-bit-blocks&lt;/summary&gt;
/// &lt;param name="message"&gt;Stream containing the message&lt;/param&gt;
/// &lt;returns&gt;Stream containing the same
///        message with two bytes per original byte&lt;/returns&gt;
private Stream PrepareMessage(Stream message)
{
    message.Position = 0;

    MemoryStream preparedMessage = new MemoryStream();
    int messageByte;
    int highHalfByte;
    int lowHalfByte;

    while ((messageByte = message.ReadByte()) &gt; -1) {
        //split into high and low part
        highHalfByte = (messageByte &gt;&gt; 4);
        lowHalfByte = (messageByte - (highHalfByte &lt;&lt; 4));

        //intervals of 0 seconds are not possible -&gt; add 1 to all intervals
        preparedMessage.WriteByte((byte)(highHalfByte + 1));
        preparedMessage.WriteByte((byte)(lowHalfByte + 1));
    }

    preparedMessage.Position = 0;
    return preparedMessage;
}</PRE>


Worauf warten wir noch? Wir haben den Sound, den vorbereitete Nachrichten-Stream, und eine benutzbare Frequenz.
Das ist alles, was wir brauchen, um kleine Ger&auml;usche an speziellen Sekunden einzuf&uuml;gen, was die Aufgabe von
<CODE>CreateBeep</CODE> und <CODE>InsertBeep</CODE> ist.

<PRE>/// &lt;summary&gt;Creates sine sound of a specific frequency&lt;/summary&gt;
/// &lt;param name="frequencyHz"&gt;Frequency in Hertz&lt;/param&gt;
/// &lt;param name="volumne"&gt;Amplitude&lt;/param&gt;
private WaveSound CreateBeep(int frequencyHz, int volumne)
{
    // samples for 1/32 seconds
    short[] samples = new short[waveSound.Format.SamplesPerSec / 32];
    double xValue = 0;
    short yValue;

    double xStep = (2 * Math.PI) / waveSound.Format.SamplesPerSec; // 1 Hz
    xStep = xStep * frequencyHz;

    for (int n = 0; n &lt; samples.Length; n++) {
        xValue += xStep;
        yValue = (short)(Math.Sin(xValue) * volumne);
        samples[n] = yValue;
    }

    WaveSound beep = new WaveSound(waveSound.Format, samples);
    return beep;
}

/// &lt;summary&gt;Replaces a part of the sound with a beep&lt;/summary&gt;
/// &lt;param name="insertAtSecond"&gt;Where to put the beep&lt;/param&gt;
/// &lt;param name="frequencyHz"&gt;Frequency of the beep in Hertz&lt;/param&gt;
/// &lt;param name="volumne"&gt;Maximum sample value of the beep&lt;/param&gt;
public void InsertBeep(float insertAtSecond, int frequencyHz, int volumne)
{
    short[] beepWave = CreateBeep(frequencyHz, volumne).Samples;
    int insertAtSample = (int)(waveSound.Format.SamplesPerSec * insertAtSecond);
    int longWaveIndex = insertAtSample;
    for (int index = 0; index &lt; beepWave.Length; index++) {
        waveSound[longWaveIndex] = beepWave[index];
        longWaveIndex++;
    }
}</PRE>


Wenn <CODE>Hide</CODE> die <CODE >while</CODE>-Schleife verl&auml;sst, wurde der gesamte Nachrichten-Stream
in den Sound gepiept. Vorausgesetzt, man hat eine passende Frequenz und nicht zu hohe Amplitude eingestellt,
ist die Ver&auml;nderung kaum h&ouml;rbar (andernfalls kann sie dem menschlichen Ohr zumindest wie gew&ouml;hnliche St&ouml;rungen vorkommen).
Mann kann das Ergebnis in eine <I>.wav</I>-Datei speichern, oder abspielen und ungespeichert aufnehmen.


<H4>Auslesen einer Nachricht</H4>


Vor dem Lesen einer versteckten Nachricht muss der Benutzer den Sound filtern.
Falls der Cassetten-Spieler sehr schlimme St&ouml;rungen in ausgerechnet unserer Frequenz hinzugef&uuml;gt hat,
so dass falsche Piepser erkannt werden, k&ouml;nnen diese Fehler aussortiert werden.


<OL>
* Frequenz der erwarteten T&ouml;ne eingeben, und den Klang Band-Pass filtern.
             Der zweite Filter-Button - threshold volume - ist nicht wirklich n&ouml;tig.
             Man kann damit Sampels aus der Grafik entfernen, die ohnehin als Stille behandelt w&uuml;rden.
* Die T&ouml;ne finden. Ein Piepser ist eine Gruppe von Samples, die gr&ouml;&szlig;er sind als die gew&auml;hlte Schwellenlautst&auml;rke.
        In der Grafik werden Anfang und Ende jedes erkannten Piepsers mit roten Linien markiert.
        Die CheckBoxes erm&ouml;glichen es, einzelne Piepser aus der Auswertugn auszuschlie&szlig;en,
        wenn man sicher ist, dass sie nicht zur Nachricht geh&ouml;ren.
* Die Nachricht lesen. Der letzte Schritt listet die Abst&auml;nde zwischen den ausgew&auml;hlten Ger&auml;suchen auf,
        und rekonstruiert daraus die versteckte Nachricht.</LI>
</OL>

<IMG height=444 src="/images/screenFind.png" width=600>


"Filter sound" wendet den gleichen Band-Pass-Filter an, den wir schon kennen.
Diesmal z&auml;hlen wir aber keine hohen Samples, sondern zeigen die gefilterte Wave an.

<PRE>String outFileName = waveUtility.FindFrequency(
        Path.GetTempPath(),
        soxPath,
        (int)numExtractFrequency.Value);

waveControl.OpenFromFile(outFileName);</PRE>


Jeder Ton ist 1/32 Sekunde lang. Das ist nicht ivel f&uuml;r menschliche Ohren, aber es sind eine Menge Samples.
Die Sample-Gruppen werden durch Stille getrennt (dank Band-Pass-Filter und Schwellenlautst&auml;rke),
daher k&ouml;nnen wir ein Scan-Fenster &uuml;ber die Samples schieben, und ein St&uuml;ck Stille so definieren:
<i>Alle Samples im aktuellen Zeitfenster liegen unterhalb der Schwelle</i>.
Samples &uuml;ber der Schwelle werden als Teil desselben Tons behandelt, wenn keine Stille zwischen ihnen liegt.

<PRE>/// &lt;summary&gt;Find anything but silence in the sound&lt;/summary&gt;
/// &lt;remarks&gt;Raises the BeepFound event everytime a sound
///      is detected between two blocks of silence&lt;/remarks&gt;
/// &lt;param name="tolerance"&gt;
/// Sample values greater than [tolerance] are sound,
/// sample values less than [tolerance] are silence
/// &lt;/param&gt;
public void FindAnything(short tolerance) {
    //size of scan window in samples
    int scanWindowSize = waveSound.Format.SamplesPerSec / beepLength;
    //size of scan window in seconds
    float scanWindowSizeSeconds = (float)scanWindowSize /
                (float)waveSound.Format.SamplesPerSec;

    int startIndex = -1;
    int endIndex = -1;
    int countSilentSamples = 0;
    for (int n = 0; n &lt; waveSound.Count; n++) {
        if (Math.Abs(WaveSound[n]) &gt; tolerance) { //found a sound
            countSilentSamples = 0;
            if(startIndex &lt; 0){
                startIndex = n;
            }
        } else if (startIndex &gt; -1) { //searched and found silence
            countSilentSamples++;
            if (countSilentSamples == scanWindowSize) {
                endIndex = n - scanWindowSize;

                //tell the caller to mark a found beep in the wave
                NotifyOnBeep(startIndex, endIndex, scanWindowSizeSeconds);

                //scan next time window
                countSilentSamples = 0;
                startIndex = -1;
             }
        }
   }

   if (startIndex &gt; -1) { //wave ends with a beep
       NotifyOnBeep(startIndex, waveSound.Count-1, scanWindowSizeSeconds);
   }
}</PRE>

Wenn der Benutzer schlie&szlig;lich auf "Read message" klickt, wurden alle Informationen bereits extrahiert,
wir m&uuml;ssen nur noch die Halbbytes zusammensetzen.

<PRE >private void btnExtract_Click(object sender, EventArgs e)
{
    //list the beginning seconds of the selected beeps
    Collection&lt;Beep&gt; selectedItems = waveControl.SelectedItems;
    Collection&lt;float&gt; startSeconds = new Collection&lt;float&gt;();
    foreach (Beep beep in selectedItems) {
        startSeconds.Add(beep.StartSecond);
    }

    //read the hidden message from the seconds
    Stream messageStream = waveUtility.Extract(startSeconds);
    StreamReader messageReader = new StreamReader(messageStream, Encoding.Default);
    String message = messageReader.ReadToEnd();
    messageReader.Close();

    txtExtractMessage.Text = message;
}</PRE>

## Klingt nach einem Beispiel-Klang


Falls du gerade keine Wave-Dateien zur Hand hast, um mit der Anwendung zu spielen,
f&uuml;hl dich frei, diese Beispiel-Aufnahmen zu verwenden: <A href="/assets/demoWaves.zip">demoWaves.zip</A>.

* <I>101seconds.wav</I> - 101 Sekunden langer Original-Sound.
* <I>result_f2500_v2000.wav</I> - Der Sound inklusive "! C# RULES !", versteckt mit 2500Hz und einer Amplitude von 2000.
* <I>record_f2500_v2000.wav</I> - Gespielt von dem Casseten-Spieler auf dem Foto, aufgezeichnet von meinem Notebook, via Kopfh&ouml;rer/Mikrofon-Anschl&uuml;sse.
       Versuch es ruhig, "! C# RULES !" kann immer noch problmlos extrahiert werden.

