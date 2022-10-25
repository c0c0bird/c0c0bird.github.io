---
title: Listen ohne Mathe
category: Steganografie
subcategory: Listen
order: 12
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 34.9 Kb]({% link /assets/steganodotnet14_src.zip %})

## Worum geht es?


Was haben ein GIF-Bild, eine HTML-Seite, und ein Einkaufszettel gemeinsam?
Sie alle sind oder enthalten Listen, die keine besondere Reihenfolge erfordern.
die Farben in der Palette eines GIF-Bildes k&ouml;nnen jede Reihenoflge haben,
so wie die Attribute in einem HTML-Tag, oder die W&ouml;rter auf deinem Einkaufszettel.


In jeder Liste mit <code>anzahl</code> Elementen, k&ouml;nnen wir <code>anzahl-1</code> Bits verstecken,
nur durch das Sortieren der Eintr&auml;ge. Keine Tricks, keine komplizierten Formeln.
Fangen wir an mit einer einfache Liste von W&ouml;rtern,
dann setzen wir das Prinzip fort mit GIF-Paletten und danach mit HTML-Seiten.
The Algorithmus ist immer der gleiche, weil alle Listen im Grunde gleich sind.


Der erste Schritt ist, die Listenelemente in eine bestimmte Reihenfolge zu zwingen.
Diese kann alphabetisch sein, oder eine eigene Sortierung.
Dann wird die sortierte Liste neu sortiert, abh&auml;ngig von den Bits der geheimen Nachricht.


<img src="/images/listHide.png" width="552" height="446" style="background:#ffffff;">


Wenn du die Standard-Sortierung kennst, kannst du die Liste noch einmal sortieren, und die
Stegano-Liste mit der sortierten Liste vergleichen.
Die Unterschiede sagen dir alles &uuml;ber die Nachrichten-Bits, von denen die Reihenfolge stammte:


<img src="/images/listExtract.png" width="497" height="298" border="0" style="background:#ffffff;">


## Zeile f&uuml;r Zeile


Man braucht nicht unbedingt einen Computer f&uuml;r Listen-Steganografie.
Such dir ein St&uuml;ck Papier und einen Stift, und schreib neun Tiere auf.
Nein, das ist nicht der Anfang eines Psychotests, es ist ein Tr&auml;gerdokument.
Die W&ouml;rter brauchen eine Standard-Sortierung, auf die wir uns bei der
sp&auml;teren Neu-Sortierung beziehen k&ouml;nnen, darum sortieren wir sie alphabetisch:


<div style="background:#ffffcc">
<ol>
    <li>Vogel</li>
    <li>Katze</li>
    <li>Dinosaurier</li>
    <li>Hund</li>
    <li>Fisch</li>
    <li>Horse</li>
    <li>Kaninchen</li>
    <li>Schaf</li>
    <li>Einhorn</li>
</ol>
</div>


Ohne mathematische Tricks k&ouml;nnen neun Listeneintr&auml;ge 9-1 = 8 Bits an geheimer Information verstecken,
zum Beispiel das ASCII-Zeichen 'c' (99 or 01100011).
Jeder Eintrag (au&szlig;er dem Letzten) repr&auml;sentiert ein Bit.
F&uuml;r jedes <i>1</i>-Bit verschieben wir das ersten Element aus der Original-Liste in die neue Liste,
f&uuml;r <i>0</i>-Bits verschieben wir einen anderen Eitnrag in die neue Liste.
Angefangen mit dem h&ouml;chsten Bit, l&auml;uft der Prozess folgenderma&szlig;en ab:


<pre>
Original-Liste        Neue Liste

Vogel                 ---
Katze                 ---
Dinosaurier           ---
Hund                  ---
Fisch                 ---
Horse                 ---
Kaninchen             ---
Schaf                 ---
Einhorn               ---

'0' Verstecken - Irgendein Element au&szlig;er dem Ersten verschieben

Vogel                 Dinosaurier
Katze                 ---
---                   ---
Hund                  ---
Fisch                 ---
Horse                 ---
Kaninchen             ---
Schaf                 ---
Einhorn               ---


'1' Verstecken - Erstes Element verschieben

---                   Dinosaurier
Katze                 Vogel
---                   ---
Hund                  ---
Fisch                 ---
Horse                 ---
Kaninchen             ---
Schaf                 ---
Einhorn               ---

'1' Verstecken - Erstes Element verschieben

---                  Dinosaurier
---                  Vogel
---                  Katze
Hund                 ---
Fisch                ---
Horse                ---
Kaninchen            ---
Schaf                ---
Einhorn              ---

'0' Verstecken - Irgendein Element au&szlig;er dem Ersten verschieben

---                  Dinosaurier
---                  Vogel
---                  Katze
Hund                  Kaninchen
Fisch                 ---
Horse                ---
---                  ---
Schaf                ---
Einhorn              ---

