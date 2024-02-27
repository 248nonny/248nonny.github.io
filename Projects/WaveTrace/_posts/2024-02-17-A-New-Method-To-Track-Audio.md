---
layout: post
author: Ronny
---


I was describing how the theory behind how this project could work to a friend of mine when I realized that there is a much better way to guess what direction audio comes from than triangulation (read [this post](/projects/wavetrace/2024/how-it-will-work/) for the original idea's context).

Now, this new method is actually pretty obvious; I feel just a little bit dumb for not thinking of this before thinking of the comparatively more complex triangulation method. Consider this: you take your microphones, and put some sort of a cone on each one (kind of like a sick dog), so that it picks up sound in one direction stronger than sound from other directions. Then, you just place your coned microphones so that they all point in different directions, and based off the intensity of the signal from each microphone, you can predict which direction the sound is coming from (likely somewhere between the mic with the strongest and second strongest signals). By placing the microphones closer together, the effects that the distance between mics has on the signals is minimized, and this also has the added benefit that the microhpones can be placed on a smaller object (e.g. helmet) which may be convenient for future applications.

Now, proper directional microphones would be best suited for this application, but I've already bought a couple omnidirectional mics for testing, so I'll try to use these with 3D printed cones and then perhaps upgrade.

<img src="/assets/images/WaveTrace/poc/mic-setup.svg" class="gallery-image">

A cutaway of the setup might look something like the above image, where the red dots represent microphones, and the green(ish) lines represent barriers or dividers of sorts that essentially act to make the microphones more directional.

With some testing, I think I can develop some sort of a curve fit that for a setup like the above, which would map the angle that the device is at to the intensity of a sound signal in each of the microphones, given a single source of sound from one direction. Using this, I could then work backwards to figure out in what direction a sound is relative to the device given the intensity of the signal at each microphone.

The key part in this new method is still the FFT, which is still what will allow multiple sounds to be tracked at once; we just measure the intensity of each frequency at each microphone, and apply the inverse curve fit to each frequency to estimate its origin.

The immediate next step I will take is to design some 3D printed mounts for the microphones that have cones built in, so that I may implement the above strategy. I will also make a frame to fix the microphones to. All in all, it looks like I'll have a (hopefully) working proof of concept in the near future! (I am excited eheheh :D)