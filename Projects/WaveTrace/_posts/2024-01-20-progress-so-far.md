---
layout: post
---

Over summer 2023, the ideas for this project were still fermenting; I decided to use c++ for portability and speed, did some thought experiments on the topic of audio triangulation by frequency, and did some experimenting with libraries for calculating FFTs and reading from audio devices; the libraries I ended up choosing are [FFTW](https://www.fftw.org/), which has many overkill features, but is free to use, and [port audio](https://www.portaudio.com/), which is also free and open source.

After going through some examples and tutorials, I wanted to test out both libraries at once, feeding audio data to FFTW, but I wanted to graph the output, and could not find any good visual graphing libraries.

I also wanted some GUI elements for testing purposes, which led me to learn about [GTK](https://www.gtk.org/) and [gtkmm](https://gtkmm.org/en/index.html), the latter being a c++ interface for the GTK library. This library has many built in UI elements, such as buttons, input fields, and check boxes, but no graphs. However, I saw that there was a "DrawingArea" widget that allows you to draw shapes and lines in different colors, so I decided to use this widget to create a graph widget myself.

The code for the graph is available [here](https://github.com/248nonny/gtkmm-graph/), in a github repository that I'm using as a [submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) in the main WaveTrace and testing repos. I will also be able to use this as a submodule for future projects if I ever need to graph something with gtkmm or c++.


The code for this, if memory serves correctly, was made late into summer 2023, and I only got to testing all the libraries together in September, once school started. Also, since this is my first "proper" c++ project, the tests went through multiple iterations, where I would go crazy and code code code, realize after the fact that I made some rather large planning errors, then essentially restart and make a slightly more refined version of the test.

I also made the poor choice to try to implement multiple FFTs of different array sizes all at once without first testing a single one; this led to much troubleshooting, and I never got it working properly. This is why baby steps are important.

At the time of writing this, I believe my repo titled [FFT-Test](https://github.com/248nonny/FFT-Test/) contains my most recent iteration of this process.

<br>

## Moving Forwards

From here, I plan to do a few things. First, I'm going to give the graph object an overhaul, and make it less of a pain to use; currently you need to specify the domain and range to graph on, and if part of the graph is cut off the line doesn't render properly. I would like to make it more autonomous and adaptive in its dimensions.

After that, I will make a simple example project which reads audio from my laptop's microphone, calculates an FFT on it, and displays the FFT on a graph; this will work as a test to ensure all the libraries and the graph work as expected.

Also, the next iteration of WaveTrace (and hopefully all future ones) will have two compile modes, one with a GTK GUI, and one without a GUI. This way, the GUI can be used for testing only, and in a production environment WaveTrace can be a much smaller library, and possibly have less overhead from the GTK library.