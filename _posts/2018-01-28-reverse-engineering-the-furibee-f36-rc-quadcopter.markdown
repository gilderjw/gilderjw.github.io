---
title: Reverse Engineering the FuriBee F36 RC Quadcopter
layout: post
date: 2018-01-28
categories: hardware
---

A couple months ago, I bought a FuriBee F36 RC Quadcopter to play around with. After a bit, I began to wonder how it works. In this writeup, I will discuss the beginning of my journey to get a firmware dump of this neat little drone.

![drone](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0039.JPG)

## Taking it apart

I do not know much about hardware hacking, but I do know that the program used by this drone has to be on a chip that I can interact with. Hopefully, I can persuade this chip to dump its contents so I can see what is going on under the hood. I started by disassembling my drone to get a look at what I was dealing with.

![drone with top off](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0043.JPG)

Disassembling the drone was pretty easy and straightforward. Once the top was off, I could see a PCB with a couple IC's on it along with some wires going to the motors. Two screws and eight desoldered wires later, I had the PCB by itself:

| ![PCB with boxes](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0046_COLORED.JPG) | ![PCB bottom](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0044.JPG) |

There are two interesting things on this PCB. The large unmarked IC in the yellow box, and the smaller marked chip in the red box. Unfortunately, I did not have access to a nice camera with a macro lens, to I cannot capture an image with clear text. The text on the smaller, red chip is:

```
M540
259LA1
L1519
```

At this point, I went on a furious Google spree trying to find some hint to how I could get to the firmware. I did not have much look in my research, but I did get some ideas about what was going on. None of my research was conclusive, but the little I got is better than nothing.

Here is what I have concluded so far:

* The red chip is a gyroscope
  * A Google result said gyroscope, but did not show any numbers or images
  * A quadcopter does need a gyroscope to keep balanced in the air
  * The IC is in the middle of the PCB
* The yellow chip may be an ARM processor with memory on chip
  * FuriBee has a few other flight controllers that I was able to find and they were all ARM

These results are pretty lame, I did not get anything that I can really use to get to the firmware. As a last-ditch effort to get some information, I decided to lift the large IC off of the PCB. Hopefully, there would be some markings on the bottom that could lead be down the path that I was after. Besides, I needed to lift the chip off anyway to wire it up to a breadboard.

| ![PCB with chip lifted](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0053.JPG) | ![chip removed from PCB](/assets/2018-01-28-reverse-engineering-the-furibee-f36-rc-quadcopter/IMG_0052.JPG) |

This was the first time I had used a desoldering wick. Because of this, I no longer have the option to solder the chip back to the PCB and have a working drone. The next time that I have to use a wick will turn out much better based on the quality of the second side that I did.

To my disappointment, there were no markings on the bottom of the chip. I didn't have my hopes too high, but it would have been nice to have a part number for the IC.

Now that this drone is dead, I will have to get another and attack the problem from a different angle.
