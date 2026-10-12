# linux-in-a-makefile

**Linux 6.19 (with preemptive multitasking, networking, etc.) booting inside GNU Make.** Just one (very large) Makefile and `make`: there's no other source code or binaries or anything else. 

```
$ xz -d Makefile.xz
$ make
[    0.000000] Linux version 6.19.0-00258-gc06286b9aa8a (root@eternal) (clang version 22.1.0 (ESI), LLD 22.1.0) #3652 PREEMPT Sat Oct 10 21:43:51 PDT 2026
[    0.000270] printk: legacy bootconsole [subleq0] enabled
[    0.000948] Eternal Software Initiative Linux 6.19.0-00258-gc06286b9aa8a (https://eternal-software.org)
[    0.001329] CPU: ESI Subleq+ OISC
[    0.001542] Memory: 0x00000000 - 0x60000000 (1536 MB)
[    0.003543] Framebuffer reserved at 0x5fe70000, size 1638400 bytes
... watch a movie, then come back to your terminal ...
[    2.209407] This architecture does not have kernel memory protection.
[    2.209899] Run /init as init process

Welcome to Eternal Linux running on GNU Make!

BusyBox v1.38.0.git (2026-03-14 22:54:22 PDT) hush - the humble shell

~ # ping -A 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=1.845 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.687 ms
64 bytes from 127.0.0.1: seq=2 ttl=64 time=0.993 ms
...
```

Yes, that's a real Linux kernel booting to a shell. Every instruction is executed by GNU Make's macro expander: `$(let)`, `$(eval)`, `$(foreach)`, `$(subst)` and a lot of variable lookups. Make was never meant to do this, and it does it anyway.

## What it is

The [Eternal Software Initiative](https://eternal-software.org) builds Linux for **ESI Subleq+**, a one-instruction computer. Its only instruction is *subtract and branch if less than or equal to zero*. The reference VM (console version, no graphics) is about 50 lines of C. Since GNU Make is Turing complete, we can reimplement that VM entirely in GNU Make, and embed the whole kernel image in the Makefile itself:

- **The CPU** is written in Make. Words are 32-bit two's complement. While it runs, the VM holds each one as four hex bytes (`0c 00 00 00`), so Make's word functions split it for free. Addresses use the same bytes joined with dots (`.0c.00.00.00`), because they become parts of variable names. Subtraction uses precomputed byte tables with a borrow chain, so the arithmetic is done with variable lookups.
- **The memory** is 1.5 GB of address space as Make variables, loaded lazily. The 120 MB boot image is embedded as signed decimal integers, one variable per 64 KB page. A page is split into words the first time it's touched. Each word is converted from decimal on its first read.
- **The JIT**: yes, there's a just-in-time compiler in a Makefile. Each instruction is decoded once into specialised Make code, with the subtraction pasted in and the operands filled in. Common patterns like "clear this word and jump" get fast paths, and pointer operands are a single lookup, because every address the kernel uses is a whole word. Each compiled instruction expands to the address of the next one, so running instructions is just nested variable lookups, `$(INS$(INS$(…)))`. Writes over code turn the compiled code back into a stub that decodes it again, so self-modifying code works.
- **The timer interrupt** is checked between batches of 1,000 instructions and fires after 500 batches with interrupts enabled. That's close to the C VM's every 300,001 non-branching instructions, without counting each instruction.
- **The clock** is virtual: it advances 3 µs for every 1,000 instructions, starting at midnight UTC on 11 October 2026. To the guest, the whole boot takes about 2 seconds.
- **The console**: Make can only print whole lines, so each output byte is printed with `$(info)` as a line holding its octal code. The recipe turns those codes back into bytes with a shell loop using only built-ins (`read` and `printf`), so output appears a character at a time without starting a process per character. Input is read with `$(file </dev/stdin)`, which returns everything typed so far without waiting, so you can type ahead of the slow emulation.
- **The main loop** nests instruction steps 1,000 deep and loops 1000³ times, because Make has no unbounded loop. That allows 10¹² instructions, about 100 days of runtime.

The only external command is `stty`, which switches the terminal to raw, non-blocking input and back on exit. Everything else is Make and shell built-ins. The kernel image itself is embedded in the Makefile.

## Running it

You need **GNU Make 4.4 or newer**, because the VM relies on `$(let)`. On macOS, the system `make` is too old; use `brew install make` and run `gmake`.

```sh
make
```

When run from a terminal, it switches the terminal to raw, non-blocking input for you, the same `stty` settings the C VM needs, and restores them on exit or Ctrl-C.

You can tune the virtual clock using these two settings:

```sh
make NSPB=1000               # 1,000 ns per 1,000 instructions (default 3000)
make EPOCH=$(date +%s)       # boot at the real time (default 2026-10-11 00:00 UTC)
```

## How fast is it?

On my 2023 MacBook Pro, it manages roughly 100,000 instructions per second, against roughly 100 million per second for the C VM. The first boot lines appear within seconds, but a full boot to userspace takes hundreds of millions of instructions, which is an hour or more. Please be patient: this is Linux running on a build tool!

## Files

- `Makefile` is the whole thing: the VM, followed by the embedded kernel image and byte table (about 176 MB).

To regenerate the embedded image data after the `Image data` banner from an ESI VM capsule, see the Python one-liner in the header of `Makefile`.

## Why?

Because `make` has built Linux hundreds of thousands of times, but always for someone else. After thirty years of faithful service, it deserves to run Linux itself.