---
layout: post
author: Ronny
title: 'Atomic Ring Buffers'
---

After setting up the [shared memory struct with alignment checking]({% link Projects/DSP-PEG/_posts/2026-02-15-Shared-Memory-Struct.md %}), I decided to implement some atomic ring buffers so that I could pass messages back and forth between userspace and bare metal.

After doing some research, I came up with the following implementations (detailed descripsions below):

<div style="max-height: 500px; overflow-y: auto;" markdown="1">

```rust


pub trait BufPayload: Copy + Default {}
impl<T> BufPayload for T where T: Copy + Default {}

pub trait FIFOBuffer<T: BufPayload, E> {
    fn new() -> Self;
    fn push(&self, item: T) -> Result<(), E>;
    fn read_one(&self) -> Result<T, E>;
}

trait CheckPow2<const N: usize> {
    // N must be a power of 2...
    const CHECK_POW_2: () = assert!(N > 0 && (N & (N - 1)) == 0,);
}

#[repr(C, align(64))]
pub struct AtomicRingBufferSPSC<T: BufPayload, const N: usize> {
    write_index: CacheAligned<AtomicUsize>,
    read_index: CacheAligned<AtomicUsize>,
    buffer: [UnsafeCell<T>; N],
}

#[repr(C)]
struct StatusElement<T> {
    element: UnsafeCell<T>,
    status: AtomicBool,
}

#[repr(C, align(64))]
pub struct AtomicRingBufferMPSC<T: BufPayload, const N: usize> {
    write_index: CacheAligned<AtomicUsize>,
    read_index: CacheAligned<AtomicUsize>,
    buffer: [CacheAligned<StatusElement<T>>; N],
}

impl<T: BufPayload, const N: usize> CheckPow2<N> for AtomicRingBufferSPSC<T, N> {
    const CHECK_POW_2: () = assert!(N > 0 && (N & (N - 1)) == 0,);
}

impl<T: BufPayload, const N: usize> AtomicRingBufferMPSC<T, N> {
    const CHECK_POW_2: () = assert!(N > 0 && (N & (N - 1)) == 0,);
}

impl<T: BufPayload, const N: usize> FIFOBuffer<T, AtomicRingBufferErr>
    for AtomicRingBufferSPSC<T, N>
{
    fn new() -> Self {
        Self::CHECK_POW_2;

        Self {
            write_index: CacheAligned(AtomicUsize::new(0)),
            read_index: CacheAligned(AtomicUsize::new(0)),
            buffer: core::array::from_fn(|_| UnsafeCell::new(T::default())),
        }
    }

    fn push(&self, item: T) -> Result<(), AtomicRingBufferErr> {
        let write_idx = self.write_index.0.load(Ordering::Relaxed);
        let read_idx = self.read_index.0.load(Ordering::Acquire);

        let next_write_idx = (write_idx + 1) & (N - 1);

        if next_write_idx == read_idx {
            return Err(AtomicRingBufferErr::NoSpaceErr);
        }

        unsafe {
            let write_ptr = self.buffer[write_idx].get();
            write_ptr.write(item);
            // (*self.buffer.get())[write_idx] = item;
        }

        self.write_index.0.store(next_write_idx, Ordering::Release);

        Ok(())
    }

    fn read_one(&self) -> Result<T, AtomicRingBufferErr> {
        let read_idx = self.read_index.0.load(Ordering::Relaxed);
        let write_idx = self.write_index.0.load(Ordering::Acquire);

        if write_idx == read_idx {
            return Err(AtomicRingBufferErr::NoMessagesErr);
        }

        unsafe {
            let read_ptr = self.buffer[read_idx].get();
            let out = read_ptr.read();
            self.read_index
                .0
                .store((read_idx + 1) & (N - 1), Ordering::Release);

            Ok(out)
        }
    }
}

impl<T: BufPayload, const N: usize> FIFOBuffer<T, AtomicRingBufferErr>
    for AtomicRingBufferMPSC<T, N>
{
    fn new() -> Self {
        Self::CHECK_POW_2;

        Self {
            write_index: CacheAligned(AtomicUsize::new(0)),
            read_index: CacheAligned(AtomicUsize::new(0)),
            buffer: core::array::from_fn(|_| {
                CacheAligned(StatusElement {
                    element: UnsafeCell::new(T::default()),
                    status: AtomicBool::new(false),
                })
            }),
        }
    }

    fn push(&self, item: T) -> Result<(), AtomicRingBufferErr> {
        let write_idx = self.write_index.0.load(Ordering::Acquire);
        let read_idx = self.read_index.0.load(Ordering::Acquire);

        let next_write_idx = (write_idx + 1) & (N - 1);

        if next_write_idx == read_idx {
            return Err(AtomicRingBufferErr::NoSpaceErr);
        }

        let update_write_idx_result = self.write_index.compare_exchange(
            write_idx,
            next_write_idx,
            Ordering::Acquire,
            Ordering::Relaxed,
        );

        match update_write_idx_result {
            Ok(_) => (),
            Err(_) => return Err(AtomicRingBufferErr::AtomicConflictErr),
        }

        unsafe {
            let element_ptr = self.buffer[write_idx].element.get();
            element_ptr.write(item);
        }

        self.buffer[write_idx].status.store(true, Ordering::Release);

        Ok(())
    }

    fn read_one(&self) -> Result<T, AtomicRingBufferErr> {
        let read_idx = self.read_index.0.load(Ordering::Relaxed);

        if !self.buffer[read_idx].status.load(Ordering::Acquire) {
            return Err(AtomicRingBufferErr::NoMessagesErr);
        }

        let out;
        unsafe {
            let element_ptr = self.buffer[read_idx].element.get();
            out = element_ptr.read();
        }

        self.buffer[read_idx].status.store(false, Ordering::Release);

        self.read_index
            .0
            .store((read_idx + 1) & (N - 1), Ordering::Release);
        Ok(out)
    }
}
```
</div>

