---
layout: post
author: Ronny
---

When talking to someone with bare-metal kernel experience about my project, they
shared a truly glorious idea: Instead of running bare metal code everywhere,
including on core 0 for handling UI, they suggested I run linux on core 0 only, and
keep the bare metal data pipeline on cores 1-3!

The reason that this is such a huge improvement over my previous plan is that with linux
I would have access to tons of built-in drivers and functionality, whereas on bare metal
I would have to implement absolutely everything myself (and for example implementing
USB or network drivers would be HUUUUUGGGEEE projects in and of themselves).

This would suddenly allow me to use the graphics card, use existing UI libraries on a
screen, use the network and usb systems with relative ease, and many many more things!

## The Plan

The way this would be achieved is to have the default RPi flavor of linux running on
the RPi Zero 2 W. Then, I would have to write some kernel hooks and drivers to do the
following:
- At boot time, reserve a large chunk of physical memory for the bare metal effects
processing software/pipeline.
- Confine linux to use only 1 core, disabling linux interrupts and making sure the core's
clock speed is still at the max.
- Load in the bare-metal software to memory, and get the 3 cores to start executing it.
- Allow real-time communication between user space programs in linux and the bare metal
pipeline, mainly for updating effects parameters in real time.

Then, I would write and run UI software on the linux side of things, which would handle
reading physical inputs (potentiometers, buttons, rocker pedals, etc) and writing updates
to the real time pipeline.

I also need an abstract representation of arbitrary signal chains, as well as a way to
resolve them into reconfiguration instructions for the bare-metal pipeline, but that is
a later problem (lots to experiment with for now). I am also not sure whether such
reconfiguration would be best done by the kernel driver or in the bare-metal code...

<br>

In any case, the scope of the project has shifted, and for the better! The next steps
will be to learn how to write kernel drivers and hooks to achieve the above, which will
take some learning (I have never written a kernel module/driver or hook hehe). These changes
mean that I will now also learn a lot about the linux kernel and adjacent topics, which
is quite exciting!
