---
title: SteganoDotNet
category: Steganografie
subcategory: Sonstiges
order: 16
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [Quellen, nur Framework - 110 Kb]({% link /assets/SteganoDotNet_0.1alpha_src.tar.gz %})

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [Quellen, Framework und Demo-Anwendung - 1.5 Mb]({% link /assets/SteganoDotNet_0.1alpha_with_S4U_src.tar.gz %})

## Ein Framework für Steganographie in .NET/mono

### Unterstützte Trägerformate

* Pixelgrafiken 24 Bit (verlustfreie Formate)
* Pixelgrafiken 8 Bit (GIF, PNG)
* Stichwort-Listen
* HTML-Seiten
* MIDI-Sequenzen
* .NET Assemblies
* Wave-Klänge (unkomprimiert)
* Audio-Daten jeder Art (analog-sicher)

### Extras

* Verteilen des Klartextes über mehrere Träger verschiedener Formate
* Sichern mit einem beliebig langen Schlüssel

### Was kann es?
<i>SteganoDotNet</i> fasst alle hier vorgestellten Techniken plus einige Extras in einem Framework aus zwei Namespaces zusammen:

* <b>SteganoDotNet.Action:</b> Der steganographische Kern des Frameworks. Er enthält die abstrakten Typen FileType und FileUtility und alle davon abgeleiteten Klassen, die jeweils eine Art von Trägermedium beschreiben und die passenden Stego-Methoden kapseln. Läuft sowohl in .NET als auch in mono.
* <b>SteganoDotNet.UserControls:</b> Fertige Oberflächen-Bausteine. Für jedes *FileUtility wird ein Konfigurationsdialog als UserControl bereitgestellt. Wenn Sie die Einstellungen nicht selbst festlegen möchten/können, binden Sie das passende UserControl ein, um sie interaktiv abzufragen.

Darüber hinaus enthält das Projekt die Demo-Anwendung <i>Stego For You</i>, die so gut wie alle Framework-Features unter einer Oberfläche vereint.
### Wie alles begann... ein Framework stellt sich vor
Wie SteganoDotNet entstand, warum einige schöne Features aus <i>Stego For You</i> wieder entfernt wurden und Platz für Diskussionen finden Sie auf <img src="/images/codeproject88x31.gif" alt="CodeProject" /> <i>CodeProject</i>:

* <a href="https://www.codeproject.com/script/Articles/MemberArticles.aspx?amid=475133" target="_blank"> Artikel-Serie mit Forum zu jedem Teilprojekt</a>
* <a href="https://www.codeproject.com/Articles/15029/A-Stegano-Framework-for-NET-Developers" target="_blank"> Artikel über <i>Stego For You</i> in seiner Urform, das Framework und Beispiel-Anwendungen. </a>

