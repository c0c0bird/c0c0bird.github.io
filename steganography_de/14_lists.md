---
title: Musiknoten
category: Steganografie
subcategory: Listen
order: 14
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 38.87 Kb]({% link /assets/steganodotnet20_src.zip %})

## Worum geht es?


Im [letzten Artikel]({% link steganography_de/13_lists.md %}) hast du gelernt, wie man eine Nachricht durch Sortieren einer Liste beliebiger Dinge codieren kann. Musik besteht aus Melodien, die gewissermaßen; Notenlisten sind. Warum nicht mal ein Lied komponieren, dessen Melodie ein Text ist?

<img src="/images/steganodotnet201.png" alt="screen" width="506" height="481" />

Entscheiden müssen wir uns nur für die Menge der Noten, die im Lied vorkommen sollen. Weil es <code>Factorial(count)</code> aufzählbare Permutationen gibt, erlauben mehr verschiedene Noten ein größeres Alphabet für die geheime Nachricht. Dieser Artikel wird sich nicht um Harmonielehre und Komposition kümmern. Für den Anfang werden wir eine ganze chromatische Tonleiter aufmischen, um ein langes oder einige kurze Wörter zu codieren. Für längere Texte kann man mehrere Melodieteile verketten.
## Hallo Welt

Ein einfaches Beispiel: Eine Tonleiter hat 12 Noten, zwei davon machen 24 Noten, das reicht für ein "hello world". Der Nachrichtentext wid als ein einziger <code>BigInteger</code> behandelt. Die leere Nachricht ist '0', die Default-Reihenfolge der Noten:

<img src="/images/steganodotnet202.png" alt="chromatic scale" width="384" height="39" />

Ich habe mifür 42 verschiedene Zeichen entschieden: a-z 0-9 .:() und Leerzeichen. Mit diesem Alphabet wird die numerische Repräsentation von "hello world" folgendermaßen berechnet:
<pre>h =  7		w = 22
e =  4		o = 14
l = 11		r = 17
l = 11		l = 11
o = 14		d =  3
------------------------
   7 * (42^10)	<em>h</em>
+  4 * (42^9)	<em>e</em>
+ 11 * (42^8)	<em>l</em>
+ 11 * (42^7)	<em>l</em>
+ 14 * (42^6)	<em>o</em>

+ 36 * (42^5)	<em>[space]</em>

+ 22 * (42^4)	<em>w</em>
+ 14 * (42^3)	<em>o</em>
+ 17 * (42^2)	<em>r</em>
+ 11 * (42^1)	<em>l</em>
+  3 * (42^0)	<em>d</em>
------------------------
= 121297199112622725
</pre>

