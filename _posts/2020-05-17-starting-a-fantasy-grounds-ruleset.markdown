---
layout: post
title:  "Starting a Fantasy Grounds Ruleset"
date:   2020-05-17 16:50:48 +0200
categories: fantasygrounds
---
# Creating a Fantasy Grounds Ruleset (Part 1/n)

Starting a ruleset isn't hard - the problem is to find the information you require
and to find out where / what to do.

According to https://www.fantasygrounds.com/modguide/introduction.xcp you only
need a base.xml file in the right directory. So I started with
`%APPDATA%\Fantasy Grounds\rulesets\CP2020` because I want to create a CP2020
ruleset and added a base.xml file

```html
<?xml version="1.0" encoding="iso-8859-1"?>
<root version="3.0" release="0.1" logo="graphics\Cyberpunk-2020-Logo.jpg">
	<!-- Attributes -->
	<description>
		<text>
        Cyberpunk 2020 - A rulset for the rpg by R. Talsorian Games Inc
        </text>
		<author>Kurald</author>
	</description>
    <importinfo>
        <acceptfrom ruleset="CoreRPG" />
    </importinfo>
</root>
```

The first line is simply the encoding. The second line determines for which version
of Fantasy Grounds the ruleset is applicable. The logo points to a graphic that
will be displayed in the rule-set selector. The text in the description is the
text displayed in the box below when the ruleset is selected. I haven't found out
how to format the description. HTML tags, \n and such don't work. Author is your
name. I think the importinfo describes which ruleset may provide extensions.

After starting a new Campaign with this ruleset, everything was black. I wasn't
even able to close Fantasy Grounds, so obviously this isn't enough.

Then I modified my base.xml by adding the following after the description:

```HTML
    <importinfo>
        <acceptfrom ruleset="CoreRPG" />
    </importinfo>
```

Once that was done, the ruleset displayed itself as a clone from the CoreRPG
ruleset. A good place to get started. The next step would be adapting / overwriting
the character sheet.