... ... und so weiter ... ...

'1' Verstecken - Erstes Element verschieben

---                  Dinosaurier
---                  Vogel
---                  Katze
---                  Kaninchen
---                  Einhorn
---                  Fisch
---                  Hund
Schaf                Horse
---                  ---

Kein nicht-erster Element f&uuml;r ein '0'-Bit &uuml;brig,
dass bedeutet keine Kapazit&auml;t f&uuml;r weitere Bits.
Letzten Eintrag trotzdem verschieben.

---                  Dinosaurier
---                  Vogel
---                  Katze
---                  Kaninchen
---                  Einhorn
---                  Fisch
---                  Hund
---                  Horse
---                  Schaf
</pre>


Die neue Liste enth&auml;lt die gleichen Eintr&auml;ge wie die alte,
und einen zus&auml;tzlichen Sub-Inhalt, von den nur wir etwas wissen.
Das versteckte Byte kann wieder gelesen werden, wenn wir den gleichen Prozess r&uuml;ckw&auml;rts durchspielen.


<pre>
Sortierte Liste      Tr&auml;gerliste

Vogel                Dinosaurier
Katze                Vogel
Dinosaurier          Katze
Hund                 Kaninchen
Fisch                Einhorn
Horse                Fisch
Kaninchen            Hund
Schaf                Horse
Einhorn              Schaf

'Dinosaurier' ist nicht das erste Element der sortierten Liste, also war das versteckte Bit '0'.
Notier dir das, und streich das Element aus beiden Listen.

Vogel                ---
Katze                Vogel
---                  Katze
Hund                 Kaninchen
Fisch                Einhorn
Horse                Fisch
Kaninchen            Hund
Schaf                Horse
Einhorn              Schaf

'Vogel' aus der Tr&auml;gerliste ist das erste Element in der sortierten Liste =&gt; '01'.

---                  ---
Katze                ---
---                  Katze
Hund                 Kaninchen
Fisch                Einhorn
Horse                Fisch
Kaninchen            Hund
Schaf                Horse
Einhorn              Schaf

'Katze' aus der Tr&auml;gerliste ist das erste Element in der sortierten Liste =&gt; '011'.

---                  ---
---                  ---
---                  ---
Hund                 Kaninchen
Fisch                Einhorn
Horse                Fisch
Kaninchen            Hund
Schaf                Horse
Einhorn              Schaf

'Dinosaurier' ist nicht das erste Element der sortierten Liste =&gt; '0110'.
... ... und so weiter f&uuml;r alle Tiere ... ...
</pre>


Das lateinische Alphabet ist keine gute Standard-Sortierung, denn jeder w&uuml;rde es gleich als Erstes ausprobieren.
Mein Vorschlag: Misch dir Buchstaben durcheinander, und schreib dir ein benutzerdefiniertes Alphabet.
Die Liste wie oben sortieren, k&ouml;nnen wir genausogut mit C#:


<pre>
public StringCollection Hide(String[] lines, Stream message, String alphabet)
{
 //sort the lines according to a custom alphabet
 StringCollection originalList = Utilities.SortLines(lines, alphabetFileName);
 StringCollection resultList = new StringCollection();

 int messageByte = message.ReadByte();
 bool messageBit = false;
 int listElementIndex = 0;
 Random random = new Random();

 //for each byte of the message
 while (messageByte &gt; -1) {
       //for each bit
       for (int bitIndex=0; bitIndex&lt;8; bitIndex++) {

           //decide which line is going to be the next one in the new list

           messageBit = ((messageByte & (1 &lt;&lt; bitIndex)) &gt; 0) ? true : false;

           if (messageBit) {
              //pick the first line from the remaining original list
              listElementIndex = 0;
           }else{
              //pick any line but the first one
              listElementIndex = random.Next(1, originalList.Count);
           }

           //move the line from old list to new list

           resultList.Add(originalList[listElementIndex]);
           originalList.RemoveAt(listElementIndex);
       }

       //repeat this with the next byte of the message
       messageByte = message.ReadByte();
 }

 //copy unused list elements, if there are any
 foreach (String s in originalList) {
         resultList.Add(s);
 }

 return resultList;
}
</pre>


Mit einer Liste und einem Alphabet l&auml;sst sich der Prozess ganz einfach umkehren:


