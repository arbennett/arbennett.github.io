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

In ASCII each character in a text document is 7-bits long, and in UTF-8 a character occurring in the ASCII alphabet is encoded in 8-bits (1 byte).  This encoding gives all characters equal weight, leading to an inefficient (although conveniently standardized) storage method.  By giving each character a variable length code we can reduce the number of bits required to store the data, hence compression.  To do this we will begin by constructing the Huffman Tree.

  

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