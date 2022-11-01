---
title: Strickmuster
category: Steganografie
subcategory: Sonstiges
order: 16
layout: default
---

# {{ page.title }}

<img src="/images/save.gif" width="14" height="14" alt="" border="0"> [Java Quellcode - 4 kB]({% link /assets/SteganoYarn_src.zip %})

## Introduction

As you possibly remember from school, a <a href="https://en.wikipedia.org/wiki/Turing_machine">Turing Machine</a> is a mathematical model of computation. It defines an abstract machine working with characters on an endless tape. In theory, a system that simulates a turing machine is able to support all tasks accomplishable by computers.

Each cell on the tape has to be writable only once. For a second write access, you can copy the used tape to a new segment. As every calculation can be done in binary, the alphabet of writable characters can be limited to {0, 1}, though a larger alphabet may be good to save tape. In this article we'll use {0, 1, 2, 3} to encode everything in base-4.

A cheap and simple way to simulate the endless, writable tape is a clew of yarn. You can write characters by knitting different stitches. If you need to overwrite a used position, just start a new row; that is like rewinding and copying the tape of the turing machine.

This article explains how to encode any content as stitches, render a knitting chart and decode the finished yarn-tape again. It uses the <a href="https://xmlgraphics.apache.org/batik/">Apache Batik SVG Toolkit</a> to draw the pattern.

You might want to feed the patterns into a knitting machine, turning it into a real hardware Turing Machine. You can as well knit them by hand to transmit secret messages just by letting your messenger wear a warm hat.
## The Application

<img src="/images/steganodotnet21_chart.png" alt="" />

The Java application accepts a text, converts the characters to groups of base-4 numbers and translates those into a row of stitches:

* 0 = knit
* 1 = purl
* 2 = yarn over, knit
* 3 = yarn over, purl


The screenshot shows the knitting chart generated from the text "<i>Hi CP!</i>". Only the first row contains information. The second row is there to reduce the excess stitches created by the digits "2" and "3". All following rows are just repetitions to make the code look like decoration.

Such a pair of rows would look strange in a piece of cloth. But usually, knitted stuff has decorative patterns and everything looks like a pattern, if you just repeat if often enough. So, feel free to repeat the code row until the result looks pretty unsuspicious.
## Encoding and Decoding

This little demonstration uses only latin text. As a character is a value between 0 and 255, it corresponds to four digits between 0 and 3. Those digits are concatenated to one long string and then sent to the graphics frame.

<span id="prehide882844" class="code-keyword">private</span> <span class="code-keyword">void</span> encode(<span class="code-sdkkeyword">String</span> clearText){
<pre id="pre882844" class="notranslate" lang="Java" style="margin-top: 0px;">	ByteBuffer bytes = Charset.forName(<span class="code-string">"</span><span class="code-string">ISO-8859-15"</span>).encode(clearText);
	StringBuilder resultBuilder = <span class="code-keyword">new</span> StringBuilder(clearText.length() * <span class="code-digit">4</span>);

    <span class="code-comment">//</span><span class="code-comment"> convert each character</span>
	<span class="code-keyword">while</span>(bytes.hasRemaining()){
		<span class="code-keyword">int</span> currentValue = bytes.get();			
		<span class="code-sdkkeyword">String</span> currentWord = Integer.toString(currentValue, <span class="code-digit">4</span>);
		
		<span class="code-comment">//</span><span class="code-comment"> pad the number to a length of 4</span>
		<span class="code-keyword">while</span>(currentWord.length() &lt; <span class="code-digit">4</span>){
			currentWord = <span class="code-string">"</span><span class="code-string">0"</span> + currentWord;
		}
		
		resultBuilder.append(currentWord);
	}
	
	ChartFrame frame = <span class="code-keyword">new</span> ChartFrame();
	frame.drawChart(clearText, resultBuilder.toString());
}
</pre>

When reading a piece of cloth manually, you may note down digits for the kinds of stitches you recognize...

<img src="/images/steganodotnet21_decode.jpg" alt="" />

... then call the decoder with the row of stitches you found. They'll be decoded to the original text:

<img src="/images/steganodotnet21_commandline.png" alt="" />

The <code>decode</code> method converts the chain of numbers back to the original message.

