---
layout: post
title: "Fun with mBot2"
categories: [tech]
---

 Borrowed an mBot2 from the local library :) Cool little robot kit from Makeblock. 

 
![mbot2_photos](/assets/img/posts/mbot2.jpg)


  You can go to https://ide.mblock.cc/ , connect live to the mBot2 via bluetooth, and enter a simple program using drag and drop block-based programming or with python. Click on the "when CyberPi starts up" and it will say hi, switch the led display to the colors shown, and move forwards. 

![mblock_ide](/assets/img/posts/mblock_ide.jpg)

 Had a strange issue where the robot would not go straight. Would always turn to the right. One of the wheels was turning more than the other. I tried flipping the 2 EM1 and EM2 motor connectors, tried updating the firmware, resetting the system to factory, tried to use a specific enable motor cmds to make one move at a higer rpm than the other, but could not get it to work. Took it back to the tech at the library, he validated that it was behaving strangely. So swapped for another one, and it is working normally. 

 
