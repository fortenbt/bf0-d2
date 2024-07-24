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

{% capture itemdataseed %}<span style="color:green"><b>ItemData Seed</b></span>{% endcapture %}


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
}}{% post_url d2mechanics/2024-05-15-drop-rng %}), feel free to skip down to
the [Introduction section](#introduction).

## Changes

- **2024-07-19**: The first post of this article.

## Version

The following information was derived from the project [D2MOO](https://github.com/ThePhrozenKeep/D2MOO), which has largely
annotated most of the decompiled {{ d2title }} binaries. This project
is based on {{ d2title }} version 1.10f, and thus may not be 100%
accurate for D2R or the latest version of the original D2 Lord of
Destruction, 1.14d. However, according to users more familiar with
the differences, the main drop mechanics surrounding the RNG have
not changed.

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

- [https://diablo2.diablowiki.net/Item_Generation_Tutorial](https://diablo2.diablowiki.net/Item_Generation_Tutorial)
- [https://www.reddit.com/r/diablo2/comments/13u4tqv/how_does_magic_find_actually_work_an_advanced/](https://www.reddit.com/r/diablo2/comments/13u4tqv/how_does_magic_find_actually_work_an_advanced/)
- [https://www.purediablo.com/forums/threads/guide-regular-special-and-sparkly-chests.380/](https://www.purediablo.com/forums/threads/guide-regular-special-and-sparkly-chests.380/)

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

