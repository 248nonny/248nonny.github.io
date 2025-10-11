---
layout: post
author: Ronny
title: '"Hello, World!" on Linux and Bare Metal Concurrently'
---

After much experimentation, and diving head-first into linux kernel driver development, I have been able to reserve a chunk of memory for bare-metal purposes using the device tree, load linux onto the RPi, compile and run a kernel module which loads in a binary payload to the reserved memory, and I have exectued the binary in bare-metal and observed it running!

## Recap

As a quick recap, in April I was able to compile, load, and execute a binary (compiled from rust) in the RPi Zero 2 W, and I was successfully able to run code on all four cores at once (confirmed by writing to UART from each core respectively). I have been pretty busy since May, since I did a summer semester with many classes, including the fabled [Robot Summer class](/2025/my-summer-so-far/), where I won first place in the yearly robotics competition.

In my (somewhat sparse) free time I was thinking about the idea introduced [last DSP-PEG post]({% link Projects/DSP-PEG/_posts/2025-05-18-Change-Of-Plans.md %}), where I could run Linux on one core and bare metal on the other three. This approach has the best of both worlds; a plethora of linux driver which would allow me to easily use wifi/bluetooth, the graphics card + hdmi port, the usb ports, etc., while still allowing me to execute bare metal code to avoid the scheduler and ensure maximum speed.

## What I did

Over the course of the summer, on random weekends or chunks of free time, I did some reading on the basics of Linux kernel module development, since a kernel module seemed like the best way to achieve my goal (I could have modified the kernel source, but the build time would have been super long, and I would have to use cross compilation which seemed like a pain).

First, I tried to get linux to only use one core, and to reserve a chunk of physical memory for my bare metal program. I though I did the former successfully at first by setting `maxcpus=1 isolcpus=1-3 irqaffinity=0 rcu_nocbs=1-3` in my `cmdline.txt` file, though I ran into issues down the road (more on this later). I reserved memory by modifying the `reserved-memory` section of the device tree:

```dtc
reserved-memory {
		#address-cells = <0x01>;
		#size-cells = <0x01>;
		ranges;
		phandle = <0x3c>;

		dsp_shared: dsp_shared@10000000 {
			reg = <0x10000000 0x00100000>; // 1MB of shared memory for two-wway communication between DSP and kernel.
			no-map;
		};

		dsp_reserved: dsp_reserved@10100000 {
			reg = <0x10100000 0x0B37F000>; // reserved for bare-metal cores, ending before GPU memory (last 64MB).
			no-map;
		};

		linux,cma {
			compatible = "shared-dma-pool";
			reg = <0x0C000000 0x04000000>;
			reusable;
			linux,cma-default;
			phandle = <0x3d>;
		};
	};
```

Then I wrote a kernel module which maps these memory regions to the kernel's memory space (so I could write stuff there) using `ioremap()` (which is usually uncached by default), writes a binary payload to the `dsp_reserved` region, and attempts to tell core 1 to start executing whatever is at the beginning of `dsp_reserved`.