## Long Story Long

Feel free to skip the in-depth explanation and go straight to the [summary](#long-story-short).

### What is a Ring Buffer?

We can start by defining the ring buffer. A ring buffer is a [FIFO](https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)) buffer which uses a fixed block of contiguous memory to store buffer data. It is called a "ring buffer" because the contiguous block of memory is looped over and ovewritten, allowing for efficient reuse of memory; no need to allocate memory as elements are added. Because of this, ring buffers typically have fixed size (my implementations certainly will).

In order to achieve this behaviour, we can consider a naive implementation of a ring buffer, assuming a single thread will write and read from it (we'll deal with concurrency later).

In pseudocode, suppose we allocate some large chunk of memory as an array `buffer[N]` (where `N` is the number of elements in the array). We keep track of where along the array to read and write with two indices, `read_idx` and `write_idx`, which initially both start at 0.

To write an element, we check that there is space in the buffer (we define that there is space if `write_idx != read_idx-1` (mod `N`)), we write the element to `buffer[write_idx]`, and we increment `write_idx = (write_idx + 1) % N`, wrapping around modulo `N` so we stay in the array bounds.

To read an element, we check that there is an element to read (check that `read_idx != write_idx`), read the element at `buffer[read_idx]`, increment the read index with `read_idx = (read_idx + 1) % N` again wrapping around, and return the value we read.

In this way, we can write and read elements from the buffer in FIFO ordering.

### Thread Safety

This sort of buffer is useful for communication between threads in a program, but in order to use a ring buffer for this we need to make sure it is concurrency safe.

We could use the naïve approach of putting a lock on the whole buffer, but this becomes inefficient if threads are trying to read and write at the same time (they have to wait or fail).

It is possible to implement a ring buffer in a concurrency-safe way without locks, and this is called a _Lock Free/Atomic Ring Buffer_. By using atomic variables to properly synchronize the updating of the read and write indices, the reading and writing of the buffer data does not have to be atomic!

The implememtation specifics and performance varies between SPSC (single-producer single-consumer) and MPSC (multiple-producer single-consumer) implementations since we can make more synchronicity assumptions for the SPSC case (since only one thread is writing, whereas in the MPSC case many threads could be trying to write at the same time).

### SPSC

My implementation for SPSC largely follows [this implementation](https://github.com/cale-cmd/ultra-low-latency-ring-buffer/tree/main), though I made sure I understood what was going on before copying/translating things over.

My AtomicRingBufferSPSC type consists of three fields, read/write indices and the buffer itself (with UnsafeCells for interior mutability).

When writing to the SPSC implementation, we load the write index with `Relaxed` memory ordering, and the read index with `Acquire` ordering.

Then, we test whether there is space to write a new value, and if there is, we write at the `write_idx` element of the buffer array. We used `Acquire` for the above read index since we don't want the processor or compiler to reorder things so that we write to the buffer before comparing the indices to confirm that there is space.

After doing this, we atomically increase `write_idx` by one (with `Release` ordering) to make our data available to the reader and to prepare for writing to the next slot. In this case, we use `Release` ordering to guarantee that the write to the buffer is "seen" before the update to the index pointer, so that we never have a race condition where the reader is trying to read data as it is written.

For reading we follow a similar process. We load `read_idx` and `write_idx` with `Relaxed` and `Acquire` respectively, compare them to check that there is indeed a message to read, read the actual value at stored at `read_idx`, and atomically increment `read_idx`. We again use the `Acquire` ordering to ensure that we don't read before we are sure we may do so.

With these implementations of read and write, we can read and write from the ring buffer in a concurrency-safe but still efficient/fast manner, which means I can use it to communicate between bare-metal and userspace cores for DSP-PEG!


### MPSC

The MPSC implementation is similar to the SPSC implementation, with extra safeguards to allow for multiple threads trying to write at the same time (hence the name multiple producer).

If we want to retain lock-less efficiency, we can no longer just use a `write_idx` index to keep track of where to write next. Consider the scenario where two threads concurrently want to write data. If we just keep track of an index, the writing threads have to wait in line and write one at a time, since the write index can only be incremented at the end of a write in the SPSC implementation (lest the reader read while someone is writing).

We can do better than that though; by attaching a `status` atomic boolean to each element (where `status` is set to false by default) in the buffer, a writer can "claim" a slot to write to by incrementing the write index (as usual), and the data is only considered ready to read when `status` is atomically set to `true`. This way we avoid read/write race conditions while allowing multiple cores to simultaneously write to the buffer (no waiting in line).

To write to the MPSC buffer, we again load the `read_idx` and `write_idx`, this time both with `Acquire` since `write_idx` can be changed by other cores.

Then, we calculate the next write index, and make sure that there is space to write to the buffer.

Then we use `compare_exchange` to update `write_idx` only if it hasn't changed in between (which could happen if a different thread is trying to write at the same time).

If we successfully increment the write index, then we now have exclusive write access to the value at the original write index (before incrementing it), so we write our value there and atomically set `status` to `true` (with `Release` ordering so that our write happens before anyone tries to read).

## Long Story Short
 
To summarize, I implemented both SPSC and MPSC (single/multiple producer single consumer) atomic ring buffers, which allow lock-free writing and reading. The MPSC version has a few extra details and is less efficient/fast than the SPSC due to having to cover cases where many threads are trying to write at the same time.

These should allow for fast and safe communication between the bare metal cores and the userspace program running on the RPi for DSP-PEG!

### Cool Details

If the array used by a ring buffer has size `N` which is a power of 2, we can use a much faster bit operation to get indices modulo `N`: bitwise-and with `N-1` gives us the number modulo `N`. This is something I thought would be good to include in my implementation, but I still wanted to be able to choose the size of the buffer with generics...

What I came up with was the following: I made a constant `const CHECK_POW_2: () = assert!(N > 0 && (N & (N - 1)) == 0,);` which `panic!`s if `N` is not a power of 2. By then putting `Self::CHECK_POW_2;` in the `new()` function of my SPSC and MPSC implementations, the compiler evaluates this at compile time and fails to build if `N` is specified to be something other than a power of 2 (which I tested to confirm that it works).

This way we can have a nice optimization without sacrificing too much flexibility (I can still choose buffer size as long as its a power of 2), and without sacrificing correctness (if the code builds, the buffer size is a power of 2 and the math works as expected).


# Testing the Buffers

I tested the SPSC buffer first by adding it to the `SharedMem` struct:

```rust
#[repr(C, align(64))]
pub struct SharedMem {
    core_status: CacheAligned<[AtomicU8; 3]>,
    message_bufs: [CacheAligned<AtomicRingBufferMPSC<BaremetalMessage, 1024>>; 3],
}
```

Where I made the `BareMetalMessage` type just to test out the buffers for now:
```rust
#[repr(u8)]
#[derive(FromRepr, Clone, Copy, Default, Debug, PartialEq, Eq)]
pub enum BaremetalMessage {
    #[default]
    Ping,
    TestSendingAU8Lol(u8),
}
```

I then set up the bare metal code to send messages, which I was able to receive from the userspace side:

```
2026-02-16 23:25:02.383243511 [INFO ] <userspace::pedal_controller::rpi:42>:Message: Ping
2026-02-16 23:25:02.383421844 [INFO ] <userspace::pedal_controller::rpi:42>:Message: TestSendingAU8Lol(182)
2026-02-16 23:25:02.383600646 [INFO ] <userspace::pedal_controller::rpi:42>:Message: TestSendingAU8Lol(183)
2026-02-16 23:25:02.383688355 [INFO ] <userspace::pedal_controller::rpi:42>:Message: TestSendingAU8Lol(184)
2026-02-16 23:25:02.383900334 [INFO ] <userspace::pedal_controller::rpi:44>:No new messages.
```

In this case, the baremetal side sends four messages in rapid succession, and the userspace side prints all new messages every second or so (by calling `read` in a loop until we get a `NoMessagesErr`).

So it works!!

However, when I tested the MPSC implementation, the bare metal program would just crash upon trying to write (which I know since I was reading the `core_status` field of `SharedMem`, which the bare metal side should be updating).

The first thing I tried to do to troubleshoot this was call `write` and `read` from the userspace side, and through this I was able to determine that the program was crashing at the atomic `compare_exchange()` call...

This seemed odd to me, but after doing some research, this seems to be because the memory is not mapped as cacheable, which is required for compare exchange!

On the bare metal side, this means that I have to actually configure the MMU (which I have to do anyways for reasonable DSP performance, as caching is disabled until you turn the MMU on). I also need to possibly modify the device tree and make sure that my `mmap` call has the right flags for cacheable memory when I `mmap` the shared memory chunk into my userspace rust program.

So next up I'll figure out how to configure the MMU with assembly!
