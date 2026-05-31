---
layout: post
title: "Installing a Wired IP Camera: Overcoming the DIY Learning Curve"
categories: [tech]
---

I have always wanted to try a wired IP camera. Having exclusively used wireless ones in the past, I frequently wondered if doing the extra grunt work to install a hardwired line would truly be worth the effort. 

I finally decided on the **Reolink RLC-811A PoE camera**. It delivers a crisp 4K stream and features a robust mobile app to keep things simple out of the box. Crucially for me, it also plays nice with open-source protocols, meaning I can directly integrate the raw stream into my home automation system later on. It also features an onboard microSD slot for local storage, allowing me to completely bypass paying a monthly subscription fee just to access my own footage in someone else's cloud. 

I managed to snag the Reolink on Amazon on sale for $140 CAD. Next up was local storage. After some digging, I picked up a **SanDisk High Endurance 256GB microSDXC card** for $80 CAD. Standard gaming or phone cards will quickly burn out under the relentless pressure of a 4K loop-recording surveillance stream. You need a dedicated endurance card with specialized firmware that can handle constant write cycles and survive the punishing seasonal swings of Canadian winters and summers.

One of the best features of this hardware is Power over Ethernet (PoE), which allows the camera to receive data and power over a single ethernet cable. I already had a small **TP-Link PoE switch** sitting in my home lab, so I put it to work for this project. The architecture is straightforward: connect the camera to the PoE switch, and then bridge the switch into your main network. 

During my research, I learned that standard interior ethernet cables will quickly crack and fail outdoors due to moisture and sunlight. To prevent this, I bought a **100-foot GearIT Cat6 Outdoor Pure Copper (OFC) cable**, which features a heavy-duty LLDPE waterproof jacket and excellent UV protection. I measured the physical run from my switch to the corner mounting spot at roughly 70 feet, so buying a pre-terminated 100-foot line gave me a perfect amount of slack to work with.

Before climbing any ladders, I plugged the camera directly into the TP-Link switch on my workbench using the 100-foot GearIT cable just to ensure the firmware booted and the network link was stable. 

With the bench test passed, it was time for the hardest part: getting the line outside. I thought I had a long masonry drill bit on hand, but right before boring into the wall, I realized it wasn't nearly deep enough to clear the exterior wall cavity. Time to pivot. Instead of tearing up the wall, I drilled a clean entry hole directly through the vinyl window frame trim. I ran the thick outdoor cable through, mounted the camera chassis, and sealed the entry point tightly using **GE Silicone II** waterproof sealant. 

To clean up the aesthetics, I am currently tucking the cable run beneath the vinyl siding track using **7mm Coaxial Nail Clips** to anchor it out of sight, and I am using clear **Command Outdoor Window Clips** to hold the wire perfectly flush inside the window frame. 

The camera is officially live, the 4K local playback timeline is working flawlessly, and I’ve already kicked off my very first weather time-lapse!

