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

### Identifying Images

Followed some instructions to:

* add sprite extensions, Machine Learning, Cognitive Services, User Cloud Message and Teachable Machine

![mblock_ide](/assets/img/posts/makeblock_ide_image_sprite_extensions.jpg)

* train a model to recognize yellow as happy, red as angry, and blue as sad

![mblock_ide](/assets/img/posts/makeblock_ide_image_model_training.jpg)

* use makeblock to recognize colors using the camera on my macbook and depending on recognized color to send the corelating message, happy, angry or sad to the cloud ?

![mblock_ide](/assets/img/posts/makeblock_ide_image_sprites.jpg)

* added device extensions, AI Emoji Workshop and AI Emoji

![mblock_ide](/assets/img/posts/makeblock_ide_image_device_extensions.jpg)

* created emojis for happy, sad, angry

![mblock_ide](/assets/img/posts/makeblock_ide_image_emoji.jpg)

* mbot2 connects to wifi, says hi, and waits for msg, and then plays appropriate happy, sad, angry sound and displays appropriate emoji

![mblock_ide](/assets/img/posts/makeblock_ide_image_devices.jpg)


Worked well. Pretty incredible that this is possible with such minimal effort !
