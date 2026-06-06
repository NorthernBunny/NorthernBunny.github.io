---
layout: post
title: "Fun with mBot2"
categories: [tech]
---

Borrowed an mBot2 from the local library :) Cool little robot kit from Makeblock. 

![mbot2_photos](/assets/img/posts/mbot2.jpg)

You can go to [ide.mblock.cc](https://ide.mblock.cc/), connect live to the mBot2 via Bluetooth, and enter a simple program using drag-and-drop block-based programming or Python. Click on the "when CyberPi starts up" block and it will say hi, switch the LED display to the colors shown, and move forward. 

![mblock_ide](/assets/img/posts/mblock_ide.jpg)

### Troubleshooting a Drifting Robot

I ran into a strange hardware issue where the robot would not drive straight—it consistently drifted to the right because one wheel was spinning faster than the other. Before giving up, I tried a few troubleshooting steps:

* Flipped the EM1 and EM2 motor connectors.
* Updated the firmware.
* Reset the system to factory defaults.
* Attempted to use specific encoder motor commands to manually adjust the RPMs to compensate.

Unfortunately, nothing fixed the drift. I took it back to the library tech, who validated that it was behaving strangely and swapped it for another unit. The replacement is working normally!

### Getting Into AI: Image Recognition & Cloud Messaging

With a fully functioning robot, I decided to push the software a bit further and experiment with its AI and cloud capabilities. I wanted to see if I could use my MacBook's camera to recognize a color, classify an emotion, and have the mBot2 react accordingly. 

First, I jumped into the Stage/Sprite side of mBlock and loaded up the necessary extensions: **Machine Learning**, **Cognitive Services**, **User Cloud Message**, and **Teachable Machine**.

![mblock_ide](/assets/img/posts/makeblock_ide_image_sprite_extensions.jpg)

Using the Teachable Machine extension, I trained a quick vision model right in the browser to associate different colored cards with specific emotions: yellow for happy, red for angry, and blue for sad.

![mblock_ide](/assets/img/posts/makeblock_ide_image_model_training.jpg)

With the model trained, I entered a brief script for the Stage Sprite. It uses my MacBook’s webcam to continuously scan for colors. Once a color is recognized, it fires the corresponding emotional message ("happy", "angry", or "sad") up to the Makeblock cloud using MQTT cloud messaging.

![mblock_ide](/assets/img/posts/makeblock_ide_image_sprites.jpg)

Next, I needed to configure the mBot2 to listen for those messages. On the Device side, I added the **AI Emoji Workshop** and **AI Emoji** extensions.

![mblock_ide](/assets/img/posts/makeblock_ide_image_device_extensions.jpg)

I used the matrix editor to design custom pixel-art emojis matching the three target emotions.

![mblock_ide](/assets/img/posts/makeblock_ide_image_emoji.jpg)

Finally, I coded the CyberPi to connect to my home Wi-Fi on boot. It says hello, subscribes to the cloud message topic, and waits. The moment I hold a color up to my laptop camera, the robot instantly plays a matching sound effect and flashes the corresponding custom emoji on its display.

![mblock_ide](/assets/img/posts/makeblock_ide_image_devices.jpg)

The whole pipeline worked flawlessly. It is honestly pretty incredible how much advanced tech—computer vision, cloud IoT messaging, and hardware control—you can string together with such minimal effort using these tools!
