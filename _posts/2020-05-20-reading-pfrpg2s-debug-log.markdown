---
layout: post
title:  "Reading PFRPG2s Debug Log"
date:   2020-05-20 23:30:00 +0200
categories: fantasygrounds
---

# Reading PFRPG2s Debug Log

Today I looked a little bit into PFRPG2 to learn from it. I like the fact that
you can pick up text from different text-boxes and it'll automatically create
the proper roll for you. Attacks, damage - it doesn't matter. And it's such an
easy to use automation. No need to fill out a lot of different fields.

So I unpacked PFRPG2 first into a different folder. And then I started searching
the code. I was lucky once I searched for the word "parse". I found the
string_attackline.lua with its "onHoverUpdate" method that triggers the parsing.

I also noticed that the code contains a lot of debugging messages that I wasn't
aware of. They all started with "GlobalDebug" and then some command. After looking
into the global_debug.lua, I found those functions. global_debug.lua is loaded
in a script-tag somewhere and given the name GlobalDebug.

Inside global_debug.lua, there's a method "isDebugSet()" that checks the DB for
a variable. After searching in Google how I could set the variable using a
chat command (and failing), I thought - why not create the value myself by editing
the database.

```lua
function getDebugNode()
	local nodeDebug = DB.findNode("global_debug");
	
	if not nodeDebug then
		nodeDebug = DB.createNode("global_debug", "string");
	end
	
	-- Set public so player instance can access.
	if User.isHost() then
		nodeDebug.setPublic(true);
	end
	
	return nodeDebug;
end
```

This is the code to read the value. The function createNode is found here:
https://www.fantasygrounds.com/refdoc/DB.xcp. It simply creates a top-level tag
in the db.xml - if you set it to "on", debug messages are enabled.

```html
<global_debug type="string">on</global_debug>
```

To view these messages, you simply have to enter \console in chat and move the
window that pops up somewhere - ideally to a second monitor. If you now do an
action (e.g. hover over a damage action), you'll immediately see the output as
given by the script.

That's it for today,

Regards

Tobias
