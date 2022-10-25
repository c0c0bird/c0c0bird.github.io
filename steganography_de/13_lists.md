---
title: Listen effizient
category: Steganografie
subcategory: Listen
order: 13
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [C# Quellcode - 49.62 Kb]({% link /assets/steganodotnet19_src.zip %})

## Worum geht es?

Menschen lieben Chaos. Viele Dinge könnten geordnet und sortiert werden, aber üblicherweise tun wir es nicht. Alles das eine eindeutige Ordnung haben könnte kann genutzt werden, um eine Zahl zu codieren. Denn es gibt <code>Fakultaet(anzahl)</code> mögliche Ordnungen welche man nummerieren kann.

Hier lernst du, wie du irgendeine Nachricht in der Sortierung beliebiger Dinge codieren kannst. (Vielleicht wirst du nachher aus reinem Spaß das Zimmer aufräumen.)

<img src="/images/steganodotnet191.png" alt="screen" width="541" height="542" />

Ich war ziemlich beeindruckt von Da wäre <a href="http://www.jerde.net/peter/cards/" target="_blank">Peter's Beispiel wie man einen Text in einem Spiel mit 52 Karten versteckt.</a> Der Algorithmus ist so klar verständlich und leicht zu implementieren. Deshalb habe ich ihn angepasst, um jeden beliebigen binären Datenstrom in jeder <code>Collection&lt;IComparable&gt;</code> zu speichern.

Die Klasse kann benutzt werden, um Sachen in - zum Beispiel - Noten, Einkaufslisten oder die Palette eines GIF-Bildes hinein zu codieren. Falls du den Sport <i>Geo Caching</i> magst, hast du oft mit Listen von Wegpunkten zu tun. Die Reihenfolge der Wegpunkte in einer LOC- oder GPX-Datei hängt vielleicht von der Zeit ab zu der du die Orte besuchen willst, von der Entfernung nach Hause oder schlicht vom Zufall. Das bedeutet, wir können den Spielkarten-Algorithmus auf deine Wegpunkte anwenden, um einen kurzen Text in der Reihenfolge der Orte zu speichern.

Dieser Artikel beschreibt die verallgemeinerte C#-Implementierung und stellt eine Demo-Anwendung sowohl für einfache Wortlisten als auch für Geo Caching-Dateien vor. Das Programm kann GPX- und LOC-Dateien importieren, durch Sortieren einen UTF-8-Text hinein codieren und das Ergebnis exportieren. (Wenn das niemand braucht, ist es immer noch ein schneller GPX-LOC-Konverter.)
## Codieren durch Sortieren

Der Plan lautet, eine Collection von <code>IComparable</code> zu nehmen und die Objekte zu vertauschen, um den Inhalt eines <code>Stream</code> zu notieren:#

<pre>public static void Encode&lt;T&gt;(
	Collection&lt;T&gt; source, // the carrier list
	Collection&lt;T&gt; result, // an empty list to store the result
	Stream messageStream) // the message to encode
	where T: class, IComparable
{
</pre>

Jede Nachricht ist eine Zahl. Wir müssen nicht wissen, ob sie ein Text, eine Musikdatei oder sonstwas ist, denn wir werden den ganzen Stream als eine große Ganzzahl behandeln.
<pre>// initialize message

messageStream.Position = 0;
byte[] buffer = new byte[messageStream.Length];
messageStream.Read(buffer, 0, buffer.Length);
BigInteger message = new BigInteger(buffer);
</pre>

Bevor wir eine Objektliste sortieren können, müssen wir einen Sortierschlüssel und seine Default-Reihenfolge festlegen. Für eine Liste von Personen könnten das die alphabetisch sortierten Namen sein. Geo Caches (Wegpunkte) lassen sich nach Koordinaten sortieren, oder nach ID, Name, Entfernung von zu Hause, ... wie auch immer, <code>IComparable</code>-Objekte vergleichen sich selbst. Wir sortieren die Trägerobjekte so wie sie es gern hätten und definieren diese Reihenfolge als "0".
<pre>// initialize source list in "0"-order
T[] sortedItems = new T[source.Count];
source.CopyTo(sortedItems, 0);
Array.Sort(sortedItems);
</pre>

Eine Liste in Default-Reihenfolge (z.B. alphabetisch) codiert den Wert "0". Die ersten beiden Elemente zu vertauschen heißt "1". Eine Möglichkeit, die Elemente der Quellliste auf die freien Felder der Zielliste zu verteilen, ist die ganzzahlige Division: Für jedes Element der Quellliste teilen wir die Nachricht durch die Anzahl freier Felder und legen das Objekt [Rest] Felder von links ab. Folglich brauchen wir eine Trägerliste die genauso lang wie die Quellliste ist, und wir müssen die Anzahl gerade noch freier Felder darin kennen.
<pre>// initialize carrier

Collection&lt;int&gt; freeIndexes = new Collection&lt;int&gt;();
result.Clear();
for (int n = 0; n &lt; source.Count; n++)
{
	freeIndexes.Add(n);
	result.Add(null);
}
</pre>

Nach diese Vorbereitungen können wir über die Quellliste iterieren (die in "0"-Ordnung ist) und die neue Position jedes Objekts aus der noch übrigen Nachricht berechnen.
<pre>int skip = 0;
for (int indexSource = 0; indexSource &lt; source.Count; indexSource++)
{
	skip = (message % freeIndexes.Count).IntValue();
	message = message / freeIndexes.Count;
	int resultIndex = freeIndexes[skip];
	result[resultIndex] = sortedItems[indexSource];
	freeIndexes.RemoveAt(skip);
}
</pre>

Hier ist die vollständige Codier-Methode:
<pre>public static void Encode&lt;T&gt;(
	Collection&lt;T&gt; source, // the carrier list
	Collection&lt;T&gt; result, // an empty list to store the result
	Stream messageStream) // the message to encode
	where T: class, IComparable
{
	// sort source list into "0"-order
	T[] sortedItems = new T[source.Count];
	source.CopyTo(sortedItems, 0);
	Array.Sort(sortedItems);
	
	// initialize message
	messageStream.Position = 0;
	byte[] buffer = new byte[messageStream.Length];
	messageStream.Read(buffer, 0, buffer.Length);
	BigInteger message = new BigInteger(buffer);
	
	// initialize carrier
	Collection&lt;int&gt; freeIndexes = new Collection&lt;int&gt;();
	result.Clear();
	for (int n = 0; n &lt; source.Count; n++)
	{
		freeIndexes.Add(n);
		result.Add(null);
	}

	// encode by integer division
	int skip = 0;
	for (int indexSource = 0; indexSource &lt; source.Count; indexSource++)
	{
		skip = (message % freeIndexes.Count).IntValue();
		message = message / freeIndexes.Count;
		int resultIndex = freeIndexes[skip];
		result[resultIndex] = sortedItems[indexSource];
		freeIndexes.RemoveAt(skip);
	}
}

</pre>
## Decodieren der Sortierung

Diesmal beginnen wir mit der Trägerliste in besonderer Ordnung und einer Nachricht von "0". Für jedes Listenelement ist der Rest der ganzzahligen Division aus dem Codierprozess zu finden, so dass wir die Divisionen umkehren können. Woher wissen wir, wie viele freie Felder ein Objekt übersprungen hatte? Ganz einfach: Die Quellliste war in "0"-Ordnung, so dass nach einem Objekt nur noch "größere" Objekte folgen konnten. Das heißt, die Anzahl der "größeren" Elemente nach links ist genau der Sprungwert welcher genau der Rest der vorherigen Division ist.

Die Nachricht wurde durch die Anzahl freier Felder geteilt, was einen neuen Rest ergab. Vor diesem Schritt war sie bereits durch alle frühreren Anzahlen freier Felder geteilt worden. Deshalb könen wir den Sprungwert mit <code>(list.Count - position)</code> für jede vorherige Position multiplizieren, die Werte für alle Elemente aufsummieren - und das Ergebnis ist wieder die Nachricht als Zahl.

Hier ist die Decodier-Methode:
<pre>public static void Decode&lt;T&gt;(
	Collection&lt;T&gt; carrier, // the carrier list
	Stream messageStream)  // an empty tream to store the message
	where T : class, IComparable

{

	// get the source list in "0"-order
	T[] sortedItems = new T[carrier.Count];

	carrier.CopyTo(sortedItems, 0);
	Array.Sort(sortedItems);
	
	// initialize message
	BigInteger message = new BigInteger(0);

	for (int carrierIndex = 0; carrierIndex &lt; carrier.Count; carrierIndex++)
	{
		// count skipped slots to reconstruct the division's remainder
		int skip = 0;

		for (int countIndex = 0; countIndex &lt; carrierIndex; countIndex++)
		{
			if (carrier[countIndex].CompareTo(carrier[carrierIndex]) &gt; 0)
			{ // There is a bigger item to the left. It's place
			  // must have been skipped by the current item.
				skip++;
			}
		}

		// Revert the division that resulted in this skip value
		int itemOrdinal = Array.IndexOf(sortedItems, carrier[carrierIndex])+1;
		BigInteger value = new BigInteger(skip);
		for (int countIndex = 1; countIndex &lt; itemOrdinal; countIndex++)
		{
			value *= (carrier.Count - countIndex + 1);
		}

		message += value;
	}


	// write message
	byte[] messageBytes = message.getBytes();
	messageStream.Write(messageBytes, 0, messageBytes.Length);
	messageStream.Position = 0;
}
</pre>
## Flexiblible Sortierschlüssel

Beim sortieren von Objekten ist ein eindeutiger Sortierschlüssel wichtig: In einem Adressbuch können zwei Leute denselben Namen, aber eindeutige Geburtstage besitzen - in einem anderen Adressbuch sind sie Namen eindeutig, aber nicht die Geburtstage. Auf einer Route aus Wegpunkten mag es Punkte mit gleicher Länge oder Breite geben, jedoch mit eindeutigem Zeitstempel - auf anderen Routen kann die Höhe die einzige eindeutige Eigenschaft sein.

Da der beste Sortierschlüssel von den gegebenen Daten abhängt, sollte er eine Eigenschaft der Collection sein. Nachdem wir sie mit Comparable-Objekten gefüllt haben, sagen wir ihr einfach, welches Feld das Vergleichbare ist.
<pre>// fill the typesafe flexi-collection with objects
FlexibleComparableCollection&lt;GeoCache&gt; source = 
		new FlexibleComparableCollection&lt;GeoCache&gt;();
foreach (ListViewItem viewItem in lstGC.Items) {
	source.Add((GeoCache)viewItem.Tag);
}
// get the property name from the selected radio button
source.ComparablePropertyName = GetGCSortPropertyName();
</pre>

Wie funktioniert das? Mit etwas Reflexion! <code>FlexibleComparableCollection</code> ist eine typsichere generische Collection. Wenn <code>ComparablePropertyName</code> gesetzt ist, schlägt sie die Eigenschaft mit diesem Namen in ihrem Elemente-Typ nach und informiert jedes Element.
<pre>public class FlexibleComparable : IComparable
{	// read the key value only when it changes
	private object comparableValue;
	internal void UpdateComparableValue(PropertyInfo comparableProperty)
	{
		this.comparableValue = comparableProperty.GetValue(this, null);
	}

	// Interface IComparable
	public int CompareTo(object obj)
	{
		IComparable thisValue = this.comparableValue as IComparable;
		if (thisValue == null) return -1;

		IComparable otherValue;
		FlexibleComparable other = obj as FlexibleComparable;
		
		// get the other's key value (if any), or compare to the whole object
		if (other == null) otherValue = other as IComparable;
		else otherValue = other.comparableValue as IComparable;
	
		// compare, if you can!
		if (otherValue == null) return 1;
		else return thisValue.CompareTo(otherValue);
	}
}
</pre>

Das ist alles was du brauchst um jede Sammlung von irgendwas als Trägermedium zu nutzen. Du musst nur eine eindeutige Eigenschaft aussuchen oder erfinden und sicherstellen, dass die Nachricht in die Liste rein passt.
<pre>private void EncodeInSomething()
{	// set up carrier list
	FlexibleComparableCollection&lt;Something&gt; source = GetList();
	source.ComparablePropertyName = "ChosenField";
	// initialize message and destination list
	Stream message = TextToStream(txtMessage.Text);
	Collection&lt;Something&gt; result = new Collection&lt;Something&gt;();
	// encode message
	OrderUtility.Encode&lt;Something&gt;(source, result, message);
	SaveList(result);
}

private void DecodeInSomething()
{	// set up carrier list
	FlexibleComparableCollection&lt;Something&gt; carrier = GetList();
	carrier.ComparablePropertyName = "ChoseField";
	// decode message
	MemoryStream message = new MemoryStream();
	OrderUtility.Decode&lt;Something&gt;(carrier, message);
	txtMessage.Text = new StreamReader(message).ReadToEnd();
}
</pre>
## Die Demo-Anwendung

Die Anwendung lässt dich eine Wortliste eingeben und GPS-Daten importieren/bearbeiten. Sie zeigt die Kapazität der Liste an und codiert/decodiert Text. Zum besseren Verständnis werden auf der letzten Tab-Seite die Zahlen 1..256 als Liste verwendet, so dass offensichtlich wird, wie sich die Reihenfolge mit dem Text ändert.
## Schlusswort

Mit diesen paar Zeilen Code kann man einen Sortierschlüssel auf einer Liste definieren, eine binäre Nachricht in die Reihenfolge der Liste codieren und wieder decodieren. Einzige Voraussetzung ist eine eindeutige Eigenschaft, aber sie muss nur für die betroffene Liste eindeutig sein, nicht allgemein für die Objektklasse.

Mit Dank an <a href="https://www.codeproject.com/MemberArticles.aspx?amid=44086">Chew Keong TAN</a> für seine <a href="http://www.codeproject.com/KB/cs/biginteger.aspx">C# BigInteger Class</a> und Peter Jerde für die <a href="http://jerde.net/peter/cards/explanation.html">Erklärung der Grundlagen</a>.
