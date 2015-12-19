---
layout: page
title: Projects
---

In case you are wondering what I was up to for the last 10 years, here is a summary of some project I've been working on. 

# Classify Minecraft Plugin

[Classify](http://dev.bukkit.org/bukkit-plugins/classify/) annotates the in-game list of online players with their current behavior (building, mining, exploring, fighting or idle). It uses a classifier I developed in my thesis about statistical player analysis. I made this plugin mainly to promote the HeapCraft project. 

# DiviningRod Minecraft Plugin

[DiviningRod](http://dev.bukkit.org/bukkit-plugins/diviningrod/) is an in-game navigation tool which makes it fun and easy to find places or other players. The findable targets can be defined by a remote server using heuristics from recorded gameplay (e.g. to most violent player). I originally developed the plugin to affect player behavior for a scientific study. Unfortunately, the adoption rate was too low and I was forced to fall back other, less cool methods. 

# HeapCraft

In my master thesis, I analyzed player behavior in Minecraft. To get data from players, I wrote a logging plugin called [Epilog](http://heapcraft.net/?p=epilog-manual) which records gameplay and sends it to our server. The project involved a lot of advertising and convincing server administrators to participate. In the end, the work paid off and we now have a very interesting and evergrowing dataset. The approach was kind of novel and I keep getting inquiries by various researchers. More information is available on the [website](http://heapcraft.net/) I built for the project. 

# Multiplayer Sheepherding Game for Xbox

In 2013, I attended the game programming lab lecture where we go through a whole game development cycle in one semester. The [end result](https://www.youtube.com/watch?v=eh2yhc_WBUY) doesn't look like much, but it's very fun to play! Consider that our team had only two members (also attending other lectures) and we spent a huge part of the semester coming up with a concept and building physical prototypes :-)

# RFID Enabled Beer Vending Machine

One year our student association made too much profit. We decided to balance our budget by giving every member a free beer per day. To simplify the beer delivery, we bought an old beverage vending machine from eBay. I then added some circuitry identify students by their RFID enabled student ID card and change the price to free temporarily if the student was eligible. Logging the beer consumption resulted in some interesting statistics. And of course I hardcoded my card to get unlimited free beer. 

# Bicycle Controller for Mario Kart

For a bicycle themed party organized by [[project21]](http://www.project21.ch/), I modified two bicycles so they can be used to play Super Mario Kart on a SNES emulator. I used an ATmega8 microcontroller to detect handlebar position (potentiometer), brake usage and wheel speed (by counting zero crossings from the dynamo output) and translated it to PS/2 keyboard signals. For the analog inputs, I simulated multiple button presses with PWM. 

# Multiplayer Snake for BIRD

I wrote a multiplayer snake game for the interactive dance floor created by [bastli](http://bastli.ethz.ch/). The [video](https://www.youtube.com/watch?v=um1bMXSOXw8) shows an early prototype which was accidentally still installed on the device when it was used for a party. Turns out drunk people provide very valuable usability feedback!

# Picture Converter for Rotary Display

Since I had some background in computer vision, I was asked by the nice guys over at [bastli](http://bastli.ethz.ch/) to make an image converter to create bitmaps for their [display from hell](http://hackaday.com/2008/11/22/stupidly-huge-pov-display/). The trick to get crisp images is to map the polar coordinate pixels from the display to the source image instead of the other way around. 

# Exam Preparation Course Registration Tool

[AMIV](https://www.amiv.ethz.ch/), the Academic Mechanical and Electrical Engineering Association of ETH Zurich, holds exam preparation courses every semester. The courses are very popular and the supply is limited, which led to the registration page being overloaded regularly when the registration period started. I created a new registration web application which not only could handle the enormous stress, but also made the registration process more convenient and could handle waiting lists where students get notified when new spots became available. A main part of increasing the number of simultaneous user was to discourage and reduce the benefit of frequent page reloads by smart UI design. 

# Event Registration Tool

I created an online event registration tool for my student association. The main feature was a custom form markup language which allowed administrators to collect additional information during the registration process. 

# CRM for Student Job Fair

While being a member of the organization committee of the [AMIV Kontakt](http://www.kontakt.amiv.ethz.ch/), I wrote a custom customer relationship management web app. The app replaced a shared spreadsheet. Our recruiters would log all communication they had with potential exhibitors which allowed them to easily pick up where others left off. 

# CMS and InDesign Automation for Student Newspaper

To streamline my process while I was working as the layouter of my student association's biweekly [magazine](https://www.blitz.ethz.ch/), I developed a web interface where authors and editors could work on articles. It replaced the old system of mostly emailing word documents. Articles (including images, descriptions and author information) could be imported from InDesign and were automatically put into a layout template. Last minute changes could also be merged into the final layout. This saved me hours of work on every issue. 