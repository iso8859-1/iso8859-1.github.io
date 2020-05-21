---
layout: post
title:  "Changing Windows"
date:   2020-05-21 13:21:00 +0200
categories: fantasygrounds
---

# Changing Windows

Now it's time to start modifying the CoreRPG for our purpose. To do that, you
need to either modify, replace or add new windowclasses. I decided to follow
the approach that I've seen in all rulsets inspected - so that anymone picking
up my ruleset knows where to look.

## A first experiment

I wanted to play around with the npc sheet to find out how everything works.
At first, I created in campaign a file called "record_npc.xml". Please bear in
mind, that this is just a convention. But if you are in Rome, do as the Romans
do - and everyone will find without searching. In addition to that, I added
an empty script file called npc.lua in campaing/scripts - again, just as 
everyone does. The record_npc.xml looks like this:

```html
<?xml version="1.0" encoding="iso-8859-1"?>
<root>
    <windowclass name="npc">
        <frame>recordsheet</frame>
        <placement>
            <size width="350" height="300" />
        </placement>
        <sizelimits>
            <minimum width="350" height="300" />
            <dynamic />
        </sizelimits>
        <nodelete />
        <script file="campaign/scripts/npc.lua" />
        <nodelete />
        <sheetdata>
            <resize_recordsheet />
            <close_recordsheet />
        </sheetdata>
    </windowclass>
</root>
```

The window class with name "npc" replaces the existing one in CoreRPG. The
frame tag gives the sheet a background (otherwise it would really be invisible -
I tried). Placement & sizelimits determine the initial size and how close (or
large) it can be resized. Dynamic tells you that it can be resized. I copied
nodelet from the source - but I don't know what it means. The documentation found
here: [https://www.fantasygrounds.com/refdoc/windowclass.xcp](https://www.fantasygrounds.com/refdoc/windowclass.xcp)
is not helpful. The sheetdata tag contains the content - in my case nothing 
except the marker to have a close button and something called resize_recordsheet
- but I haven't found documentation about it yet.