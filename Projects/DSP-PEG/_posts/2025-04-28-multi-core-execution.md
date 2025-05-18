---
layout: post
author: Ronny
title: Multi-Core Execution
---

I spent some time figuring out how to get concurrent (simulatneous) execution in all
4 cores at once. The relevant git commit hash at the time of writing is f9a0f3e.

## Concurrent Execution

What I ended up doing is creating functions "coreN_main()" for cores 1, 2, and 3 (core
0 is what starts executing code at startup). Using `#[no_mangle]` and `pub extern "C"`,
as well as some modifications to my linker script, these functions are placed at known
memory addresses.

Then, I followed suggestions from
[this RPi forum post](https://forums.raspberrypi.com/viewtopic.php?t=273010), which
says that writing function addresses to 0xE0, 0xE8, and 0xF0, (for cores 1, 2, and 3
respectively), and then emitting the `sev` instruction will cause the cores to start
executing the functions at said addresses.

Before the bss section in my linker script, I added the following:
```
.core1_main 0x90000 : {
    __core1_main_start = .;
    KEEP(*(.core1_main))
}

.core2_main 0xA0000 : {
    __core2_main_start = .;
    KEEP(*(.core2_main))
}

.core3_main 0xB0000 : {
    __core3_main_start = .;
    KEEP(*(.core3_main))
}
```

Then, in my `main.rs` I wrote the functions, and I also created a `start_cores()` function:
```rust
#[no_mangle]
#[link_section = ".core1_main"]
pub extern "C" fn core1_main() -> ! {
    unsafe {
        for _ in 1..(4670000) {
                asm!("nop");
        }
        core::ptr::write_volatile(UART0_DR, b'A' as u32);
        loop {
            for _ in 1..1010000 {
                    asm!("nop");
            }
            core::ptr::write_volatile(UART0_DR, b'1' as u32);
        }
    }
}

#[no_mangle]
#[link_section = ".core2_main"]
pub extern "C" fn core2_main() -> ! {
    unsafe {
        for _ in 1..(4500000) {
                asm!("nop");
        }
        core::ptr::write_volatile(UART0_DR, b'B' as u32);
        loop {
            for _ in 1..1000000 {
                    asm!("nop");
            }
            core::ptr::write_volatile(UART0_DR, b'2' as u32);
        }
    }
}

#[no_mangle]
#[link_section = ".core3_main"]
pub extern "C" fn core3_main() -> ! {
    unsafe {
        for _ in 1..(6780000) {
                asm!("nop");
        }
        core::ptr::write_volatile(UART0_DR, b'C' as u32);
        loop {
            for _ in 1..2300700 {
                    asm!("nop");
            }
            core::ptr::write_volatile(UART0_DR, b'3' as u32);
        }
    }
}

unsafe fn start_cores() {

    // addresses to write fn pointers to.
    const CORE1_START: *mut u32 = 0xE0 as *mut u32;
    const CORE2_START: *mut u32 = 0xE8 as *mut u32;
    const CORE3_START: *mut u32 = 0xF0 as *mut u32;

    // write function ptrs to addresses:
    core::ptr::write_volatile(CORE1_START, 0x90000);
    core::ptr::write_volatile(CORE2_START, 0xA0000);
    core::ptr::write_volatile(CORE3_START, 0xB0000);
    
    core::arch::asm!("sev");
}
```

So when `start_cores()` is called, the addresses where the functions are stored are
written to 0xE0, 0xE8, and 0xF0, and then the `sev` instruction is emitted, telling the
cores to turn on. Currently, all the core wise main functions do is write characters to
the UART peripheral over and over. Each core will write its core number as a character
at different time periods. Sure enough, reading the UART from my laptop shows:
```
HBA2121C2121321213212123121213212132121231212132121321
```

Printing the 'H' character is part of my main function, and then 'B', 'A', and 'C' are
printed by each core (see the above functions) after waiting for different time periods.
Then, the characters '1', '2', and '3' start appearing at different intervals. This shows
that the functions are indeed being executed in parallel by the separate cores!! From
here, the stack pointer for each core should be initialized, and from there we are free
to execute whatever rust code we want on each core!

## Stack Pointer Setup

The stack pointer needs to be set up in a valid memory location before trying to use the
stack in rust (i.e. calling functions, local variables, etc).

I added a section to my linker script (after the bss section) to reserve a 1MB stack per core:

```
. = ALIGN(4096);
. = . + 1024 * 1024;
_stack_start_0 = .;
. = . + 1024 * 1024;
_stack_start_1 = .;
. = . + 1024 * 1024;
_stack_start_2 = .;
. = . + 1024 * 1024;
_stack_start_3 = .;
. = . + 1024 * 1024;
```

And then in each of the `coreN_main()` functions, I added some inline assembly to set the
stack pointer. In each of the core main functions:

```rust
asm!(
    "
    ldr x0, =_stack_start_N
    mov sp, x0
    ",
    options(nostack, nomem),
);
```

Where `N` is replaced with 1-3 depending on the core. This should initialize the stack
pointer to the correct memory addresses. This follows a similar structure to the
example [found here](https://developer.arm.com/documentation/dui0471/g/Embedded-Software-Development/Stack-pointer-initialization).

So now I have successfully run code on all 4 cores simultaneously, and the stack pointers
should be set up correctly. Each stack has 1MB allocated to it, which should be waaay more
than enough for our purposes (as we are writing pretty low level code, so there will not
be excessive function nesting).

Now that multi-core execution works, the mailbox system for inter core comms needs to be
tested, and a benchmarking framework can be created.
