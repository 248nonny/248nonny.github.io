---
layout: post
tags: bash
title: Mono Audio on KDE Plasma
---


After I went deaf in my right ear (see [this](/about/) and [this](/projects/wavetrace/)), it became somewhat annoying to listen to songs that employ heavy panning, such as the guitar part in Herbie Hancock's <a href="https://open.spotify.com/track/2zQl59dZMzwhrmeSBEgiXY?si=10bfc94528db4116" target="_blank">Watermelon Man (Head Hunters)</a>. To fix this, I made a quick bash script that, through PortAudio (the audio API used by <a href="https://kde.org" target="_blank">KDE</a> until recently), creates a new virtual playback device (or virtual sink) that mixes the left and right channels to mono.

However, after installing Arch Linux on my new laptop, I discovered that the new default for audio device controls is PipeWire, which has similar features and capability, but broke my script's functionality.

I had to switch from using the `pacmd` command to `pactl`, as only the latter was compatible with the new PipeWire based installation. After some tweaking and adapting the script to the new command, the script looks something like this:

```bash
#!/bin/bash
if ! pactl list sinks | grep mono_aud > /dev/null; then
        pactl load-module module-remap-sink master=alsa_output.pci-0000_06_00.6.analog-stereo sink_name=mono_aud sink_properties="device.description='Mono'" channels=1 channel_map=mono
        echo "Global Mono Audio enabled."
else
        echo "mono audio already enabled."
fi
```

<br>

Essentially, it's almost just an alias for the below command, which creates a virtuial sink (essentially a virtual device) that when selected, takes the audio output of my laptop, down mixes it to mono (hence the `channels=1 channel_map=mono`), and then feeds it to its master device, in this case `alsa_output.pci-0000_06_00.6.analog-stereo`, which is the id of my laptop's audio card.
```bash
pactl load-module module-remap-sink master=alsa_output.pci-0000_06_00.6.analog-stereo sink_name=mono_aud sink_properties="device.description='Mono'" channels=1 channel_map=mono
```

If you're curious as to where to find the id of your audio card, you can run `pactl list sinks`, which should give you the names of all real and virtual audio devices on your system; it is from here that I obtained the name `alsa_output.pci-0000_06_00.6.analog-stereo`.

The difference is that the script first uses an if statement to check whether a virtual sink with the name `mono_aud' already exists, and only makes the new sink if none exist.

I named the script `monoAUD` for mono audio, and I put it in my bash scripts directory, which is added to `$PATH` by my .bashrc file. This way, whenever I want mono audio, I just have to type in `monoAUD` into a terminal, and boom, the new virtual device is created and automatically selected by my desktop environment. Problem solved; now I can hear all parts of all songs, regardless of how they're mixed :D