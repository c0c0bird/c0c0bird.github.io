---
title: Unsichtbare Änderungen
category: Steganografie
subcategory: Bilder
order: 3
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 20.7 Kb]({% link /assets/steganodotnet3_src.zip %})

## Worum geht es?

Die Anwendung aus [Teil 2]({% link steganography_de/2_multicarrier.md %})
f&uuml;gt sichtbares Rauschen in die Tr&auml;ger-Bilder ein. In diesem Artikel
werden die Bytes aufgeteilt und jedes Bit in einem anderen Pixel versteckt.
Die versteckte Nachricht ist endlich unsichtbar, da nur das niedrigste Bit einer Farb-Komponente
ge&auml;ndert wird.

## Ende des Regenbogens

Ein ganzes Byte einer RGB-Farbe zu ersetzen verursacht ein Regenbogen-Muster.
So sieht ein weisses Bild (alle Pixel einfach nur weiss) aus, nachdem ein Text darin versteckt wurde:

<img src="/images/steganodotnet31.png" width="178" height="54" alt="typisches farbiges Rauschen" border="0">

Auf Farbfotos sind diese <i>Regenbogen-Pixel</i> vielleicht akzeptabel.
Als monochromes Rauschen sieht die gleiche Nachricht im gleichen Bild so aus:

<img src="/images/steganodotnet32.png" width="178" height="54" alt="typisches graues Rauschen" border="0">

Wie man sieht wurden zu viele Bits der Pixel ge&auml;ndert.
Jetzt werden wird jedes Byte &uuml;ber acht Pixel verteilen, so dass der <i>Regenbogen</i> verschwindet.

## Bits lesen und setzen

<img src="/images/steganodotnet33.png" width="447" height="56" alt="Ein Byte &uuml;ber acht Pixel verteilen" border="0">

F&uuml;r jedes Byte der Nachricht m&uuml;ssen wir

* Ein Pixel holen
* Das erste Bit des Nachrichten-Bytes holen
* Eine Farb-Komponente des Pixels holen
* Das erste Bit der Farb-Komponente holen
* Wenn sich das Farb-Bit vom Nachrichten-Bit unterscheidet, es setzen/zur&uuml;cksetzen
* Das gleiche mit den anderen sieben Bits tun

Die C#-Funktionen zum Lesen und Setzen einzelner Bits sind einfach:

<pre>
private static bool GetBit(byte b, byte position){
        return ((b & (byte)(1 &lt;&lt; position)) != 0);
}

private static byte SetBit(byte b, byte position, bool newBitValue){
        byte mask = (byte)(1 &lt;&lt; position);
        if(newBitValue){
                return (byte)(b | mask);
        }else{
                return (byte)(b &amp; ~mask);
        }
}
</pre>

Der Rest des Codes hat sich kaum ge&auml;ndert. Verstecken funktioniert jetzt so...
<img src="/images/steganodotnet34.gif" width="361" height="204" alt="Nachricht verstecken" border="0">

...und Extrahieren so:
<img src="/images/steganodotnet35.gif" width="368" height="204" alt="Nachricht auslesen" border="0">

Das ist genug Text f&uuml;r diese Seite.
Wenn Du mehr Details wissen m&ouml;chtest, solltest Du den <a href="/assets/steganodotnet3_src.zip">Quellcode herunterladen</a>.

