---
layout: post
Author: Ronny
---

After not working on WaveTrace for a few weeks, I've spent some time since mid January making good progress.

While I said in my <a href="/projects/wavetrace/2024/progress-so-far/" target="_blank">previous update</a> that I wanted to give the graph an overhaul, I changed my mind, since it works well enough for testing purposes, and will not be used in the final project (so testing purposes is the only important context for now).

Instead, I put together yet another iteration of the project, which I uploaded to the repo <a href="https://github.com/248nonny/FFT-Graph-test" target="_blank">FFT-Graph-Test</a>. I set out with a few goals for this iteration: I wanted to get the PortAudio library working with an audio buffer that could store up to a couple seconds worth of audio data (read from my laptop's microphone), and then I wanted to use the FFTW library to process this data, and display the output spectrum in a Graph (one of the GTK ones I <a href="/projects/wavetrace/2024/progress-so-far/" target="_blank">coded myself</a>).

<br>
## The Latest Iteration

The reason I decided to once again start from scratch, instead of forking or adding on to my previous iteration (available in the <a href="https://github.com/248nonny/FFT-Test" target="_blank">FFT-Test</a> github repo), was that I had some second thoughts about how my previous iteration was structured; I created lots of classes for objects and had an idea of how the program could operate at a high level, but when I got into the details, my original plan turned out to be rather messier than I deemed tolerable.

<br>
### Details for Nerds (liek me :o )

#### General Structure

The latest iteration works like this: I created a namespace `Audio`, which has functions for initializing PA (PortAudio), printing the available PA devices to the terminal, and an object class `Audio::AudioHandler` which has the capacity to open a PA stream with the device of your choice. Then, partially copying code from other c++ tests I had written in the past, I made the `Audio::AudioBuffer` class, which can cyclically store incoming audio data that PortAudio streams receive (I believe what I created is very similar to a <a href="https://en.wikipedia.org/wiki/Circular_buffer" target="_blank">circular buffer</a>, although I was unaware of this term when I made it). Each `AudioHandler` has an associated buffer, and a pointer to the buffer is fed through the handler's audio stream to the audio callback function, which is called every time there is a certain amount of new audio data.

Then, I also made a Gtk `MainWindow` class (inheriting from `Gtk::Window`), which contains the graph objects and the code to read values and display them on the graphs. In order to read data from the audio buffer and display it on the graphs (as a test), I used a `Glib::signal_timeout` to repeatedly call a function every couple (about 40) milliseconds.

I also created an `FFTProcessor` class, but I haven't yet gotten around to writing the code for it; I decided to first test all the other components of the program to ensure they worked. `FFTProcessor` are also fed through the audio streams to the callback function, so that the processors can be informed when there is new audio data to process.

So, to recap so far, port audio is initialized with functions that exist in the `Audio` namespace, the audio handler creates PA streams and feeds audio buffers and fft processors to said streams.

The program flow is something like this:
- initialize PA.
- create audio handler, choose PA device.
- open the stream, creating buffer(s) and fftprocessor(s) (one per channel) for the stream.
- start an instance of `MainWindow` as a GTK app, feeding it pointers to the audio buffers from the (now open) stream.

<br>

#### An Option to Disable GTK
Something that I added last iteration of this project, but only properly implemented this time around, is that I added a `USE_GTK_GUI` option in cmake that when disabled, does not link or include any glib or gtk libraries. Furthermore, it also defines a compile definition `USE_GTK` that is used in conjunction with `#ifdef` to only read headers and .cpp files during compilation if the `USE_GTK_GUI` option is enabled. This is useful because I think this iteration is likely to be the final one (I am (so far at least) pretty happy with the structure, so any iteration would likely be building on top of this instead of starting from scratch), and in the future, I'm considering using a raspberry pi (or something similar) to mount the finished proof of concept (POC) on my bike so I can road bike safely. Instead of a GTK GUI, I would likely use an LED ring or similar, something more hardware based, and for such an application it is useful to be able to exclude GTK libraries and files from being compiled (for speed, filesize and efficiency I suppose).

<br>

### What it does so far

In its current state, the program is able to successfully open an audio stream to access my left / right laptop microphones, write the data to the buffer, and then read data from the buffer and display it on the graph. Below is a GIF showing it in action:
<img src="/assets/images/WaveTrace/audio-graph.GIF" class="gallery-image">
Lamentably, running the program makes the audio unavailable to other programs, so I wasn't able to record the sound (hence the choice of GIF as the format), but I am speaking and whistling and making noise, and the raw audio data is being displayed on the graph.

#### Coming Up Soon...
This is a very exciting milestone for me, since none of the past iteration ever had all these components working together. Also, I'm quite happy with the structure of the program; with the use of classes, and better code structure than previous iterations, I feel confident that it will be easy to adapt this code to other audio sources (which I must do for triangulation and the true WaveTrace POC), so hooray for <a href="https://en.wikipedia.org/wiki/Object-oriented_programming" target="_blank">OOP</a>!

Next up I'll write the code for the FFTProcessor, and make it so that the graphs van display the spectra they output (as another small POC), after which I will build an experimental setup with multiple microphones so I can start experimenting with triangulation.