The kernel module code (which is currently in a bit of a hacky, spaghetti state, sorry :P) can be found [in the project repo](https://github.com/248nonny/DSP-PEG). At a high level, this is what it does:

- Search and claim `dsp_shared` and `dsp_reserved` memory regions from device tree (by name), and map them to writable (virtual) addresses.
- Load in the bare metal binary payload using the Linux `request_firmware` API ([check it out here](https://www.kernel.org/doc/html/v4.13/driver-api/firmware/request_firmware.html)).
- Attempt to wake up cpu1 and get it to execute the bare metal binary payload.

The kernel module also has various sanity checks along the way, like printing out a hexdump of the binary payload after it is loaded into memory, and in general being very verbose in terms of logging.

The binary payload currently mutates a number at the base of the `dsp_shared` region, which the kernel module reads constantly to look for the mutation (as proof that the program is actually running).

Here is what the bare metal code looks like in rust:

```rust
#![no_std]
#![no_main]

const SHARED_BASE: usize = 0x10000000;

const MAGIC_COUNTER: *mut u64 = (SHARED_BASE + 0x00) as *mut u64;

use core::arch::asm;
use core::panic::PanicInfo;

mod boot {
    use core::arch::global_asm;
    global_asm!(
        "
            .section .text._start
            .globl _start
        _start:
            ldr x0, = _stack_start_1
            mov sp, x0
            bl _rust_main
            "
    );
}

#[export_name = "_rust_main"]
pub extern "C" fn rust_main() {
    unsafe {
        let mut magic_counter = 0xAAAA_AAAA;
        core::ptr::write_volatile(MAGIC_COUNTER, magic_counter);

        loop {
            magic_counter += 1;
            core::ptr::write_volatile(MAGIC_COUNTER, magic_counter);

            for _ in 1..1000000 {
                asm!("nop");
            }

            for _ in 1..1000000 {
                asm!("nop");
            }
        }
    }
}

...

```

It is similar to the previous bare metal code, except now instead of blinking an LED, all it does is increments a magic value after setting it to `0xAAAA_AAAA`. Note that I also had to update the linker script to match the hardware address that the code will be loaded into.


## What Went Wrong

When I first tried to get this to work (i.e. compiled the bare metal binary, compiled and loaded the kernel module), nothing happened. This was really annoying, since it was my first time writing a kernel module, and I was not super confident in how everything works, so it was difficult to guess at where the problem was. After trying various quick changes (e.g. using memmap instead of ioremap, as well as adding many sanity checks such as the hex dump), I was somewhat stumped!

Over a few months (August-October), I did not work on the project much other than trying something here or there on a weeked; but on one such weekend, I figured it out!

It turns out that even with the stuff I added to cmdline.txt, linux was claiming the cpus from the spin tables and parking them somewhere else. I had a hunch that this might be the problem (though I had many other "hunches" which is why it took me a while to figure it out lol), and what I tried doing which finally worked was commenting out cores 1-3 in the device tree! This is definitely something of a hacky workaround, since linux can autodetect stuff sometimes, but it seems that it relies on the device tree to properly initialize the cpu cores!

This is what the relevant section of the device tree looked like after commenting stuff out for one of the cores (there are 4 such sections in the device tree):
```
cpu@1 {
	device_type = "cpu";
	compatible = "arm,cortex-a53";
	reg = <0x01>;
	enable-method = "none";
	// enable-method = "spin-table";
	// cpu-release-addr = <0x00 0xe0>;
	// d-cache-size = <0x8000>;
	// d-cache-line-size = <0x40>;
	// d-cache-sets = <0x80>;
	// i-cache-size = <0x8000>;
	// i-cache-line-size = <0x40>;
	// i-cache-sets = <0x100>;
	// next-level-cache = <0x20>;
	// phandle = <0x22>;
};
```

Once the information of which wake up method/address the cores have was removed, linux no longer touched them at all, and the existing binary file and kernel module immediately worked!!!

Below are the kernel logs before and after I commented out this part of the device tree:


Before updating the device tree:
```
[ 3555.714094] Loading DSP PEG driver... pi ≈ 3
[ 3555.714101] Reserving DSP Comms memory...
[ 3555.714136] Acquired resource [mem 0x10000000-0x100fffff] from name 'dsp_shared'.
[ 3555.714175] Finished executing ioremap for 'dsp_shared'.
[ 3555.714195] Acquired resource [mem 0x10100000-0x1b47efff] from name 'dsp_reserved'.
[ 3555.717690] Finished executing ioremap for 'dsp_reserved'.
[ 3555.717702] Successfully mapped memory!
[ 3555.717707] Writing and reading to shared memory as test...
[ 3555.717713] shared roundtrip: deadbeefc0dec0de
[ 3555.717721] Loading in bare-metal DSP firmware...
[ 3555.719070] dsp_reserved: (Hexdump omitted for brevity)
[ 3555.719212] Starting bare metal execution!
[ 3555.719218] spin-table[0x00000000000000e0] = 0x0000000010100000
[ 3555.920373] BM heartbeat64: deadbeefc0dec0de -> deadbeefc0dec0de
[ 3556.124367] BM heartbeat64: deadbeefc0dec0de -> deadbeefc0dec0de
[ 3556.328361] BM heartbeat64: deadbeefc0dec0de -> deadbeefc0dec0de
...
[ 3559.592342] BM heartbeat64: deadbeefc0dec0de -> deadbeefc0dec0de
[ 3559.796342] BM heartbeat64: deadbeefc0dec0de -> deadbeefc0dec0de
[ 3559.796353] Done loading DSP PEG kernel driver.
```


After updating the device tree (to ignore the other cpus):
```
[ 3830.588836] Loading DSP PEG driver... pi ≈ 3
[ 3830.588845] Reserving DSP Comms memory...
[ 3830.588879] Acquired resource [mem 0x10000000-0x100fffff] from name 'dsp_shared'.
[ 3830.588915] Finished executing ioremap for 'dsp_shared'.
[ 3830.589589] Acquired resource [mem 0x10100000-0x1b47efff] from name 'dsp_reserved'.
[ 3830.592092] Finished executing ioremap for 'dsp_reserved'.
[ 3830.592101] Successfully mapped memory!
[ 3830.592106] Writing and reading to shared memory as test...
[ 3830.592112] shared roundtrip: deadbeefc0dec0de
[ 3830.592119] Loading in bare-metal DSP firmware...
[ 3830.595036] dsp_reserved: (Hexdump omitted for brevity)
[ 3830.595182] Starting bare metal execution!
[ 3830.595188] spin-table[0x00000000000000e0] = 0x0000000010100000
[ 3830.796959] BM heartbeat64: deadbeefc0dec0de -> 00000000aaaaaaab
[ 3831.000953] BM heartbeat64: 00000000aaaaaaab -> 00000000aaaaaaac
[ 3831.204954] BM heartbeat64: 00000000aaaaaaac -> 00000000aaaaaaad
[ 3831.408951] BM heartbeat64: 00000000aaaaaaad -> 00000000aaaaaaae
[ 3831.613141] BM heartbeat64: 00000000aaaaaaae -> 00000000aaaaaaae
[ 3831.816960] BM heartbeat64: 00000000aaaaaaae -> 00000000aaaaaaaf
[ 3832.020973] BM heartbeat64: 00000000aaaaaaaf -> 00000000aaaaaab0
[ 3832.224991] BM heartbeat64: 00000000aaaaaab0 -> 00000000aaaaaab1
[ 3832.428965] BM heartbeat64: 00000000aaaaaab1 -> 00000000aaaaaab1
[ 3832.632969] BM heartbeat64: 00000000aaaaaab1 -> 00000000aaaaaab2
[ 3832.836997] BM heartbeat64: 00000000aaaaaab2 -> 00000000aaaaaab3
[ 3833.040978] BM heartbeat64: 00000000aaaaaab3 -> 00000000aaaaaab4
[ 3833.244975] BM heartbeat64: 00000000aaaaaab4 -> 00000000aaaaaab5
[ 3833.448976] BM heartbeat64: 00000000aaaaaab5 -> 00000000aaaaaab5
[ 3833.652991] BM heartbeat64: 00000000aaaaaab5 -> 00000000aaaaaab6
[ 3833.856979] BM heartbeat64: 00000000aaaaaab6 -> 00000000aaaaaab7
[ 3834.060977] BM heartbeat64: 00000000aaaaaab7 -> 00000000aaaaaab8
[ 3834.264978] BM heartbeat64: 00000000aaaaaab8 -> 00000000aaaaaab8
[ 3834.468985] BM heartbeat64: 00000000aaaaaab8 -> 00000000aaaaaab9
[ 3834.673005] BM heartbeat64: 00000000aaaaaab9 -> 00000000aaaaaaba
[ 3834.673029] Done loading DSP PEG kernel driver.
```

So it works!!! The bare metal code is alive and kicking! The strange thing is that if I reboot the rpi, I have to load in the kernel module twice for it to work, which is something I will definitely have to look into/troubleshoot, but for now at least I have concrete evidence that running bare metal code along side linux is possible!

## Next Steps

Now that I have confirmation that the linux + bare metal approach is feasible, I have a lot to do. I have to set up communication between the linux and bare metal sides, I have to implement the actual bare metal effects DSP, I have to write a user-facing front end program, and I have to confirm that I can talk to the ADC and DAC from bare metal.

These will be my next steps in order of priority:
- Troubleshoot and refine the bare-metal binary loading and execution; troubleshoot why I have to load the module twice for it to work properly.
- Clean up all the code I have written (device tree, kernel module, bare metal program, and UI, the last of which is coming soon) into one repository, with a commit in which I know everything works.
- Test out my ADC/DAC boards, which I have put together / soldered but not yet tested.
- Set up a communications channel between the linux core and the bare metal cores to enable changing effects / effects parameters in real time.
- Communicate with the ADC/DAC from bare metal (this may involve some more device tree shenanigans to block linux from claiming the I2S hardware peripheral hehe).
- NEXT MILESTONE: Write a basic/simple guitar effect with a few parameters (e.g. overdrive/distortion or echo), and build a first prototype with a screen, a few knobs/buttons, and input/output audio connections!


Executing bare metal code while linux runs on core 0 is a huge milestone for me and this project, since I wasn't even sure it was possible, and I am very proud that I was able to get it to happen! I am incredibly excited to get my hands dirty with the next steps; updates coming soon!
