---
layout: default
title: Huffman Coding with Compressure
permalink: /projects/compressure/
---

Introduction 
============
Storing and transmitting data is a fact of life in the modern world.  Often it would be nice if we could shrink the amount of space the data takes up, so we could transfer it more quickly or so that it wouldn't fill the disk right away.  The process of shrinking data is called compression.  When discussing data compresssion there are two main categories: lossless, and non-lossless (or lossy, for the initiated).  Just as it sounds, lossless data compression is a form where the entire full set of pre-compressed data is still somehow encoded in the compression (example: .zip files).  Lossy compressions are flattenings of the original data that would pass adequately for the original thing (example: .mp3 files).  This project deals exclusively with the lossless compression algorithm called Huffman Encoding.

Huffman Encoding
================
The Huffman Encoding method was first described by David Huffman in the early 1950's [\[1\]](http://compression.ru/download/articles/huff/huffman_1952_minimum-redundancy-codes.pdf).   The algorithm takes fixed-length data and converts each symbol to a variable length code based on the number of times the symbol occurs (symbols that occur more often get shorter codes, less likely symbols get longer codes).  For the sake of demonstration we will assume that the data we'd like to compress is pure text data, encoded in ASCII or UTF-8.  For more information about how ASCII encoded text works check out [\[2\]](http://en.wikipedia.org/wiki/ASCII) and [\[3\]](http://en.wikipedia.org/wiki/UTF-8).

In ASCII each character in a text document is 7-bits long, and in UTF-8 a character occurring in the ASCII alphabet is encoded in 8-bits (1 byte).  This encoding gives all characters equal weight, leading to an inefficient (although conveniently standardized) storage method.  By giving each character a variable length code we can reduce the number of bits required to store the data, hence compression.  To do this we will begin by constructing the Huffman tree.  We will use a simple method of building the tree with a priority queue that has a time complexity of O(log n).  There is an O(n) method of building the tree if the symbols are sorted by the frequency which they occur, but we will not worry ourselves with it.  

The first step to building the Huffman tree is to count the number of times each character occurs in the data set we are trying to compress.  Once we have the characters and their frequencies we will build the tree.  Each of the leaf nodes of the tree will contain one of the characters and the number of times it occurred in the data.  We begin by putting each of these nodes into the priority queue:

	for(int i = 0; i < charFreqs.length -1 ; i++){
		if(charFreqs[i] > 0){
			huffQueue.offer( new HuffmanTree<Character>((char) i, charFreqs[i]) );
		}
	}

Then we can build up our tree by popping two elements off of the queue and combining them into a new Huffman tree, and finally putting that back onto the priority queue.  We do this until the final tree is all that is in the queue:


	while(huffQueue.size() > 1){
		leftTree = huffQueue.poll();
		rightTree = huffQueue.poll();
		tempTree = new HuffmanTree<Character>(leftTree, rightTree);
		huffQueue.offer(tempTree);
	}

Once we have the tree we can begin building the codes used to compress the data.  Building the codes follows a simple algorithm:  starting at the root of the Huffman tree we will traverse until we reach each leaf node.  The path taken to the leaf will determine the code; every left we take is denoted a 0 and every right we take is denoted a 1.  We will store the character and codes as key-value pairs in a hashmap:

	private HashMap<Character, String> buildMap(HashMap<Character,String> codeMap, HuffmanTree<Character> huffTree, StringBuilder code){
		if (huffTree.symbol != null){
			// Put the <Symbol,Code> pair in the map
			codeMap.put(huffTree.symbol,code.toString());
		} else {
			// Traverse left
			code.append(0);
			codeMap = buildMap(codeMap, huffTree.left, code);
			code.deleteCharAt(code.length()-1);
			
			// Traverse right
			code.append(1);
			codeMap = buildMap(codeMap, huffTree.right, code);
			code.deleteCharAt(code.length()-1);
		}
		
		return codeMap;
	}

To write out the compressed file there are two main steps remaining.  First, we need to encode some sort of header so that we can recover the data.  Second, we will go through and replace each character in the original message with the Huffman code from our hashmap. To write out the header we will encode the Huffman tree by writing the structure of the Huffman tree:

	private void writeHeader(BinaryWriter output, HuffmanTree<Character> huffTree){
		if(huffTree.symbol == null){
			output.write(0);
			if(huffTree.left != null)  writeHeader(output,huffTree.left);
			if(huffTree.right != null) writeHeader(output,huffTree.right);
		}else{
			output.write(1);
			Integer symbol = (int) huffTree.symbol.charValue();
			output.writeByte( Integer.toBinaryString(symbol) );
		}
	}


Next we simply have to parse through the original text and replace the characters with their associated Huffman code:

	while( (line = reader.readLine()) != null ){
		for(char codeChar : line.toCharArray()){
			// Get the code for each character
			code = codeMap.get(codeChar);
			// Write bitwise
			for( char bit : code.toCharArray() ){
				output.write((int) (bit - 48)); //Scaled for ascii
			}
		}
	}

Huffman Decoding
================
talk about writing a header and how to rebuild the huffman tree or code mapping

Java Implementation
===================
needing to write the bitreader/writer since java normally handles things as bytes

Analysis
========
show differences between random symbols, regular text, and compare to tar

Conclusion
==========
Say something about adaptive huffman encoding and other improvements like lzq

References
==========
[1] [A Method for the Construction of Minimum-Redundancy Codes](http://compression.ru/download/articles/huff/huffman_1952_minimum-redundancy-codes.pdf)

[2] [ASCII Wikipedia Page](http://en.wikipedia.org/wiki/ASCII)

[3] [UTF-8 Wikipedia Page](http://en.wikipedia.org/wiki/UTF-8)

Another header
==============
Hopefully this gives me a scrollbar