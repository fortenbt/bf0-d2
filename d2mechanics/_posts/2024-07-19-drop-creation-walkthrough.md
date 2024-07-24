---
layout: post
title: "A Full Walkthrough of the Creation of a Diablo II Item Drop"
date: 2024-07-19 12:00:00
description: >
  Detailing Each Step of Drop Creation, Including Magic Find Effects,
  Quality Assignment, and Item and Item Affix Property Selection
image: /images/drop-rng/chest.png
categories: d2mechanics
---

{% capture d2title %}Diablo&nbsp;II{% endcapture %}

{% capture objcontrolseed %}<span style="color:green"><b>ObjectControl Seed</b></span>{% endcapture %}

{% capture drlgseed %}<span style="color:green"><b>DRLG Seed</b></span>{% endcapture %}

{% capture gameseed %}<span style="color:green"><b>Game Seed</b></span>{% endcapture %}

{% capture unitseed %}<span style="color:green"><b>{{ unitseed }}</b></span>{% endcapture %}

{% capture itemdataseed %}<span style="color:green"><b>``ItemData Seed``</b></span>{% endcapture %}


# Background

The purpose of this post is to take the deepest of dives into the entire process
of item creation due to a killed monster or clicked object (chest, corpse, etc.)
dropping that item. This will cover all of the code that decides what the end
item will be and all of its affixes and stats, but will not cover the differences
in magic/rare chances for a super chest, for example. It will go over the
calculations that use a player's magic find and show why simply having more magic
find won't necessarily produce a magic item rather than a normal item for a
specific drop case. It will also necessarily cover all the text files in
the {{ d2title }} resource [MPQ files](https://web.archive.org/web/20120222093346/http://wiki.devklog.net/index.php?title=The_MoPaQ_Archive_Format)
that control overall drop chances for specific uniques and define all of the
[Treasure Classes](https://d2mods.info/forum/kb/viewarticle?a=410) that govern
drop rates.

Note that the rest of this first "Background" section is a lot of the same
across these {{ d2title }} mechanics posts in order to provide the uninitiated
with some context behind where this information is coming from as well as where
to go to read up on all the great work that has already been done in the past
by other groups. If you've already read any of my [other posts]({{ site.baseurl
}}/d2mechanics), feel free to skip down to
the [Introduction section](#introduction).

## Changes

- **2024-07-19**: The first post of this article.

## Version

The following information was derived from the project [D2MOO](
https://github.com/ThePhrozenKeep/D2MOO), which has largely annotated most of the
decompiled {{ d2title }} binaries. This project is based on {{ d2title }} version
1.10f, and thus may not be 100% accurate for D2R or the latest version of the
original D2 Lord of Destruction, 1.14d. However, according to users more familiar
with the differences, the main drop mechanics surrounding the RNG have not changed.

The RNG involved in drop mechanics of **online** {{ d2title }} games is
currently unknown. A fairly reliable source who worked at Blizzard
has mentioned that the game’s server code was patched to use Microsoft’s
cryptographically secure `CryptGenRandom` function for some random 
numbers rather than the source’s built-in MWC RNG, but it is unknown
where exactly this change applies. Others have performed lengthy analysis
on Lower Kurast chest drops in online games to verify that the RNG
does not match the single player game’s. Therefore, this information may
only be relevant for single player games.

## Thanks

I’d like to thank all of those individuals (most of whom are from the
Phrozen Keep and blizzhackers) who have done the painstaking work of
reverse engineering and annotating the disassembly and decompilation of
{{ d2title }}. I don’t have enough information about who they are, so
I’m sure I’ll miss many of the early names, but Necrolis, Kingpin,
Mysterio, Pavke, Nefarius, Lectem, Jarulf, Beowulf, McGod, and
Dark_Mage- were all associated in one way or another at some point.

## Past Work

Some great work has been done on understanding the drop mechanics within
{{ d2title }}. However, the understanding and write ups on these mechanics
focused mainly on the high-level details of predicting drop chances and
understanding item and item property selection. Previous write ups that
focus on exact item drop chances, how magic find affects the equations
used to determine drop quality, and chest drop specifics can be found at:

- [https://diablo2.diablowiki.net/Item_Generation_Tutorial](
   https://diablo2.diablowiki.net/Item_Generation_Tutorial)
- [https://www.reddit.com/r/diablo2/comments/13u4tqv/how_does_magic_find_actually_work_an_advanced/](
   https://www.reddit.com/r/diablo2/comments/13u4tqv/how_does_magic_find_actually_work_an_advanced/)
- [https://www.purediablo.com/forums/threads/guide-regular-special-and-sparkly-chests.380/](
   https://www.purediablo.com/forums/threads/guide-regular-special-and-sparkly-chests.380/)

This is not to say that this knowledge is new. Plenty of groups
throughout the years have understood {{ d2title }}’s internals. When the
server code was identical to the reverse engineered code back in the
day, it was an even more guarded secret on how the internals of RNG
worked so that private groups could exploit it online. The information
here may not be that interesting for general players given that
exploiting it requires the use of a set seed, which is typically viewed
as “cheating” in any competitive environment. If you’re using a seed in
single player specifically to get a high rune or a godly item or
property, why not just use the D2 LoD Hero Editor[^1] or the D2R Hero
Editor[^2]?

[^1]: https://www.moddb.com/games/diablo-2-lod/downloads/hero-editor-v-104
[^2]: https://d2runewizard.com/hero-editor

## Correction and Contact

If you are more familiar than I am with any of the concepts described in
this document and would like to help my understanding of them, or if you
notice something that is technically incorrect, feel free to contact me
at any of the social media sites listed on this site's main page. I would
be happy to make corrections and credit you.


# Introduction

A dropped item in {{ d2title }} is represented in the game's code as a
structure containing all the information required by the server to track
that item as an individual item unique from all other items. It contains
the item's location coordinates in the world, the item's level, a unique
identifier, the quality level, and a host of other metadata about the item.
This deep dive will list all the data about a dropped item and show exactly
how that data was created.

The drop we'll be exploring is one from a normal chest. While the normal
chest object is just one of many places from which an item can be created,
its codepath is very straightforward for beginning our walkthrough.

## ``Object`` Vs. ``Item`` Vs. ``Unit``

The terminology used above was carefully chosen to be accurate to and match
that used in the D2MOO codebase. "Object" generally refers to those objects
placed around the rooms and map areas in {{ d2title }}, while "Item" refers
to anything that can be dropped by a monster or object. Examples of Objects
include barrels, armor stands, rogue corpses, crates, wells, urns, etc.

In the D2MOO codebase, both *Objects* and *Items* are "Units", where the
Unit structure contains a field named ``UnitType`` that specifies if that
Unit is of type ``OBJECT``, ``ITEM``, ``MONSTER``, or a few other various
types. Besides the **type**, the Unit structure for the ``ITEM`` type keeps
track of whether that item is in storage (inventory, cube, stash, etc.),
equipped, in the belt rows, on the ground, under the cursor, currently
being dropped, or socketed within another item. It holds what type of item
it is (shield, armor, gold, weapon, potion, ring, et al) and the act from
which the item was dropped. The Unit structure also stores the {{ unitseed }}
and a pointer to a separate ``ItemData`` structure. The ``ItemData`` structure
holds the Item's quality, the {{ itemdataseed }}, and flags specifying if the
item is identified, ethereal, socketed, personalized, etc. It holds a
timestamp for when the item was picked up from the ground, the item's level,
all the affixes applied to the item, and a host of others we'll describe in
detail at some point within this post.

As you can see, there's a lot of stuff to keep track of for every object
and every item drop.

## Psuedo Random Number Generation

I discussed the random number generation that {{ d2title }} uses in detail
in my [RNG Within Diablo II Controlling Drop Mechanics]({{ site.baseurl
}}{% post_url d2mechanics/2024-05-15-drop-rng %}) post, but this post
wouldn't be complete without at least touching on it a bit here.

### Video Game Random Numbers

All video games use psuedo random number generation (PRNG). True random
number generation is too time consuming and/or requires special hardware access
to use, so games simply use an algorithm that produces a stream of numbers
that are uniformly distributed in order to provide random chance. Typically,
PRNG algorithms require a "first value"---called a "seed"---that selects
a single stream of many random numbers. The algorithm used determines how
long the stream of random numbers is before it begins to repeat. How many
different streams there are is also determined by the algorithm and the size
of the seed.

{{ d2title }} uses a 32-bit seed and a multiply-with-carry pseudo random
number generator algorithm. The algorithm used produces streams of uniformly
distributed bits of length $$2^{64}$$