<span id="prehide195397" class="code-keyword">private</span> <span class="code-sdkkeyword">String</span> decode(<span class="code-sdkkeyword">String</span> encodedText){
<pre id="pre195397" class="notranslate" lang="Java" style="margin-top: 0px;">	<span class="code-keyword">int</span> n = <span class="code-digit">0</span>;
	StringBuilder resultBuilder = <span class="code-keyword">new</span> StringBuilder(encodedText);

    <span class="code-comment">//</span><span class="code-comment"> translate blockwise to characters</span>
	<span class="code-keyword">while</span>(n &lt; resultBuilder.length()){
		<span class="code-sdkkeyword">String</span> currentWord = resultBuilder.substring(n, n+4);
		
		<span class="code-keyword">int</span> currentValue = Integer.parseInt(currentWord, <span class="code-digit">4</span>);
		<span class="code-sdkkeyword">String</span> currentClearChar;
		
		<span class="code-comment">//</span><span class="code-comment"> decode or ignore</span>
		<span class="code-keyword">if</span>(currentValue &lt; <span class="code-digit">256</span>){
			ByteBuffer buffer = ByteBuffer.wrap(<span class="code-keyword">new</span> <span class="code-keyword">byte</span>[]{(<span class="code-keyword">byte</span>)currentValue});
			currentClearChar = Charset.forName(<span class="code-string">"</span><span class="code-string">ISO-8859-15"</span>).decode(buffer).toString();
		}<span class="code-keyword">else</span>{
			currentClearChar = <span class="code-string">"</span><span class="code-string">?"</span>;
		}
		
		<span class="code-comment">//</span><span class="code-comment"> compress 4 digits to 1 character		</span>
		resultBuilder.replace(n, n+4, currentClearChar);
		n++;
	}
	
	<span class="code-keyword">return</span> resultBuilder.toString();
}
</pre>
## Rendering the Chart

For each digit, the corresponding stitches have to be drawn. I used the knit chart symbols by the <a href="http://www.craftyarncouncil.com/chart_knit.html">Craft Yarn Council</a>, which are easy to render with geometric shapes.

The chart consists of three components: Column numbers, odd rows encoding the payload, even rows fixing the count of stitches.
<pre id="pre978849" class="notranslate" lang="Java" style="margin-top: 0px;"><span class="code-keyword">public</span> <span class="code-keyword">void</span> drawChart(<span class="code-sdkkeyword">String</span> title, <span class="code-sdkkeyword">String</span> base4Digits){
	titleText = title;

    <span class="code-comment">//</span><span class="code-comment"> prepare SVG document</span>
	DOMImplementation impl = SVGDOMImplementation.getDOMImplementation();
	<span class="code-sdkkeyword">String</span> svgNS = SVGDOMImplementation.SVG_NAMESPACE_URI;
	SVGDocument doc = (SVGDocument) impl.createDocument(svgNS, <span class="code-string">"</span><span class="code-string">svg"</span>, null);
	SVGGraphics2D g = <span class="code-keyword">new</span> SVGGraphics2D(doc);
	g.setFont(<span class="code-keyword">new</span> Font(<span class="code-string">"</span><span class="code-string">sans serif"</span>, Font.PLAIN, <span class="code-digit">10</span>));
	
	<span class="code-comment">//</span><span class="code-comment"> calculate chart width</span>
	countColumns = getCountCols(base4Digits);
	countRows = (<span class="code-keyword">int</span>)(<span class="code-digit">200</span> / boxSize);
	svgCanvas.setSize((2+countColumns)*boxSize, countRows*boxSize);
	
	<span class="code-comment">//</span><span class="code-comment"> fill background</span>
	g.setPaint(Color.darkGray);
	g.fillRect(<span class="code-digit">0</span>, <span class="code-digit">0</span>, svgCanvas.getWidth(), svgCanvas.getHeight());
	
	<span class="code-comment">//</span><span class="code-comment"> draw odd coding rows, even fixing rows</span>
	<span class="code-keyword">for</span>(<span class="code-keyword">int</span> y = <span class="code-digit">0</span>; y &lt; (countRows-<span class="code-digit">2</span>); y+=2){
	    <span class="code-comment">//</span><span class="code-comment"> code symbols may add stiches</span>
    	drawOddRow(y, base4Digits, g);
    	
    	<span class="code-comment">//</span><span class="code-comment"> fix excess stitches by knitting two together</span>
    	drawEvenRow(y+1, base4Digits, g);	    	
    }
	
	<span class="code-comment">//</span><span class="code-comment"> draw column numbers</span>
	<span class="code-keyword">for</span>(<span class="code-keyword">int</span> x=0; x &lt; countColumns; x++){
		drawColNumber(x+1, countRows-<span class="code-digit">1</span>, g);
	}
	
	<span class="code-comment">//</span><span class="code-comment"> prepare and show frame</span>
	titleText = titleText + <span class="code-string">"</span><span class="code-string"> - "</span> + <span class="code-string">"</span><span class="code-string">start with "</span> + (base4Digits.length()+2) + <span class="code-string">"</span><span class="code-string"> stitches"</span>;
	g.getRoot(doc.getDocumentElement());
	svgCanvas.setSVGDocument(doc);
	svgCanvas.getParent().setPreferredSize(svgCanvas.getSize());
	pack();
	setVisible(true);
}
</pre>

