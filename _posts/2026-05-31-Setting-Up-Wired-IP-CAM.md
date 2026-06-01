---
layout: post
title: "Installing a Wired Outdoor POE IP Camera: Overcoming the DIY Learning Curve"
categories: [tech]
---

I have often wondered if doing the extra work to install a wired outdoor IP camera would offer better results than a simpler wireless indoor one.

First I needed to research an affordable outdoor wired IP camera. I wanted something at a reasonable price, with good weatherproofing, a good operating temperature range, 4K, good optics, optical zoom, a microSD card slot, a decent app so it would work right away, but also the ability to integrate with open-source systems like Home Assistant. I finally decided on the **Reolink RLC-811A PoE camera**. It features 5x optical zoom, 8MP 4K resolution, person/vehicle/animal detection, time-lapse recording, color night vision, a microSD slot (max 512GB), an IP67 rating, a -10°C to 55°C (14°F to 131°F) temperature range, 802.3af PoE, and a 10/100Mbps single RJ45 Ethernet port. 
Found it on Amazon on sale for $140 CAD. 

Next, I needed a microSD card so that I could keep some footage locally. After some research, I found that a standard microSD card could fail under the constant writes from a 4K loop-recording video stream, as well as from harsh outdoor conditions like heat, cold, temperature fluctuations, and UV rays. Apparently there are specialized microSD cards for outdoor cams. Based on price and size, I settled on the **SanDisk High Endurance 256GB microSDXC card** for $79 CAD at Best Buy. This one should handle about 3 days of capture. 

One of the best features of this hardware is Power over Ethernet (PoE), which allows the camera to receive data and power over a single Ethernet cable. I already had a small **TP-Link PoE switch** sitting in my home lab, so I put it to work for this project. The architecture is straightforward: connect the camera to the PoE switch, and then bridge the switch into your main network. 

During my research, I learned that standard interior Ethernet cables will quickly crack and fail outdoors due to moisture and sunlight. To prevent this, I bought a **100-foot GearIT Cat6 Outdoor Pure Copper (OFC) cable**, which features a heavy-duty LLDPE waterproof jacket and excellent UV protection. I measured the physical run from my switch to the corner mounting spot at roughly 70 feet, so buying a pre-terminated 100-foot line gave me a perfect amount of slack to work with.

Before installing, I plugged the camera directly into the TP-Link switch on a bench using the 100-foot GearIT cable just to ensure the firmware booted and the network link was stable. 

![reolink_testing_pre_install](/assets/img/posts/reolink_testing_preinstall.jpg)


With the bench test passed, it was time for the hardest part: getting the line outside. I thought I had a long masonry drill bit on hand, but right before boring into the wall, I realized it wasn't nearly deep enough to clear the exterior wall cavity. Instead of buying a longer drill bit, I decided to drill through a corner of a vinyl window which is right next to my PoE switch and main home switch. I drilled two holes right next to each other with the bit I had so the RJ45 end could pass through. I removed the plastic cover around the head of the Ethernet RJ45 end so that it was smaller and could run through the hole. Then I sealed the entry point using **GE Silicone II** waterproof sealant. 

![reolink_window_hole](/assets/img/posts/reolink_window_hole.jpg)

I screwed the Reolink into the deck under the soffit for some extra weather protection. 

![reolink_installed](/assets/img/posts/reolink_installed.jpg)

Next I connected the waterproof kit and plugged in the ethernet. Had to remove the plastic bit covering the rj-45 ethernet end to get it to fit. The ethernet cable is thicker than normal, so had a little issue with that, the fit was very tight. 

![reolink_waterprf_kit](/assets/img/posts/reolink_waterprf_kit.jpg)

![reolink_rm_eth_hat](/assets/img/posts/reolink_rm_eth_hat.jpg)

Next, I tucked the cable run beneath the vinyl siding track using **7mm Coaxial Nail Clips** to anchor it out of sight, and I am using clear **Command Outdoor Window Clips** to hold the wire perfectly flush inside the window frame. 

![reolink_cbl_mgmt](/assets/img/posts/reolink_cbl_mgmt.jpg)

![reolink_cbl_mgmt_2](/assets/img/posts/reolink_cbl_mgmt_2.jpg)

![reolink_cbl_mgmt_3](/assets/img/posts/reolink_cbl_mgmt_3.jpg)


This project turned out to be more work and cost than I expected at the beginning. I wasn't expecting to pay more for things like fancy outdoor Ethernet, a high-performance SD card, specialized clips, sealant, and potentially a long drill bit. At this point, the camera is officially live, the 4K local playback is working well, and I’ve already kicked off my very first time-lapse! The question is, after spending about $400, and executing substantially more effort than deploying a wireless indoor, or even a wireless outdoor camera, was it worth it?
