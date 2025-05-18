---
name: DSP-PEG
layout: project_page
gallery: true
github_repo: DSP-PEG
status: Ongoing
started: April 2025
importance: "aaa"
description: 'Digital Signal Processing - Programmable Effects Generator'
---


### Digital Signal Processing - Programmable Effects Generator!

After purchasing my first electric guitar, I decided it would be
fun and interesing to create my own guitar pedals. Making analog pedals sounded impractical,
since implementing each new effect would require substantial engineering
(designing the circuit, building and testing it, creating an enclosure, etc etc), so I took
inspiration from [Neural DSP's Quad Cortex](https://neuraldsp.com/quad-cortex) and
decided I would make an all-in-one digital effects pedal! While this is a much larger/more
difficult project than some simpler analog effects pedals, the marginal cost/effort
for each new effect would be much reduced - with a digital pedal, each new effect could
be implemented with just a bit of extra code!

Currently, I plan on using a [Raspberry Pi Zero 2 W](https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/)
(which will run linux in parallel with some diy bare-metal rust code for high-speed DSP
and digital effects), as well as a TI PCM1863 ADC and an AKM AK4432 DAC for high-quality
audio IO.

Simplified project block diagram:

<img  src="/assets/images/DSP-PEG/DSP-PEG-block-diagram.excalidraw.svg">

Find [here](https://github.com/248nonny/Digital_Stereo_IO_Breakout_Board) KiCAD files
for my ADC/DAC test board.

Here is the original [Project Overview](/projects/dsp-peg/2025/Project-Overview/).