Mit dieser Zahl lässt sich der zuvor erklärte Algorithmus füttern, mit der chromatischen Tonleiter als Trägerliste. (Für Nicht-Musiker: <em>C C# D D# E F F# G G# A H B c c# d d# e f f# g g# a h b</em>.)

Das Ergebnis sieht so aus:

<img src="/images/steganodotnet203.png" alt="mixed up notes" width="384" height="39" /><br /> d d# D e f A G# f# F H g F# C# E g# c# B a G D# c C h b
## Weniger Zeichen, mehr Wörter

Die größmögliche numerische Nachricht (und damit die Länge der Textnachricht) ist durch <code>Factorial(countNotes)</code> beschänkt. Um mehr Text zu codieren könnte man entweder mehr Noten verwenden - oder die numerischen Werte kleiner halten. Im obigen Beispiel verursachte die Basis 42 extrem große Werte. Beschränkt man die Zeichen auf a-z und Leerzeichen, so ist die Basis 27 und der Text "hello world" hat den Wert 1474967912327400. Das heißt, wir können einen längeren Text in einer Melodie derselben Länge speichern.
<pre>   7 * (27^10)	<em>h</em>		+ 22 * (27^4)	<em>w</em>
+  4 * (27^9)	<em>e</em>		+ 14 * (27^3)	<em>o</em>
+ 11 * (27^8)	<em>l</em>		+ 17 * (27^2)	<em>r</em>
+ 11 * (27^7)	<em>l</em>		+ 11 * (27^1)	<em>l</em>
+ 14 * (27^6)	<em>o</em>		+  3 * (27^0)	<em>d</em>
+ 36 * (27^5)	<em>[space]</em>
---------------------------------------------------
= 1474967912327400
</pre>
## Rhythmus - Warum nicht?

Eine Melodie aus reinen Viertelnoten klingt schnell langweilig. Aber wenn die Noten einmal sortiert sind, kann man überall Duplikate einfügen. Alle Wiederholungen werden vor dem Decodieren entfernt. Da Zeit keine Rolle spielt (im Prototypen noch nicht), dürfen Vierten nach Belieben in Achtel aufgespalten werden. Nichts davon beeinflusst die codierte Nachricht. Hier sind ein paar Varianten von "coding in c sharp" (Nummer 201255931456816565832252 mit dem 27-Zeichen-Alphabet).

<img src="/images/steganodotnet204.png" width="384" height="80" /><br /> Text: "coding in c sharp"<br /> Tonleiter: beginnt mit A<br /> Melodie: f e c# b g a G h F# H g# G# A D c f# E F C d C# B d# D#

<img src="/images/steganodotnet205.png" width="384" height="80" /><br /> Text: "coding in c sharp"<br /> Tonleiter: beginnt mit D<br /> Melodie: H A F# e c d C d# b D# c# C# D g F B a h f G f# E G# g#

<img src="/images/steganodotnet206.png" width="384" height="80" /><br /> Text: "coding in c sharp"<br /> Tonleiter: beginnt mit F<br /> Melodie: c# c A g d# f D# f# D F# e E F h G# d C C# g# H a G B b

<img src="/images/steganodotnet207.png" width="384" height="80" /><br /> Text: "coding in c sharp"<br /> Tonleiter: beginnt mit G<br /> Melodie: dis d B a f g F gis E Gis fis Fis G C H e D Dis h c b A cis Cis
## Die Demo-Anwendung

Die Demo codiert in vier Schritten Text als Musik:
<ol>
<li>Wähle ein Offset für die chromatische Tonleiter.</li>
<li>Wähle ein Alphabet. Die Zeichen des aktuellen Alphabets stehen jeweils in der Titelleiste.</li>
<li>Gib deine Nachricht ein.</li>
<li>"Mix" deine Melodie. Falls das Ergebnis nicht schön genug ist, probier "Random melody" aus, bis es gut klingt.</li>
</ol>

Natuürlich kann man auch eine Melodie zurück in Text decodieren:
<ol>
<li>Wähle das Offset das beim Codieren benutzt wurde.</li>
<li>Wähle das gleiche Alphabet.</li>
<li>Gib die Noten ins Ergebnisfeld ein.</li>
<li>"Unmix" die Noten zum Klartext.</li>
</ol>

Das Programm wurde für eine Vorführung geschrieben. Daher hat es ein paar versteckte Funktionen, welche die Zuschauer nicht mit sichtbaren Buttons oder Menüs verwirren.
### Zahlenspiele

Manche Geeks fragten mich, was die Textbedeutung von <code>pi</code> sei. Für solche Situationen können Text- und Zahlenfeld vertauscht werden: <strong>Doppelklicke auf das Feld "Content numeric".</strong> Es wird dann entsperrt und die eingegebenen Zahlen werden zu Text decodiert.
### Schnellnotizen

Um den Inhalt eines Textfelds zu speichern, <strong>setze den Fokus in das Feld und drücke Ctrl+Y</strong>. Der Text wird dann an die Datei <em>clipboard.txt</em> im exe-Verzeichnis angehängt.
### Das Notenbild speichern

Die Anwendung zeigt das Ergebnis in Notenname und malt gleichzeitig die Noten. Weil aber nur die Notennamen später wieder eingetippt und decodiert werden können, können nur diese in die Zwischenablage bzw. <em>clipboard.txt</em> kopiert werden. Um die Noten zu speichern hilft ein <strong>Linksklick auf das Bild.</strong> Es wird dann als PNG-Datei im exe-Verzeichnis abgelegt.
### Den Klang speichern

Die Melodie, wie sie vom "Play"-Button gepiept wird, kann mit einem <strong>Rechtsklick auf das Bild</strong> gespeichert werden. Sie wird dann als WAV-Datei im exe-Verzeichnis abgelegt.
### Workaround für mono/Linux

Mit einigen Kombinationen von mono-Version und Audio-System ist <code>Console.Beep</code> unbrauchbar. Wenn bei dir so ein Problem auftritt, <strong>definiere die Präprozessor-Variable <code>IsMono</code> in Form1.cs</strong>.
<pre>#define IsMono</pre>

Anstatt zu piepen, speichert das Programm dann den Klang in einer temporären Datei und lässt ihn von sox abspielen. Es sollte keinen sicht-/hörbaren Unterschied zum normalen Verhalten geben.
## Ausblick

Die Demo-Anwendung zeigt nur die grundlegende Idee. Vielleicht hast du Lust es zu erweitern, um lange Textnachrichten in mehrteilige Melodien aufzuteilen, eine Tonart zu garantieren, nur Harmonien in vorgegebene Melodien einzufügen und so weiter. Im Prinzip gibt es kaum Grenzen, deinen Instrumentalstücken einen geheimen Subtext zu verleihen.