<pre>
public Stream Extract(String[] lines, String alphabet)
{
 //initialize empty writer for the message
 BinaryWriter messageWriter = new BinaryWriter(new MemoryStream());

 StringCollection carrierList = new StringCollection();
 carrierList.AddRange(lines);
 carrierList.RemoveAt(carrierList.Count - 1);

 //sort -&gt; get original list
 StringCollection originalList = Utilities.SortLines(lines, alphabetFileName);
 String[] unchangeableOriginalList = new String[originalList.Count];
 originalList.CopyTo(unchangeableOriginalList, 0);

 int messageBit = 0;
 int messageBitIndex = 0;
 int messageByte = 0;

 foreach (String s in carrierList) {

         //decide which bit the entry's position hides

         if (s == originalList[0]) {
            messageBit = 1;
         }else{
            messageBit = 0;
         }

         //remove the item from the sorted list
         originalList.Remove(s);

         //add the bit to the message
         messageByte += (byte)(messageBit &lt;&lt; messageBitIndex);

         messageBitIndex++;
         if (messageBitIndex &gt; 7) {
            //append the byte to the message
            messageWriter.Write((byte)messageByte);
            messageByte = 0;
            messageBitIndex = 0;
         }
 }

 //return message stream
 messageWriter.Seek(0, SeekOrigin.Begin);
 return messageWriter.BaseStream;
}
</pre>

## Bunte Bits


Jede indizierte Bitmap enth&auml;lt eine Liste, die sich auf die gleiche Weise missbrauchen l&auml;sst.
Diese beiden Paletten geh&ouml;ren zum gleichen GIF-Bild. Eine ist das Original,
die andere enth&auml;lt einen versteckten Text von 31 Zeichen:


<img src="/images/paletteOriginal.png" width="266" height="266">

<img src="/images/paletteNeu.png" width="267" height="266">


Wieder ist das Erste was wir brauchen eine Palette mit einer Standard-Sortierung.
In diesem Beispiel werden wir die Farben nach ihren ARGB-Werten sortieren.


<pre>
public Bitmap Hide(Bitmap image, Stream message)
{
 //list the palette entries an integer values
 int[] colors = new int[image.Palette.Entries.Length];
 for (int n = 0; n &lt; colors.Length; n++) {
     colors[n] = image.Palette.Entries[n].ToArgb();
 }

 //initialize empty list for the resulting palette
 ArrayList resultList = new ArrayList(colors.Length);

 //initialize and fill list for the sorted palette
 ArrayList originalList = new ArrayList(colors);
 originalList.Sort();
</pre>


Viele Pixel verweisen auf die Paletteneintr&auml;ge, und die werden wir ebenfalls anpassen m&uuml;ssen.
Also sollten wir, wenn wir eine Farbe in die neue Palette verschieben, alten und neuen Index notieren.


<pre>
 //initialize list for the mapping of old indices to new indices
 SortedList oldIndexToNewIndex = new SortedList(colors.Length);
</pre>


Jetzt sind die Listen fertig, und wir k&ouml;nnen in die Nachricht eintauchen und
f&uuml;r jedes Bit eine Farbe verschieben...


<pre>
 Random random = new Random();
 int listElementIndex = 0;
 bool messageBit = false;
 int messageByte = message.ReadByte();

 //for each byte of the message
 while (messageByte &gt; -1) {
       //for each bit
       for (int bitIndex = 0; bitIndex &lt; 8; bitIndex++) {

           //decide which color is going to be the next one in the new palette

           messageBit = ((messageByte & (1 &lt; bitIndex)) &gt; 0) ? true : false;

           if (messageBit) {
              listElementIndex = 0;
           }else{
              listElementIndex = random.Next(1, originalList.Count);
           }
</pre>


...aber nicht vergessen, von welcher Position in der Original-Palette die Eintr&auml;ge kamen!
Tausende Pixel warten auf aktualisierte Farbindizes.


<pre>
           //log change of index for this color
           int originalPaletteIndex = Array.IndexOf(colors, originalList[listElementIndex]);
           if( ! oldIndexToNewIndex.ContainsKey(originalPaletteIndex)) {
               //add mapping, ignore if the original palette contains more than one entry for this color
               oldIndexToNewIndex.Add(originalPaletteIndex, resultList.Count);
           }

           //move the color from old palette to new palette
           resultList.Add(originalList[listElementIndex]);
           originalList.RemoveAt(listElementIndex);
     }

     //repeat this with the next byte of the message
     messageByte = message.ReadByte();
     }
 //copy unused palette entries
 foreach (object obj in originalList) {
    int originalPaletteIndex = Array.IndexOf(colors, obj);
    oldIndexToNewIndex.Add(originalPaletteIndex, resultList.Count);
    resultList.Add(obj);
 }

 //create new image
 Bitmap newImage = CreateBitmap(image, resultList, oldIndexToNewIndex);
 return newImage;
}
</pre>


Die korrespondierende <code>Extract</code>-Methode &auml;nelt einer Kombination der
<code>Hide</code>- und <code>Extract</code>-Methoden f&uuml;r Text-Listen.
Sie sortiert die Palette, rekonstruiert die Nachricht, und entfernt dann die Nachricht aus dem Bild
(produziert ein Bild mit einer Palette in Standard-Sortierung).
Das hier ist die Palette aus dem Beispiel, nachdem die Nachricht ausgelesen und entfernt wurde:


