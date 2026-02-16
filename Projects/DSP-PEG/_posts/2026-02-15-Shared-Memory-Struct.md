---
layout: post
author: Ronny
title: 'Shared Memory Struct'
---

It's been a while since I've written an update, and I have gotten some things done since last time! The git commit hash relevant to this post is `bb266ae`.

## Shared Memory Architecture

As a quick refresher, I have the Raspberry Pi Zero 2 W set up with a modified device tree file which hides cores 1-3 from Linux, and also reserves two chunks of memory, one for bare metal memory space (for executing baremetal programs), and one chunk for shared memory between userspace and bare metal.

I ran some tests using the "mmap" syscall from userspace, and I was successfully able to read the shared memory region directly from the userspace program; I could read the "heartbeat" [I set up previously]({% link Projects/DSP-PEG/_posts/2025-10-10-Hello-World-on-Linux-and-Bare-Metal-Simultaneously.md %}), which is just an incrementing counter being updated by bare metal.

This is nice, because it confirms that I can just implement rust to rust communication (userspace to baremetal and back) without needing to implement an "in between" layer in the kernel module.

### Keeping track of Shared Layout

It is important that the userspace and bare-metal programs agree on what is stored at which address in the shared memory layout for back and forth communication to work; otherwise, one side might write or read data somewhere where the other side doesn't expect.

This could be done by hand, i.e. writing code to instantiate/create pointers to data structures at identical addresses on either side, but this is a fragile approach, since any changes I want to make to the layout later would involve re-writing and re-checking code, which is error prone and tedious.

In the spirit of not repeating myself, I decided that I would define a single `#[repr(C)]` struct in a crate that both userspace and bare metal could access, and as long as the layout of the struct itself remains the same when compiling userspace and bare-metal, then instantiating the struct once on both sides should be enough to ensure agreement on where things are stored.

To do this, I created a library crate `./lib/common` which contains things relevant to both the userspace and bare-metal programs. This includes the shared memory struct, but also includes things like the physical address for the shared memory region (so that in the event that I want to change the address, I just need to change it in one place).

Within the `common` crate, I made a module called `shared_mem`, and in that module I defined the `SharedMem` struct; this is the monolithic struct which will contain all shared datastructures (e.g. message ringbuffers, status flags) with the goal of ensuring agreement on alignment between userspace and bare metal.

I also made a wrapper type so that I could easily make a type cache-aligned:

```rust
#[repr(C, align(64))]
pub struct CacheAligned<T>(T);

impl<T> core::ops::Deref for CacheAligned<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}
```

By implementing the [Deref](https://doc.rust-lang.org/std/ops/trait.Deref.html) trait, the wrapper is basically transparent in the sense that you can still call methods as `instance.method()` if `instance` is of type `CacheAligned<T>` and `method()` is implemented on `T`.

My test `SharedMem` struct looks something like this:
```rust
#[repr(C, align(64))]
pub struct SharedMem {
    core_status: CacheAligned<[AtomicU8; 3]>,
    baremetal_message_buf: CacheAligned<AtomicRingBuffer<types::BaremetalMessage, 1024>>,
}
```

where `core_status` is basically an array of atomic u8 status flags (one per core), and `baremetal_message_buf` is an atomic ring buffer for sending messages back and forth. The latter I hadn't really implemented or tested at the time, I just wanted some meat in my struct to test the alignment.

I was still not totally sure that `#[repr(C)]` was enough to guarantee identical struct layouts, especially since I am compiling for different targets (`aarch64-unknown-linux-gnu` for userspace and `aarch64-unknown-none` for baremetal), so I looked into how I could check the struct layouts after compilation to truly guarantee correctness.


### Checking Struct Layout After Compiling

I found a linux command `pahole` which can be used to analyze debug information from ELF files, including struct field layout. I turned on debug flags for the rust release target, and ran some tests on the output binaries for both baremetal and userspace. The output looked something like this:

```rust
// > pahole -C SharedMem -c 64 userspace
struct SharedMem {
private:

	struct CacheAligned<[core::sync::atomic::AtomicU8; 3]> core_status __attribute__((__aligned__(64))); /*     0    64 */

	/* XXX last struct has 61 bytes of padding */

	/* --- cacheline 1 boundary (64 bytes) --- */
	struct CacheAligned<common::shared_mem::types::AtomicRingBuffer<common::shared_mem::types::BaremetalMessage, 1024>> baremetal_message_buf __attribute__((__aligned__(64))); /*    64  1152 */

	/* size: 1216, cachelines: 19, members: 2 */
	/* paddings: 1, sum paddings: 61 */
	/* forced alignments: 2 */
} __attribute__((__aligned__(64)));

// > pahole -C SharedMem -c 64 baremetal
struct SharedMem {
private:

	struct CacheAligned<[core::sync::atomic::AtomicU8; 3]> core_status __attribute__((__aligned__(64))); /*     0    64 */

	/* XXX last struct has 61 bytes of padding */

	/* --- cacheline 1 boundary (64 bytes) --- */
	struct CacheAligned<common::shared_mem::types::AtomicRingBuffer<common::shared_mem::types::BaremetalMessage, 1024>> baremetal_message_buf __attribute__((__aligned__(64))); /*    64  1152 */

	/* size: 1216, cachelines: 19, members: 2 */
	/* paddings: 1, sum paddings: 61 */
	/* forced alignments: 2 */
} __attribute__((__aligned__(64)));
```

The numbers at the end of the lines describing members of the struct, e.g. `/*  0  64  */` for `core_status` describe the alignment of that field relative to the overall `SharedMem` struct. As we can see, since we used the `CacheAligned` wrapper for both core_status and the message buffer, there are 3 bytes of actual data and 61 bytes of padding for core_status.

I added a bash script to the postBuild section of my nix flake that builds both the userspace and baremetal which runs the `pahole` command on both binaries, compares the alignment of the fields, and makes the build fail if there are mismatches. The script looks like this:
```bash
touch userspace.shared_layout_bare
touch baremetal.shared_layout_bare

pahole -C SharedMem -c 64 $out/bin/DSP-PEG-ui > userspace.shared_layout_struct 2> /dev/null || true
pahole -C SharedMem -c 64 $out/baremetal/baremetal-elf > baremetal.shared_layout_struct 2> /dev/null || true 

cat userspace.shared_layout_struct | grep -oE "\/\*\s*[0-9]+\s*[0-9]+\s*\*\/$" >> userspace.shared_layout_bare
cat baremetal.shared_layout_struct | grep -oE "\/\*\s*[0-9]+\s*[0-9]+\s*\*\/$" >> baremetal.shared_layout_bare

echo "Checking shared memory layout matching..."
if diff -u userspace.shared_layout_bare baremetal.shared_layout_bare; then
  echo "Shared Memory layout matches, proceeding."
else
  echo "ERROR: Shared memory struct layout does not match!"
  echo "bare metal:"
  cat baremetal.shared_layout_struct
  echo "userspace:"
  cat userspace.shared_layout_struct
  exit 1
fi
```

The script first writes the pahole output to text files, then uses `grep` to keep only the lines describing members of the structs, and finally uses `diff` to compare the baremetal and userspace alignments.

I tested comparing to a garbage file, and the build indeed failed as intended.

After implementing this, I can trust the alignment of the struct fields, so I don't need to worry about this as a potential source of errors (when debugging in the future, I will not have to worry about struct field alignment!).

Next up I am going to write about implementing atomic ring buffers. It turns out I have to turn on the MMU (which is something I had forgotten about) to use certain atomic instructions and to enable using the cache, both of which I am going to need...
