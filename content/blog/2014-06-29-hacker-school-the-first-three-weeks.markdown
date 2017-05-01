---
title: "Hacker School: The First Three Weeks"
date: 2014-06-29
comments: true
categories: hacker_school
---

For three weeks now, I've been at
[Hacker School](http://hackerschool.com). Hacker School is hard to
describe, they call themselves a "writer's retreat for programmers."
Personally I prefer "programmer summer camp," mostly because I have no
idea what writer's retreats are like. Basically it's a collection of
people working in a self-directed way to improve their skills as
programmers. They accept people of all skill levels, as long as you
have programmed before.  I've heard "but Max, you already know Ruby on
Rails" from more then one person when telling them I'm going to Hacker
School, so I think I should let you know that Hacker School doesn't
have a curriculum, isn't a bootcamp, and I'm trying to write as little
Ruby as I can get away while here.

I have a lot to say about what it feels like to go to a Montessori
school for adults, but I'll save that for another blog post. This is
going to be a inventory of what I've been working and thinking about
while here, and some of my goals. One of my goals coming in was to
blog a lot. I haven't done it, but I hope this will the first of n
Hacker School blog posts (for sufficiently large n).

## What I'm Doing

I've started a lot of projects while here, but they fall into two main categories. Courses and Projects.

### Courses
* I started [Write Yourself a Scheme in 48 Hours](https://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours) with another Hacker Schooler [here](https://github.com/mveytsman/scheme48). I'm finding that I'm not getting a lot of out of it --- I'm running into a lot of issues that would be solved with a more traditional Haskell resource, so to that end,
* I started Yorgey's Haskell [course](http://www.cis.upenn.edu/~cis194/) [here](https://github.com/mveytsman/scheme48)
* I started working through this [exploitation](http://www.trailofbits.com/training/assured_exploitation/) course.
* I signed up for Dan Boneh's [crypto](https://www.coursera.org/course/crypto) course which starts tomorrow.

This above is very ambitious, and I fully expect to not finish most of these. I'm pretty sure I'm not going to come back to the Haskell Scheme interpreter course. I would like to finish the Haskell and exploitation class. I've started the crypto class many times before already, so there's that.

### Projects
* To prove to myself just how much better I understand Ruby then Haskell, I wrote a very simple [Scheme interpreter](https://github.com/mveytsman/rubbyskeme) based on Norvig's [lispy](http://norvig.com/lispy.html)
* My main project so far is implementing an [emulator](https://github.com/mveytsman/emm-ess-pee) for the [MSP430](https://en.wikipedia.org/wiki/TI_MSP430) in clojure.

### Emulator
The emulator project has been taking up most of my time here, and it's worth talking about it more. The MSP430 is the chipset
used in the [MicroCorruption](https://microcorruption.com/) hacking
game. Working on the emulator has helped me get better at reversing the MIPS assembly used by the processor, and I finally got to the last [level](https://microcorruption.com/profile/96) after a 5 month hiatus.

I have two directions I want to take this emulator in. First of all, I have started reading [papers](https://www.usenix.org/system/files/conference/woot12/woot12-final26.pdf) about using [SMT solvers](https://en.wikipedia.org/wiki/Satisfiability_Modulo_Theories) in binary reverse engineering and exploitation. I would like to develop some heuristics for solving the microcorruption challenges automatically. Since I already have an emulator, the next step would be to emit SMT formulas based on execution traces. I don't actually understand how to solve the problem deeper then the above sentence, so the real next step is to read more papers.

The other direction I have been thinking about is to port my emulator from Clojure into Clojurescript and build a web-based UI around it. The microcorruption game has a web based emulator, but the emulation actually happens on a sever and the browser provides only the UI. I have this vague idea of using colors to map display bytes in memory and show a buffer overflow as mixing colors. That's about all I got though, but it's a good excuse to learn [Om](https://github.com/swannodette/om).

## Future posts
Next time I'll explain how the biggest lesson I learned from writing a emulator in a functional programming style was about testing and talk about writing my first (!) clojure macros.