<img src="/images/paletteSortiert.png" width="264" height="264">

## Tags voller Text-Listen


Ein HTML-Document selbst ist keine sortierbare Liste, die Tags m&uuml;ssen in ihrer Reihenfolge bleiben.
Aber innerhalb der Tags stehen Attribute, und diese k&ouml;nnen beliebig sortiert werden.
Das hei&szlig;t, jedes Tag einer HTML-Seite kann <code>Attributes.Count-1</code> Bits speichern.
Die meisten Tags haben gerade genug Attribute f&uuml;r ein oder zwei Bits,
aber wir k&ouml;nnen die Nachricht &uuml;ber alle Tags der Seite verteilen.


<img src="/images/htmlTag.gif" width="238" height="187">

<pre>
public void Hide(String sourceFileName, String destinationFileName, Stream message, String alphabet) {
       //initializations skipped
       // ... ... ...

       //list the HTML tags
       HtmlTagCollection tags = FindTags(htmlDocument);

       //loop over tags
       foreach (HtmlTag tag in tags) {

       //write beginning of the tag
       insertTextBuilder.Remove(0, insertTextBuilder.Length);
       insertTextBuilder.AppendFormat("<{0}", tag.Name);

       //list attribute names
       String[] attributeNames = new String[tag.Attributes.Count];
       for (int n = 0; n &lt; attributeNames.Length; n++) {
          attributeNames[n] = tag.Attributes[n].Name;
       }
</pre>


Ab hier ist der Code fast der gleiche wie in den anderen <code>Hide</code>-Methoden:


<pre>
       //get default sorting
       StringCollection originalList = Utilities.SortLines(attributeNames, alphabet);
       StringCollection resultList = new StringCollection();

       if (tag.Attributes.Count > 1) {
          //the tag has capacoty for one or more bits

          for (int n = 0; n &lt; attributeNames.Length - 1; n++) {

              //get next bit of the message

              bitIndex++;
              if (bitIndex == 8) {
                 bitIndex = 0;
                 if (messageByte > -1) {
                    messageByte = message.ReadByte();
                 }
              }

              if (messageByte > -1) {

                 //decide which attribute is going to be the next one in the new tag

                 messageBit = ((messageByte & (1 << bitIndex)) > 0) ? true : false;

                 if (messageBit) {
                    listElementIndex = 0;
                 }else{
                    listElementIndex = random.Next(1, originalList.Count);
                 }
              }else{
                 listElementIndex = 0;
              }

              //move the attribute from old list to new list
              resultList.Add(originalList[listElementIndex]);
              originalList.RemoveAt(listElementIndex);
          }
       }

       if (originalList.Count > 0) {
          //add the last element - it never hides data
          resultList.Add(originalList[0]);
       }

</pre>


Die sortierten Attribute m&uuml;ssen zu&uuml;ck ins Dokument geschrieben werden.
Der gr&ouml;&szlig;te Teil des Codes wurde aus dem letzten HTML-Artikel dieser Serie kopiert, und muss hier hoffentlich nicht extra erkl&auml;rt werden. ;-)


<pre>
       HtmlTag.HtmlAttribute attribute;
       foreach (String attributeName in resultList) {
              attribute = tag.Attributes.GetByName(attributeName);

              insertTextBuilder.Append(" ");
              if (attribute.Value.Length > 0) {
                 insertTextBuilder.AppendFormat("{0}={1}", attribute.Name, attribute.Value);
              }else{
                 insertTextBuilder.Append(attributeName);
              }
       }

       //replace old tag with new tag
       //.. ... ...
}
</pre>


Hast du bemerkt, dass die drei <code>Hide</code>-Methoden fast das gleiche tun?
Die <code>Extract</code>-Methoden sind genauso &auml;hnlich.
Bitte lies sie im vollst&auml;ndigen Quelltext nach, falls du noch keine Bits-Allergie hast...


## Die Demo-Anwendung

<img src="/images/demo1.gif" width="298" height="190">


Die Anwendung besteht aus drei three &quot;Halbduplex-Dialogen&quot;.
Man kann entweder einen leeren Tr&auml;ger und eine Nachricht, oder einen Tr&auml;ger zum Auslesen einer Nachricht angeben.
Bei Text-Listen und Bildern kann man das Ergebnis sofort sehen.


<img src="/images/demo2.png" width="610" height="414">


In den Dialogen f&uuml;r Text-Listen und HTML-Dokumente kann man eine &quot;Alphabet-Datei&quot; angeben.
Das ist eine Textdatei mit einem benutzerdefinierten Alphabet.
Ein beispielhaftes Alphabet findest du in <i>testdata/demo.txt</i>.


<img src="/images/demo3.png" width="610" height="360">

