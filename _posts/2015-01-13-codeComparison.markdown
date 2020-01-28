---
layout: post
title:  "First test with Julia"
date:   2015-01-13
categories: Software
---

I've been interested in the Julia language [(link)](julialang.org) for a while.  During the recent holidays I had some plane ride time to write up some simple comparisons with other languages to see how well it fared.  I prepared a couple of sets of "benchmarks" in 4 languages: C, Python2.7, Java, and Julia.  First I implemented a simple Newton's method root-finding algorithm for a simple polynomial, running it over the course of 10000 iterations for testing.  The second benchmark was an implementation of the BTW Sandpile model that I evolved 10000 steps.  Here are the results:

![Language comparison chart](http://arbennett.github.io/imgs/language_comparison.png)

Overall, Julia fared pretty well.  The code complexity was on par with Python, and the runtimes on par with Java.  I think I will continue to mess around with it for a while and see how it does with larger problems.

Julia, as a language, is hit or miss to me.  I understand that a lot of the syntax is supposed to be "matlab-y" but I really dislike array indexing starting at 1.  I know that's going to cause me some troubles as I use it more.  But, the ability to call C natively is exciting, along with the maturity of the plotting capabilities already.
