---
title: "Implementing composite video output using the Pi Pico's PIO"
author: Alan Reed
date: 2021-07-14
description: "A record of my submission for the final project of the 2021 HackadayU course 'Raspberry Pi Pico and RP2040 - The Deep Dive', which involved the creation of a composite video peripheral for the Pi Pico and a replica of the game Pong."
tags: [raspberry pi pico, pio, software, embedded, hackadayu]
layout: layouts/post.njk
---

### Introduction 

This post documents the creation of my submission for the final project of the 2021 HackadayU course, [Raspberry Pi Pico and RP2040 - The Deep Dive](https://hackaday.io/course/178733-raspberry-pi-pico-and-rp2040-the-deep-dive), run by Uri Shaked. The task for this was to implement a peripheral capable of outputting a composite video signal using the Pi Pico's novel programmable I/O (PIO) subsystem, and then use this to recreate the classic video game Pong.

### What is composite video?
[Composite video](https://en.wikipedia.org/wiki/Composite_video) is an old analogue video standard for transmitting standard-definition colour video over a single wire (plus ground), typically using a yellow RCA/phono jack. It was commonly used by older consoles and tape players, and is still available as an input source on many TVs. Several variants exist with different resolutions and methods of colour encoding, the most common being NTSC, PAL and SECAM.  For this project we were asked to focus on the PAL standard.

To make the task feasible for the Pico, we will be ignoring the colour component of the signal and generating monochrome video data. As PAL actually only refers to the method of colour encoding, the video output we will generate is technically not PAL but instead closer to a [CCIR System B](https://en.wikipedia.org/wiki/CCIR_System_B) black and white television signal, which is what PAL was built upon.

CCIR System B divides the image on the TV screen, referred to as frame, into 575 horizontal lines. Each frame is subdivided into two fields: the even field contains all even-numbered lines; and the odd field contains all odd-numbered ones. The video signal sends the data corresponding to the even field for each frame first, followed by the odd field, and so the TV alternates between displaying all of the even lines, and all of the odd lines - this is called [interlacing](https://en.wikipedia.org/wiki/Interlaced_video), a method of reducing video flicker. CCIR System B runs at a rate of 50 fields per second, therefore it displays video at 25fps.

In addition to the 575 lines of video, between each field is a further 25 lines of non-video signal referred to as the vertical blanking interval.  This indicates the beginning of a field to the TV; the lengthy delay was necessary on old CRT TVs to allow the electron gun time to return to the top of the screen between fields. There are therefore a total of 625 lines per frame.

#### Signal format
At 50fps, with 625 lines per frame, each line must take 64μs to transmit. Composite video signals vary between 0 and 1V, idling at the "true black" level of 0.3V. "True white" is at 1V, and 0V is used only for synchronisation pulses. Standard lines containing video data ("active lines") are divided into the following four regions: 
 - Horizontal sync (0V, 4μs)
 - Back porch (0.3V, 6μs), 
 - Video data (0.3-1V, 52μs)
 - Front porch (0.3V, 2μs). 
 
 They say that a picture is worth a thousand words, so below is a diagram of the timing and levels that should make this much clearer:

{% figure "Diagram of horizontal line timing." %}
{% image  "hline.svg" "Horizontal line timing diagram" 1000 %}
{% endfigure %}

As mentioned earlier, between the active lines of each field lies the 25 lines that make up the vertical blanking interval.  This is divided up as follows:
 - 5 half lines containing short (2μs) vertical synchronisation pulses
 - 5 half lines containing long (27μs) vertical synchronisation pulses
 - Another 5 half lines containing short vertical synchronisation pulses
 - 17.5 blank video lines

The timing of the half-lines used for short and long synchronisation pulses can be seen in the diagrams below:

{% figure "Diagrams of short and long vertical synchronisation pulse timing." %}
{% image  "vline_short.svg" "Vertical sync short pulse timing diagram" 500 %}
{% image  "vline_long.svg" "Vertical sync long pulse timing diagram" 500 %}
{% endfigure %}

Now we know what a line of video looks like, and the format of the vertical blanking interval, we're almost ready to put everything together. There's a catch though: I said earlier that the 575 lines of video data is split into two fields, but this is an odd number! Splitting it equally gives 287.5 lines per field, which is exactly what the CCIR System B standard did. Even fields start with half a line of video after the vertical blanking interval, and odd fields end with half a line. If you find that confusing you're not alone, as I certainly did! There's a couple of pictures and an animated gif [here](http://web.archive.org/web/20161214221811/http://www.danalee.ca/ttt/analog_video.htm) that helped me to wrap my head around it. Below is a diagram of the timing for a complete CCIR System B frame, demonstrating how the vertical blanking period fits in with the video data.

{% figure "Diagram of CCIR System B even and odd field layouts with vertical blanking periods." %}
{% image  "vsync.svg" "Diagram of CCIR System B even and odd field layouts with vertical blanking periods." 1000 %}
{% endfigure %}

Note that the timings given within the text and diagrams above are not actually exact, I've rounded them to the nearest microsecond to make visualisation and implementation easier.  This appears to be fine in practice, both in my experience and from what I've read online, however anyone interested can find the precise timing information (and much more) in [International Telecommunication Union BT.1700](https://www.itu.int/rec/R-REC-BT.1700/en).

#### Digital video variant
With a half-line of video at the top and the bottom of the screen, the analogue specification described above doesn't lend itself to digital data transmission. Every digital video or image format that I'm aware of follows a rectangular structure containing only full rows of data, so it would be convenient if the display technology did the same. By rounding up the half-lines to full ones, we end up with 288 lines of video per field and 576 lines per frame - the [576i](https://en.wikipedia.org/wiki/576i) format. 

I haven't been able to find an official document covering the timing of 576i video, however from reading online it seems that the standard approach taken by other DIY implementations is to use a modified vertical blanking period to make room for the extra line of video:
 - Even frames: 6 short syncs, 5 long syncs, 5 short syncs, 17 blank lines
 - Odd frames: 5 short syncs, 5 long syncs, 4 short syncs, 17 blank lines

Note that, unlike the original CCIR System B standard, this now means that the even and odd blanking periods are *no longer the same*. In the original standard, even/odd frames were instead indicated by whether the data started on a line boundary or not.  I've created a modified version of the previous diagram that includes the changes described above, shown below. Laid out like this, it is quite easy to see how the extra line of video fits into the data signal.

{% figure "576i video even and odd field layouts with vertical blanking periods." %}
{% image  "vsync_576.svg" "576i video even and odd field layouts with vertical blanking periods." 1000 %}
{% endfigure %}

The above diagram is probably the most important one on this page, as this is the video data transmission format that I will be implementing using the Pico's PIO. All of the diagrams in this section were created by me using [Waveme](https://waveme.weebly.com/).

#### Hardware interface

Before diving into the software, we need to be able to connect the Pico to a screen in order to try out our work!  As we've seen, compositve video signals are analogue - which poses a challenge as the Pico doesn't have any true analogue outputs.  If we restrict ourselves to binary monochrome video (*i.e.* just true black or true white), then we only need to be able to supply three voltage levels to the TV: 0V (for sync), 0.3V (for black) and 1V (for white).

As explained within the course materials [here](https://hackaday.io/project/180374-pi-pico-pal-tv-pong/log/194437-composite-signal-connecting-to-a-real-tv), this can be achieved most simply by using two pins and a simple voltage divider like so: 

{% figure "Schematic for digital composite video adapter." %}
{% image  "cvideo_schematic.png" "Schematic for digital compositve video adapter." %}
{% endfigure %}

Pulling both pins high supplies 1V, pulling both low supplies 0V, and pulling `sync` high with `data` low supplies roughly 0.3V as desired. Most TVs should be internally terminated with a 75ohm resistor as shown in the diagram, which can be verified with a multimeter.  If one isn't, then this resistor will have to be included in the adapter too.


### Software implementation
#### PIO
The Pi Pico's Programmable IO (PIO) is a unique feature of the RP2040 chip, which allows the implementation of custom peripherals not included in the silicon. The PIO modules each contain four state machines, any number of which can be used at once. These state machines run programs written using a succinct instruction set that contains just 9 discrete instructions. The instruction set needs to be succint too, because the total program memory shared between the four state machines is just 32 instructions long so efficiency is key! Programs for the PIO are usually written in PIO assembly, then compiled using the PIO assembler `pioasm` as detailed in the [Pi Pico SDK](https://datasheets.raspberrypi.org/pico/raspberry-pi-pico-c-sdk.pdf). Uri's [Wokwi simulator](https://wokwi.com/) is a particularly useful tool for PIO development, as unlike on the silicon it is possible to actually interact with the PIO subsystem via the debugger within the simulator.

For the task at hand, a PIO module will be responsible for driving the `data` and `sync` pins such that an accurate composite video signal is supplied to the TV. The independent nature of the PIO module is beneficial here, as the signal timing needs to be spot on to prevent graphical glitches on the screen.  Keeping the pin-driving separate from the main processor prevents wayward interrupts from causing delays that corrupt our graphics - an issue I had during early implementations of my peripheral, before I moved everything into the PIO state machines.

As suggested by Uri, I split my PIO implementation across two PIO state machines: one for handling the synchronisation pulses, and one for shipping out the actual video data. Before the state machines are started, I store the number of lines per field in the `sync` machine's OSR (output shift register) and the number of bits per line in the `data` machine's Y register.  These numbers are needed to reset counters within the state machines while they are running, and are too large to encode directly within the PIO memory, so I instead made use of the PIO's handy forced execution functionality to run a few PIO instructions directly from a PIO hardware register during initialisation of the PIO module.  This allowed me to load and store the necessary numbers before the PIO state machines started running.

Of the two PIO programs I created, the `data` machine is by far the smaller. It waits for an internal flag to be set, then shifts out the specified number of bits over the next 52μs to complete the current line. The number of bits sent by the data machine determines the horizontal resolution of the image; unlike the vertical resolution this isn't fixed by the video standard and can be chosen by the user.  My implementation uses 768 bits per line, to give square pixels when displayed in 4:3. The `sync` machine sets the internal flag used to signal the `data` machine at the end of the back porch during each active line. Importantly, I learned that the `data` machine needed to have as high a clock speed as possible in order for this synchronisation to work properly.  Otherwise, horizontal jittering was visible in the display due to variations in the time between the `data` and `sync` machine clock edges.

{% figure "Rectangles rendered  on the Wokwi simulator with horizontal jitter, and with jitter removed after increasing the data clock by a factor of 16." %}
{% image  "rect_jitter.png" "Rectangle with horizontal jitter" 350 %}
{% image  "rect_smooth.png" "Rectangle without horizontal jitter" 350 %}
{% endfigure %}

Although the `sync` machine is much larger, it isn't actually much more complicated. Most of the extra length is due to the fact that it has to handle each of the separate stages of a video field separately as they're all slightly different. Within this machine, the Y register is inverted after each run-through as an odd/even field flag, so that the machine knows which form of vertical blanking interval to send as previously described.

When the composite video peripheral is running, the main processor needs to periodically refill the `data` machine's input buffer so that it has data available to send to the screen.  Ideally this would be implemented using the DMA subsystem of the Pico, however I didn't have time to figure out how to do so for this project.  Currently, data is instead supplied via a system interrupt that is raised whenever the buffer isn't full.  This solution is mostly sufficient, however I have noticed that using the Pico SDK's UART `printf` function sometimes causes the PIO to briefly run out of data. I suspect that something within the SDK code delays the PIO interrupt, either by disabling interrupts or enabling a higher priority one, allowing the PIO to run down the data remaining within the buffer.

#### Handling screen data

With the low-level PIO implementation completed, I needed to create something capable of storing and manipulating data representing an image for the screen, and sending this data out to the PIO one word at a time in the correct order. This is the task of the `renderer` module in my implementation, within which the screen state is represented as a 2D array of integers where each bit corresponds to a single pixel. The `renderer` keeps track of the position of the last data word sent to the PIO within this array, and alternates between odd/even rows as required.  It also contains a selection of functions that facilitate the drawing of various objects on-screen.

Basic rectangles were the first type of object that I tackled, as they would be needed for the majority of the graphics in Pong. I then looked at drawing actual images, which I realised would be possible after I stumbled upon the [XBM image format](https://en.wikipedia.org/wiki/X_BitMap). This is an old binary image format that stores the data within a C-compatible source file, so it can be directly imported into existing code.  Even better, `imagemagick` supports it so it's straightforward to convert other image formats into XBM files. Writing some code to get this data into the screen data array was fairly easy too, barring a couple of minor hiccups. Firstly, XBM images use 1 for black and 0 for white, which is inverted relative to my rendering implementation so I had to remember to flip each bit when writing it.  Secondly, the XBM image data is stored within 8-bit unsigned chars, with the last byte of each row padded if the image width is not a multiple of 8.  I didn't initially take this padding into account when running through the image array, which led to some images rendering fine and others not depending on their dimensions.

The main reason for implementing image drawing was so that I could use a bitmap font to show the player scores on-screen, as well as some text when the game started or someone won. All of the bitmap font generators I tried seemed to pack the characters into the generated bitmap as tightly as possible, which would save space but also make text rendering more complicated as I'd need a separate lookup table containing each character's position and dimensions. Fortunately I was able to find a monospace bitmap font arranged in a uniform grid freely available [online](https://opengameart.org/content/monospace-bitmap-fonts-english-russian), so I chose to use this instead of making my own. 

{% figure "Monospace font bitmap with uniform spacing. <a href='https://opengameart.org/content/monospace-bitmap-fonts-english-russian'>Source</a>" %}
{% image  "font.png" "Monospace font bitmap with uniform spacing." %}
{% endfigure %}

Very conveniently, the position of the character within the bitmap grid corresponds to its ASCII code. After converting the font image above to XBM, it was therefore quite straightforward to get text rendering to the screen by building off of my previous image drawing code. I was even able to add text justification, so that the player score counters could remain centred when the count reached double digits.

Initially, the drawing of both images and rectangles was implemented in a bitwise fashion where each bit within the drawing region was set individually.  This kept the drawing code simple, however it turned out to be far too slow - for anything other than small shapes, the screen redrawing rate would take a significant hit.  As each line of screen data is stored as several 32-bit words, the process could in theory be sped up significantly by writing whole words to the screen data array instead.  However, unless objects are aligned to the word boundaries, this is non-trivial. I was able to refactor the rectangle drawing code to write whole words, but ran out of time to do so for the XBM image drawing as this turned out to be significantly more complicated. In addition to the screen data array word offset, there was also the image data array that would be offset too - and XBM data is stored in 8-bit chunks, which is different to the screen data word size, further adding to the confusion. 

The net result is that rendering too much text or large images on the screen still impacts the screen redrawing time significantly, however as Pong is mostly comprised of basic rectangles anyway this isn't a major issue for this project. Hopefully I will be able to return to this in future and improve the image rendering code so that my composite video implementation is more generally useful.


### The final product
Once I had a functioning renderer, actually recreating Pong was fairly straightforward. I even had time to include a couple of features of the game that I wasn't previously aware of: the ball accelerates after each successful deflection, and the player bats are not actually flat as far as ball bounces are concerned.  The game update speed is controlled by a repeating timer, so it should in theory run at the same rate regardless of clockspeed.

The Git repository for the completed project can be found [here](https://github.com/alanpreed/pico-composite-video). Below is a short video demonstrating my Pong clone running on the Pico, supplying video output to a standard TV.

<div class="youtube-video-container">
  <iframe 
    src="https://www.youtube.com/embed/IoYIfUzPTYo"
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
    allowfullscreen>
  </iframe>
</div class="youtube-video-container">

I have also created a copy of the project on the [Wokwi simulator](https://wokwi.com/arduino/projects/304031382352429632) so that it can be seen in action without access to the physical hardware, although currently the simulation does run rather slowly
