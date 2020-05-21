---
layout: post
title:  "Anatomy of an Extension"
date:   2020-05-19 23:30:48 +0200
categories: fantasygrounds
---
# Anatomy of an Extension

Today, while preparing an adventure, I stumbled on this extension:
[https://www.fantasygrounds.com/forums/showthread.php?50314-PF2CreatureParser-Extension]
(https://www.fantasygrounds.com/forums/showthread.php?50314-PF2CreatureParser-Extension).

It pics up stat-blocks from prepared adventure PDFs and allows parsing them, to
add a creature to the database in a fast way. First of all, it's an awsome extension.
Second - it's small enough to look into it and start understanding everything.

To look into it, you simply have to extract the contents - it's a zip-file,
although the extension is ext. It creates a few directories and files:

- campaign
  - scripts
    - npc_parse.lua
    - npc.lua
  - record_npc.xml
- graphics
  - tabs
    - tab_parse.png
  - graphics_tabs2.xml
- scripts
  - pf2creatureparser.lua
- strings
  - strings_pf2creatureparser.xml
- extension.xml

The extension.xml anounces the extension and the ruleset it's made for:

```html
<announcement text="Pathfinder 2E Creature Parser v1.2.3 for Fantasy Grounds" font="emotefont" />
<properties>
  <name>PF2CreatureParser</name>
  <version>1.2.3</version>
  <author>darrenan</author>
  <description>Pathfinder 2E Creature Stat Block Parser</description>
  <ruleset>
    <name>PFRPG2</name>
  </ruleset>
</properties>
```

followed by a base-tag that simply includes the other xml files using the includefile-tag
and the pf2creatureparser.lua using the script-tag.

```html
<base>
<!-- Custom script to add abilities to object tables -->
<script name="PF2CreatureParser" file="scripts/pf2creatureparser.lua" />

<!-- Graphics -->
<includefile source="graphics/graphics_tabs2.xml" />

<!-- Campaign Records -->
<includefile source="campaign/record_npc.xml" />

<!-- Strings -->
<includefile source="strings/strings_pf2creatureparser.xml" />

</base>
```

## strings_pf2creatureparser.xml

this file simply defines some strings by giving them names using the &lt;string&gt; tag:
[https://www.fantasygrounds.com/refdoc/string.xcp](https://www.fantasygrounds.com/refdoc/string.xcp)

## record_npc.xml

This is where the original npc-window is extended - using a "merge" attribute:

```html
<windowclass name="npc" merge="join">
```
[https://www.fantasygrounds.com/wiki/index.php/Developer_Guide_-_Rulesets_-_Layering](https://www.fantasygrounds.com/wiki/index.php/Developer_Guide_-_Rulesets_-_Layering)

It mainly adds the new parse tab. The parse-tab has 2 buttons. The button tag
(button_text_large) defines in a script tag what happens when the button is
pressed. In this case, they call functions from the npc_parse.lua.

## other scripts

I also noticed that the campaing folder with record_npc.xml contains a script
folder with 2 script files named after the window_classes. The script file
was associated with the windowclass using the script-tag. The nice convention
to name them as the window class helps finding what you need. Script files can
contain standard functions like onInit() or update(), but also self-defined
functions that you use to react on buttons (e.g. Parse()). The exact API seems
to be defined here: [https://www.fantasygrounds.com/refdoc/windowinstance.xcp](https://www.fantasygrounds.com/refdoc/windowinstance.xcp)

## Next Steps

My next step will be enhancing the extension to actually provide the pathfinder
symbols for action, reaction, ... - the ruleset seems to not convert them
automatically in some locations. And if I feel fancy, I'll try to update the
parsing code to handle some problems I've encounterd.