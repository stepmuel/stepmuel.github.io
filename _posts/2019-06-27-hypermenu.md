---
layout: post
title: "HyperMenu"
date: 2019-06-27 0:31:02 +0200
excerpt: "Introducing my latest project: HyperMenu, a JSON based format to describe interlinked menu structures."
tags: ios hypermenu
---

My passion for hierarchical menus started around the year 2000 while using mobile phones for the first time. I had a Nokia 3110, given to me by a classmate. Then an Ericsson T39. I used to systematically explore all menu points, like a maze, until I discovered every single feature of those fascinating little machines. I rediscovered that feeling when I got my first iPod as a birthday gift from my then girlfriend, ca. 2005. I was fascinated by its simplicity. Up, down, enter, exit. Intuitive, and yet so versatile.

In 2008, during my second year at university, I wrote an iPod inspired menu UI for a digital audio processing project, all in SHARC DSP assembly. It had a 4 line LCD display, a rotary encoder, and a D-pad, and it was glorious!

![dEQ Panel](/res/deq-panel.jpg)

Hierarchical menus introduce structure and order. I realized how they can guide users with a few simple questions, instead of presenting overloaded screens and endless lists. 

I started writing iOS apps in 2012, and quickly ended up creating libraries to make it easier to construct menus. It had to be easy, or developers like me might default back to using flat hierarchies without even considering submenus. Nobody wants to create two new files and a storyboard entry, just to figure out if this dropdown menu feels nicer as a separate screen. Or even worse, delete all the work in case it doesn't.

The idea for HyperMenu came to me during a hackathon in 2014. I shared the idea after a workshop to find projects, but nobody seemed interested. I made a few protocol drafts on my own afterwards, but couldn't find a solution I was really happy with. Until five years later, when I had a sudden inspiration: Objects can be requests. Requests that return said object.

That was it. The missing link to elevate a compact menu notation into a crafty web protocol. Screens didn't have to be independent requests, but they could, when it made sense. This added structural freedom similar to what my iOS libraries offer. After that, the rest of the specification basically wrote itself.

<div class="videoWrapper">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/WlLCZ32LJoM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

The HyperMenu specification is available [on GitHub](https://github.com/stepmuel/hypermenu). I also made an [iOS client](http://hypermenu.heap.ch/) which serves as reference implementation. I hope others will find it as useful as I did, and am looking forward to seeing more implementations and use cases in the future.
