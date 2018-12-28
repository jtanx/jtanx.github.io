---
layout: post
title: Flag drawing with Turtle
---

![flag-of-wales]({{ "/" | relative_url }}public/img/2018-12-28-flag-of-wales.png)

Yep, drawing the Flag of Wales using Turtle :P

<!--more-->

One of the first units I ever took at UWA was [CITS1200](http://teaching.csse.uwa.edu.au/units/CITS1200) - Java programming. There was this fairly basic [lab task](http://teaching.csse.uwa.edu.au/units/CITS1200/Laboratories/FlagDrawer/) involving drawing flags using a limited set of drawing commands. If I recall correctly, you were only allowed to use `drawLine`, so you had to implement your own methods to draw more complex shapes like circles or arcs. There were a set of flags in order of increasing difficulty, starting at effectively drawing rectangles, and working up to draw more complex shapes, with more complex flags earning you more points. There was also a set of flags categorised as 'impossible' to do - one of which was the Flag of Wales.

In 2012, I took the (then) newly offered [CITS1401](http://web.archive.org/web/20160309105033/http://teaching.csse.uwa.edu.au/units/CITS1401/) a.k.a. Python programming. This was a fun unit, mostly because it was ridiculously easy. Ironically, it was one of the more useful units, purely because of how much I still use Python. Needless to say, that *same* flag drawing exercise came up, albeit this time in Python and using the Turtle API. Unfortunately the unit contents have changed since I took it, so I can't find the reference. Unlike CITS1200 the labs weren't marked, but there was a prize on offer for the best flag drawer (or something like that).

I don't think I bothered much with the exercise itself because no marks were on the line, but I did have an idea - what if we could write a parser to interpret an [SVG](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics) as a bunch of Turtle drawing commands? That way, we could draw even the previously 'impossible' flags.

Digging through the files I had at the time, [this gist](https://gist.github.com/jtanx/844e60ddf43bb865b47786b373b58a01#file-svgturtle-py) is what I wrote. This generated the following flag:

![flag-of-hk-1]({{ "/" | relative_url }}public/img/2018-12-28-flag-of-hk-attempt1.png)

From my vague recollection, the attempt was to take a simplified version of the [Flag of Hong Kong](https://upload.wikimedia.org/wikipedia/commons/5/5b/Flag_of_Hong_Kong.svg) and to draw one of the petals. As clearly shown, there was something very wrong with how this was implemented - specifically in how the curveto instructions were interpreted. This was where I left it, until last year.

Last Christmas (in 2017), I fleshed out the idea a lot - most notably, getting it to a point where it more or less *worked*. I would have written this up last year, but never got the chance nor motivation to do so until now - I've forgotten (again) most of what I did to get it working, but what comes to mind are:

* Transforming the coordinates to be (0, 0) in the top left corner
* Fixing the implementation of cubic Bézier curveto commands (quadratic curves are left unimplemented)
* Implementing naïve support for xlinks (just duplicating content wherever they're encountered)
* Implementing support for transformations
* Implementing support for reading and applying styling information

It still struggles on rendering a few flags, and there were a couple of issues that I never fixed:

* Filling on self-intersection is probably broken
* Some curve commands are still unimplemented
* Units/widths are dodgy
* At the time, I spent quite a bit of time trying to remove all window decorations (and failing). Somehow it's resolved itself when I reran it this year
* Getting the turtle to have (0,0) *exactly* in the corner was hard; it kept being offset by the width of the marker. I fudged it by setting a negative offset on the world coordinates

The code for all of this can be found [here](http://github.com/jtanx/turtlesvg).

## Results

#### Flag of Libya (1977-2011)
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/01.mp4" title="Flag of Libya (1977-2011)"></video>

#### Flag of Ukraine
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/02.mp4" title="Flag of Ukraine"></video>

#### Flag of France
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/03.mp4" title="Flag of France"></video>

#### Flag of Thailand
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/04.mp4" title="Flag of Thailand"></video>

#### Flag of The Gambia
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/05.mp4" title="Flag of The Gambia"></video>

This one is slightly incorrect - the colour is off in the middle section. Probably something to do with misinterpreting the default style.

#### Flag of the UAE
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/06.mp4" title="Flag of the UAE"></video>

#### Flag of Finland
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/07.mp4" title="Flag of Finland"></video>

#### Flag of Switzerland
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/08.mp4" title="Flag of Switzerland"></video>

#### Flag of the Republic of Congo
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/09.mp4" title="Flag of the Republic of Congo"></video>

#### Flag of Seychelles
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/10.mp4" title="Flag of Seychelles"></video>

#### Flag of Scotland
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/11.mp4" title="Flag of Scotland"></video>

This one took a long time at the start (cut from here) before it started drawing something visible.

#### Flag of Qatar
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/12.mp4" title="Flag of Qatar"></video>

#### Flag of South Africa
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/13.mp4" title="Flag of South Africa"></video>

This one is inaccurate - the line cap is incorrect (circular instead of square). I don't think the line cap is configurable in Turtle, but there are probably ways around it. Or the SVG could be changed from using a thick stroke to a path.


#### Flag of Japan
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/14.mp4" title="Flag of Japan"></video>

This one required modifying the SVG to avoid the use of elliptical arcs.

#### Flag of Greenland
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/15.mp4" title="Flag of Greenland"></video>

Similar to the Flag of Japan, this also required modifying to avoid elliptical arcs.

#### Flag of the Western European Union
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/16.mp4" title="Flag of the Western European Union"></video>

A few modifications required on this one - avoiding elliptical arcs, plus because fill rules aren't supported, an additional path had to be added to 'hollow out' the `O`.

#### Flag of South Korea
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/17.mp4" title="Flag of South Korea"></video>

Minor modification to avoid elliptical arcs.

#### Flag of Algeria
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/18.mp4" title="Flag of Algeria"></video>

Ditto

#### Flag of Antigua and Barbuda
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/19.mp4" title="Flag of Antigua and Barbuda"></video>

No modifications required here!

#### Flag of Hong Kong
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/20.mp4" title="Flag of Hong Kong"></video>

This one worked fine too.

#### Flag of Albania
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/21.mp4" title="Flag of Albania"></video>

Similar to the Flag of the Western European Union, additional curves had to be explicitly added for the eye segments.

#### Flag of Wales
<video onmouseover="this.play()" onmouseout="this.pause()" src="{{ '/' | relative_url }}public/img/2018-12-28-flags/22.mp4" title="Flag of Wales"></video>

This one was fine!