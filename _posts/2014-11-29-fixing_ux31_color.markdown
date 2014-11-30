---
layout: post
title:  "Screen calibration for the Asus Zenbook on Linux"
date:   2014-11-29 
categories: linux
---
Recently I purchased an Asus Zenbook for travel, and noticed that the screen color was noticeably tinted green/yellow.  This was easy to fix under Windows, which has an easy to use though not feature-packed calibration tool.  However, on Linux I ran into some difficulties getting the screen the way I wanted it.  I do not have a colorimeter, which is required for many of the calibration tools that I could find.  Also, since I do not have a colorimeter it should be noted that I am not doing any professional visual media work on this machine, so the calibration I ended up with is for my eyes only.  My installation of Linux that you can use as reference is Arch Linux with kernel version 3.17, and Gnome 3.14 with xdg for managing the desktop.

The tool I used to calibrate the screen was called [xcalib](http://xcalib.sourceforge.net/).  This is a nice, compact, command line tool that fit perfectly for what I wanted.  There appear to be some more robust color managers such as lprof and argyll.  I cannot speak for argyll, but it appears that lprof has not been maintained some time and requires some larger dependencies (qt3).

Once you have xcalib you will find that it is easy to use, but difficult to master (as with many similar tools).  There are a couple of included color profiles that I used to tune settings.  I made a small script and took the gamma_1.icc profile and put them in /opt/Color. The script that I settled on was 
{% hightlight bash %}
#!/bin/bash
sleep 1
xcalib -d :0 -s 0 -v -co 80.0 -red 1.0  2.1 92.0 -green 1.0 2.1 89.0 -blue 1.0 2.1 98.0  /opt/Color/gamma_1_0.icc
{% endhighlight %}

After getting this script set up I put a new entry for xdg.  In /etc/xdg/autostart I made a new entry, color-correction.desktop.  In that file I just have:

{% highlight bash %}
[Desktop Entry]
Name=ColorCorrection
GenericName=Corrects the Color to be less yellow
Comment=Generated for ASUS UX31A
Exec=/opt/Color/colorCorrection
Terminal=false
Type=Application
X-GNOME-Autostart-enabled=true
{% endhighlight %}

And that's it!  Now when logging in I see the color change about the same time Gnome is ready to be used.  Thanks to the xcalib developers for making the perfect tool for this situation.