Drawing a row is simple: Place one knit symbol at each end, then fill the space with symbols:
<pre id="pre422944" class="notranslate" lang="Java" style="margin-top: 0px;"><span class="code-keyword">private</span> <span class="code-keyword">void</span> drawOddRow(<span class="code-keyword">int</span> y, <span class="code-sdkkeyword">String</span> base4Digits, SVGGraphics2D g){
	<span class="code-comment">//</span><span class="code-comment"> label and left edge</span>
	drawRowNumber(<span class="code-digit">0</span>, y, g);
	drawKnit(<span class="code-digit">1</span>, y, g);

    <span class="code-comment">//</span><span class="code-comment"> append one or two symbols per digit</span>
	<span class="code-keyword">int</span> x = <span class="code-digit">2</span>;
	<span class="code-keyword">for</span>(<span class="code-keyword">int</span> n=0; n &lt; base4Digits.length(); n++){
		<span class="code-keyword">char</span> currentChar = base4Digits.charAt(n);
		
		<span class="code-comment">/*</span><span class="code-comment"> draw box
		   0 = knit
		   1 = purl
		   2 = yarn over, knit
		   3 = yarn over, purl
		*/</span>
		<span class="code-keyword">switch</span>(currentChar){
    		<span class="code-keyword">case</span> <span class="code-string">'</span><span class="code-string">0'</span>: drawKnit(x, y, g); <span class="code-keyword">break</span>;
    		<span class="code-keyword">case</span> <span class="code-string">'</span><span class="code-string">1'</span>: drawPurl(x, y, g); <span class="code-keyword">break</span>;
    		<span class="code-keyword">case</span> <span class="code-string">'</span><span class="code-string">2'</span>: drawYarnOver(x, y, g);
    				  drawKnit(++x, y, g);
    		          <span class="code-keyword">break</span>;
    		<span class="code-keyword">case</span> <span class="code-string">'</span><span class="code-string">3'</span>: drawYarnOver(x, y, g);
    				  drawPurl(++x, y, g);
    				  <span class="code-keyword">break</span>;
		}
		
		x++;
	}

	<span class="code-comment">//</span><span class="code-comment"> right edge</span>
	drawKnit(x, y, g);
} 
</pre>

The chart symbols are primitive shapes. These are a few examples:
<pre id="pre518704" class="notranslate" lang="Java" style="margin-top: 0px;"><span class="code-keyword">private</span> <span class="code-keyword">void</span> drawBox(<span class="code-keyword">int</span> col, <span class="code-keyword">int</span> row, SVGGraphics2D g){
	<span class="code-keyword">int</span> x = col*boxSize;
	<span class="code-keyword">int</span> y = row*boxSize;
	Rectangle2D.Double box = <span class="code-keyword">new</span> Rectangle2D.Double(x, y, boxSize, boxSize);
	
	g.fill(box);
	g.setColor(Color.black);
	g.draw(box);
}

<span class="code-keyword">private</span> <span class="code-keyword">void</span> drawColNumber(<span class="code-keyword">int</span> col, <span class="code-keyword">int</span> row, SVGGraphics2D g){
	g.setColor(Color.black);
	g.setPaint(Color.cyan);
	drawBox(col, row, g);
	g.drawString(<span class="code-sdkkeyword">String</span>.valueOf(col),
	       (col*boxSize)+boxPadding,
	       (row*boxSize)+boxSize-boxPadding);
}

<span class="code-keyword">private</span> <span class="code-keyword">void</span> drawYarnOver(<span class="code-keyword">int</span> col, <span class="code-keyword">int</span> row, SVGGraphics2D g){
	g.setColor(Color.black);
	g.setPaint(Color.white);
	
	<span class="code-keyword">int</span> x = col*boxSize;
	<span class="code-keyword">int</span> y = row*boxSize;
	<span class="code-keyword">int</span> height = boxSize-(boxPadding*<span class="code-digit">2</span>);
	
	Ellipse2D circle = <span class="code-keyword">new</span> Ellipse2D.Double(
				x+boxPadding, y+boxPadding, 
				height, height);
	
	drawBox(col, row, g);
	g.draw(circle);
}
</pre>
## Points of Interest

So far, all we did was encoding and decoding. We are able to fill the tape of our fluffy Turing Machine with characters.

That is good enough to hide text in plain sight wearing a woolen scarf, which may be your only chance to smuggle secrets from Alaska to Siberia.

The next step could be to program a knitting machine, to make it process a calculation line by line. I hope you enjoyed this article and take a closer look at other people's textiles. 	</div>

