---
layout: post
author: Ronny
---

Checkout [the project's github](https://github.com/248nonny/DSP-PEG) to see the relevant
code/dev environment for this post (and the project at large I suppose). The (shortened)
commit hash at the time of writing was `9178bc6`.

In this post, I will share some ideas I have for what I have in mind for the effects
pedal's software at a high level. These ideas are still preliminary, and are likely to
evolve as I experiment with various approaches.

## Ideas So Far

To start, I will only use mono audio, as this is what comes out of most guitars anyways.
So the incoming data rate will be 192kHz @ 32 bits per sample. The period of the data
rate is $$1/192000\approx5.2\;\mu\text{s}$$. This means that the software can spend
at most around 5μs on one data point before it must be ready for the next one.

### Data Pipeline

To leverage the 4 cores available on the Rpi Zero 2, I thought of the following
setup for what happens where: core 0 would be in charge of the user interface and related
peripherals (e.g. rendering to a screen, reading potentiometers and buttons to update
effects parameters, etc), and cores 1-3 could create a tightly-coupled data pipeline.
In this pipeline, core 1 would read the incoming I2S data from the ADC, execute about
1/3rd  of the computational load of the audio effects chain, then pass the output to core
2 through the mailbox system (documented in the BCM2835 ARM peripherals docs), which would
execute another 1/3rd of the computational load, passing the output to core 3, which
would execute the remaining 1/3rd of the computational load, and write the output to the
DAC through I2S.

As long as each of the cores from 1-3 can run their section of the effects chain in under
5μs, they will be ready for the next point on time. This buys us three times the budget,
so that as long as an effects chain takes under 15μs to execute, it can be run on time
by distributing the load. This would increase latency, but since we are in the microseconds
regime, this is more than acceptable.

### Memory IO

A likely (and common) bottleneck in the effects chain is reading/writing values to RAM
(i.e. cache misses). This will be especially probable for effects which require large audio
buffers, such as delays, looping, echo, etc., as the buffers will be quite large and won't
fit in the processor cache. The good news is that these buffers would be circular, and
the memory location for the next buffer data points is deterministic. This means that some
inline assembly could be used to prefetch the next few data points ahead of time, making
it less likely to suffer from cache misses. The
[PRFM](https://developer.arm.com/documentation/ddi0602/2022-06/Base-Instructions/PRFM--immediate---Prefetch-Memory--immediate--)
ARM64 instruction could be used for this. The ARM docs for the processor used in the RPi
Zero 2 W (the [Cortex-A53](https://developer.arm.com/documentation/ddi0500/latest/))
says it can [autodetect cache miss patterns and prefetch things on its own,](https://developer.arm.com/documentation/ddi0500/j/Level-1-Memory-System/Data-prefetching/Data-prefetching-and-monitoring?lang=en)
but it might be worth using the PRFM instruction and benchmarking speed with and without
this. Memory alignment will also be something I might implement, i.e. storing things like FIR
coefficients in cacheline-aligned memory for efficient cache storage.

### Fixed Point Audio

To maximize speed and maintain deterministic timing in computations, audio will be represented in
fixed-point format, likely signed [Q2.30](https://en.wikipedia.org/wiki/Q_(number_format)),
with allowed values between -1 and 1 inclusive. This avoids floating point computation
(as the underlying math will end up being simple integer math with some small extra steps).
The underlying representation would be 32-bit integers of some sort, which means that
multiplications and divisions will be quite efficient, since the processors are 64-bit.
This means that we can simply cast the 32-bit numbers to 64 bits, multiply them together,
and then bitshift the output to the right for multiplication, and for division they could
be cast to 64 bits, the numerator would be shifted left, and then they would be divided.
On a 32-bit processor this would take extra steps, since the upper and lower bits of the
multiplication/division intermediate 64-bit steps would need to be stored separately.

I will be implementing this Q2.30 format myself, along with some mathematical operations
(the usual +, -, ×, ÷, as well as probably sin and cos for LFOs, and possibly sqrt and others
as necessary).

### Benchmarking

To measure performance of mathematical operations on my Q2.30 format, memory IO, and
the computational load of various effects for accurate distribution across processors
in the data pipeline, I will need a way to benchmark speeds. The way I plan on doing this
is to create some benchmarking macros which will run some code many many times, and measure
the clock cycles by setting up and reading from the
[performance monitor registers.](https://developer.arm.com/documentation/ddi0500/j/System-Control/AArch64-register-summary/AArch64-performance-monitor-registers?lang=en)

This should allow for accurate timing measurements, which would be necessary to properly
optimize my code.

### Serial Output and Input

The UART peripheral allows for writing and reading single characters at a time, and has
FIFO buffers for input and output (16x8 bits input, 16x12 bits output). Without anything
built on top of this, it becomes inconvenient to send messages out, especially concurrently
(i.e. from multiple cores at once). I plan on implementing a "message buffer" of sorts,
which will allow for concurrently receiving "messages," i.e. full strings to print out.
Core 0 (along with some interrupts) would handle sending out the messages in order,
character by character. This avoids interleaving of messages (i.e. trying to print
two things at once and characters getting mixed up), and also makes the overhead of
printing something as small as just writing the string to memory (at least for cores
which aren't core 0).

For serial input, I am thinking of creating a basic message receiver/parser which reads
serial input until a newline is received, and then parses it for commands. This could be
used for externally programming effects, or changing parameters on the fly during
development, before the UI is fully fleshed out.

### Effects Chain Representation

This is something to implement after I have some basic effects running, but for the final
product, the end goal is to have some representation of effects chains as a whole. Such a
representation would be resolved into how many buffers of what type are necessary, as well
as what effects should be run in each core in the data pipeline. This resolving capability
would allow maximum efficiency to be retained, while adding the ability to add, modify,
and switch between many effects chains. Perhaps these could be loaded or saved to the SD
card to make them persist across power cycles.

### User Interface

UI is something I will focus on later, once I can get the core software and hardware working.
This would be handled by core 0, and would involve rendering things to a screen, reading
input from potentiometers, encoders, and buttons. For example, if I want wah pedal
effects, I could build a rocker pedal, and the UI part of the code would read the rocker's
position, and use that to update a parametrized FIR filter's coefficients for wah style
frequency response modulation.

The UI code would similarly receive input from stomp buttons to enable/disable effects,
switch between effects chains, enable/disable looping, etc.

In terms of rendering, some form of GUI would have to be created to visualize and modify
effects chains, perhaps with the capability to arbitrarily map buttons and potentiometers
to parameters.

## Next Steps

There is a lot of programming to be done. What I plan on working on first (vaguely
in order) are the foundational elements, i.e. getting code to run on multiple cores, 
setting up a benchmarking framework, implementing basic serial output/input at the
message/string level, implementing the Q2.30 datatype, and writing a basic data
pipeline with cores 1-3 (using the mailboxes).

At some point soon I will also populate the ADC/DAC breakout board, and then I can also
start experimenting with the I2S input/output, as well as programming the ADC and DAC.
