---
layout: post
author: Ronny
---

All right! I am very excited for this project, probably the most excited I have
been about building something for a long while. So, without further ado:

## Rationale, Goals

I recently got an electric guitar (the 8 string Ibanez RGIR28FE), and I was looking
into buying guitar effects pedals, and realized how expensive they are! They can get
prohibitively expensive, especially if you need/want many chained together for the
desired sound.

Even before becoming aware of this, I decided I wanted to build a pedal or two on my
own, just for the learning experience. I was originally thinking of making analog pedals,
but the more I thought of this, the less I was convinced this would be the best approach.

### Why Not Analog?

For starters, there are a lot of effects which are quite impractical to do in the world
of analog electronics. Specifically, delays, echos, and similar are quite difficult, and
usually use specialized discrete time circuitry to get the job done (see the
[bucket brigade circuit](https://en.wikipedia.org/wiki/Bucket-brigade_device)).
This type of circuity is often rather expensive, and has limited availability. Also,
input and output analog buffers/amplifiers have to be present in all effects pedals,
since it is up to the user where in the effects chain the pedal will be used; this means
that for the best quality, a large amount of high quality ICs (op-amps or similar) would
have to be used for this purpose (at least two per pedal).

The other factor is that with analog effects pedals, each effect that I want to have
would require extensive effort to be put into circuit design, component selection, PCB layout
and ordering, casing/UI design and implementation, etc. So basically, the marginal cost both
economically and in terms of time and effort would be excessively high per effect.


### Digital to the Rescue!

The approach I settled on is to build an all-in-one digital effects pedal, for a few reasons.
An all-in-one digital effects pedal is a much more complex project than something
like a simple analog distortion pedal; it involves analog circuitry,sampling chips
(ADCs and DACs), special noise considerations due to mixed digital/analog circuity,
and incredibly time-sensitive DSP programming (digital signal processing). 
The user interface and user experience for an all-in-one pedal are also far from trivial, 
involving complex tasks such as saving/loading effects chains (how should they be represented?
how can glitches be avoided when switching between presets? etc) and making a robust physical
interface that can literally be stepped on.

However, these complexities come with rewards that make an all-in-one digital pedal
totally worth the added effort:

First and foremost, once the pedal is in working order,
adding a new effect is as simple as writing code to simulate it. This is obviously simpler
than the analog equivalent, as you skip a lot of the tediousness of worrying about biasing,
input/output buffering, and having to physically design, test, and build a full circuit per
effect.

The potential for being able to hot-swap between effects chains in real time is also a
huge bonus. Adding in looping capabilities, wireless control, and other cool features
also becomes more feasible on a fully digital platform.

### Goals

My goals for this project can be sorted into two categories, core goals that are essential
to creating a working, usable guitar pedal, and nice-to-have goals which define cool
features that could be added on top of the core of the pedal.

**Core Goals:**
- Create a digital guitar pedal which can apply multiple effects in a chain to
an incoming guitar signal.
- Have a variety of basic effects available (distortion, chorus, compressor, etc).
- Introduce minimal latency to the signal (this would be the main benefit over simply
plugging your guitar into a laptop and digital interface).
- Implement a user interface which can both program the effects as well as turn them on or
off with foot switches.
- Have input and output lines which are compatible with standart sound equipment (high
impedance instrument input, possibly balanced XLR output along with the standard 1/4" jack).
- Maintain studio quality or better throughout the signal chain (so ADC, DAC, analog
buffers, DSP techniques must conform to this) (why build a pedal yourself if it's not
going to be at least as good as what you could get at the store? :P).

**Nice-To-Have Goals**
- Allow for control/communication with a desktop app.
- Implement complex signal chain behaviour, with capability to save/load presets, possibly
saving to external computer or SD card.
- Implement looping capability.
- Implement fancy graphical EQ with arbitrary curve input by the user.

## General Plan

With these goals outlined, the general plan for achieving them can be laid out, and
components can be chosen to align with the goals and the plan.

The general plan is outlined in the below block diagram:

<img  src="/assets/images/DSP-EG/DSP-EG-block-diagram.excalidraw.svg">

First, the signal from the guitar goes through a high impedance buffer/amplifier stage,
which is necessary to separate the weak guitar signal from the ADC input. This also allows
analog pre-amplification of excessively weak signals.

Then, ADC samples and digitizes the analog signal, and communicates it to a microprocessor,
which uses DSP techinques to apply the audio effects efficiently and in real-time.

The microcontroller then feeds the output digital sound to a DAC, which converts it back
into an analog signal.

This signal goes through a final buffering stage to make it compatible with other audio
equipment (other pedals, digital interfaces, amps, mixers, etc).

### Component Selection

I have not yet chosen specific components for the analog buffering stages, but these will
likely be some high-quality op-amps of some sort, possibly with the addition of a specialized
IC for balanced XLR output.

The ADC that I have chosen for this project is the TI PCM1863, which features stereo
24-bit data output (decimated from 32-bit readings on the actual ADC) at 192kHz,
which is better than studio quality.

For the microprocessor, I originally was thinking of using the Raspberry Pi Pico 2, since this
is a cheap and fast microcontroller; it has dual 32 bit cores at 150MHz, and 512kB of RAM.

However, I was thinking about delay effects, or the possibility of looping capability, and
it became clear that this board does not have enough memory. For 1 second of mono audio,
storing 32-bit values, you would need
$$
192,000 \text{ samples/sec} \cdot 4 \text{ bytes/sample} \cdot 1\text{ sec} = 768\text{ kB}$$
, already well over the total ram on the pico.

I could not find a cheap microcontroller that I felt had both enough memory space and
enough computational throughput, so I went for something a little crazier:

The Raspberry Pi Zero 2 W goes for about $23 on digikey, it has a quad-core 64-bit ARM
processor clocked at 1GHz, and it has 512MB of memory, so a huge improvement over the
pico in basically all factors.

However, it is meant to run some flavor of linux, with code written for it running
on top of the OS. This is problematic, since one of the goals is minimal latency,
and having linux's scheduler doing weird things is a big no-no; it would be difficult
if not impossible to satisfy the strict timing requirements necessary for the project with
the code running ontop of a whole OS.

The Zero 2 W's specs were just too good to pass up for that price point, so the ultimate
choice I made was to write all the code in ***bare metal Rust!!*** This is going to take
quite some learning, but it will be worth it in the end, since I will be able to write
incredibly fast and deterministic implementations for all the effects!

The DAC I chose is the Asahi Kasei AK4432 DAC, which features 32 bits of depth at 192kHz
sample rate, again better than studio quality.

The DAC and the ADC will communicate with the Pi Zero 2 using I2S, a serial comms protocol
for digital audio communications.

## So It Begins!

This is basically the outline of the project, with some explanation as to why I've made
certain design choices!

I have a lot to learn and a lot to do! getting things up and running on bare metal will
be far from trivial; I wasn't able to find tutorials specific to the Pi Zero 2, and
I will have to write many things from scratch since no one makes libraries for bare metal
Rust on RPis.

Also, I have to build a circuit incorporating the ADC and DAC, as well as some analog
buffering.

I am very excited to work on this project, as it is complex, but I am confident in my
ability to get it done while enjoying the journey!
