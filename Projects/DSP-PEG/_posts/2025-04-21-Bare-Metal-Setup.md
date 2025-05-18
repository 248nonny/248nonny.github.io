---
layout: post
author: Ronny
---

Checkout [the project's github](https://github.com/248nonny/DSP-PEG) to see the relevant
code/dev environment for this post (and the project at large I suppose). The (shortened)
commit hash at the time of writing was `9178bc6`.

The Raspberry Pi Zero 2 W usually runs Linux. This is not an ideal platform to create
real-time, low-latency digital guitar effects since the OS's scheduler makes
timing guiarantees basically nonexistent, and it also adds elements of uncertainty to
the concurrency of the code (ideally we want to control exactly what happens in each
of the 4 cores, but with the scheduler such control is difficult if not impossible).

The solution to this is usually to use a microcontroller of sorts, but I couldn't find
anything that matches the Zero 2 W's specs for it's price (~$23 CAD). So, the solution:
write the code in Rust for ***bare metal execution!*** This post will go over the basic
setup I went through in order to get code running bare-metal on the Zero 2 W.

I ordered a couple of Raspberry Pi Zero 2 Ws from
[Digikey](https://www.digikey.ca/en/products/detail/raspberry-pi/SC1176/15298147),
which cost about $22 CAD a piece. I also got some
[64GB SD cards](https://www.digikey.ca/en/products/detail/raspberry-pi/SC1629/24627140)
(which are necessary to boot anything on the Zeros). 

### Getting Started

I found a few resources online, but nothing specific to the Zero 2 W board.
However, from what I could tell the Raspberry Pis work similarly to each other,
so I decided to follow
[this tutorial](https://harmonicss.co.uk/bare-metal/rust-on-a-raspberry-pi-part-1/)
and make changes to suit the Zero 2 W as I went on.

It seems that the Raspberry Pi needs a few things on the SD card to run things.
If you look at the official Raspberry Pi disk image for the Zero 2 W, it has
some firmware files which are read and executed on boot ([seemingly by the
graphics card for some reason?](https://williamdurand.fr/2021/01/23/bare-metal-raspberry-pi-2-programming/))
. In the official disk image, it has a bunch of versions of various files,
but from the tutorial I found that the only necessary ones are `fixup.dat`,
`start.elf`, and `bootcode.bin`. You can also add a `config.txt` file, which
is read by the GPU boot code, and here you can add some settings, such as
the filename of the kernel you want the GPU to boot.

I formatted the SD card to have a FAT32 partition (which is what is expected by the Pi),
and added the necessary boot files. I also added a 'config.txt' with the following
contents:

```
kernel=kernelDSPEG.img
arm_64bit=1
uart_2ndstage=1
```

The first line tells the GPU try and run the binary titled "kernelDSPEG.img",
the second line enables 64 bit mode (the Zero 2 W has a quad core 64 bit CPU),
and the third line enables debug output for the GPU firmware over UART.

I followed the tutorial mentioned above to set up Rust to compile for the right target,
and I copied the linker script from the tutorial as well. The Zero 2 W expects the
entry point for the kernel binary to be at the address `0x80000`, and this is achieved
through the linker script (note that the tutorial says the address should be `0x8000`;
this is for most pi models, but the RPi 4 and Zero 2 (and possibly others)
start at `0x80000`).

I continued to follow the tutorial, but could not get an LED to blink for a while.
It turns out that while most RPi models have similar relative physical addresses for
peripherals, they can have different base addresses. If you look at the
[peripheral spec](https://datasheets.raspberrypi.com/bcm2835/bcm2835-peripherals.pdf)
(found at [RPis' official "Processors" page](https://www.raspberrypi.com/documentation/computers/processors.html))
for the BCM2835 chip, there is a detailed description of which physical addresses
correspond to which registers of which peripherals (peripherals are all the non-cpu
hardware elements of the RPi boards, everything from the GPIO pins to UART to SPI and USB).

The BCM3835 is used in some Pi 1 models, as well as in the Pi Zero; however, the Pi Zero
2 (and 2 W; W is for wireless) uses the RP3A0 chip (similar to the BCM2837). This means
that there is no guarantee that the physical addresses in the BCM2835 apply to the Zero
2 W, which would explain the lack of blinking LED in my test code. However, it turns out
that the physical addresses across Pi models tend to follow the same pattern, and the
main difference is some constant offset to the addresses that varies from chip to chip.

According to some online sources, the base offset for the Pi Zero 2 W is `0x3F200000`,
which is the base address used in the tutorial. At some point, I was using ChatGPT
to help explain the code, and that's where I got the wrong base address from! I changed
it back to the correct address from the tutorial, and the LED started blinking! Hooray!

## UART Communication

The next step was to add UART communication. I used some online resources and the
peripheral address document to turn on the UART module on the RPi, as well as
configure the GPIO pins for UART. Then, since getting the built-in USB port on the Pi
is very difficult to do (since the USB protocol is kind of monstrously complex),
I simply hooked it up to an ESP32-c6 dev board that I had lying around. I put some
Arduino code on the ESP32 which forwards serial UART messages back and forth between
the RPi and my laptop; the code was quite simple:
```c++
// ESP32-C6 UART to USB bridge
void setup() {
  Serial.begin(115200);       // USB to PC
  Serial1.begin(115200, SERIAL_8N1, 20, 21);  // UART: RX=20, TX=21

  while (!Serial) {}
}

void loop() {
  // Forward data from ESP UART (Serial1) to USB serial (Serial)
  if (Serial1.available()) {
    Serial.write(Serial1.read());
  }

  // Echo PC commands back to Pi
  if (Serial.available()) {
    Serial1.write(Serial.read());
  }
}
```

This allowed me to successfully print out "Hello, World!" character by character from
the Raspberry Pi by writing individual characters to the UART peripheral. The
Rust code to send a single character looks like this:
```rust
core::ptr::write_volatile(UART0_DR, b'H' as u32);
```
Where `UART0_DR` is the UART physical address for characters to send.


## Conclusion

I am happy that I managed to get bare metal rust code to run on the RPi Zero 2 W!
The tutorial was very helpful. The next things on my to-do list are:
- Get multi-core execution working (this is important for distributing the
computational load of effects processing).
- Set up some debugging foundations
  - Specifically, I need to implement something that allows me to print full
  strings, ideally with formatting, in a concurrency-safe way.
- Set up some benchmarking foundations
  - I want to be able to benchmark my effects, DSP operations, and memory IO down to
  the clock cycle. Doing this will involve writing some macros (for convenience),
  as well as reading the clock cycle counter register.

As always, I am looking forward to continuing this project!
