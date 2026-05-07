# Could We Just Run `kernel.o` on an i386 Machine?

A natural follow-up to [build-artifacts.md](build-artifacts.md): if `kernel.o` is a fully linked, valid i386 ELF executable, could we copy it to a 32-bit Linux box and run `./kernel.o`?

This document answers that question, and uses it as a lens onto the difference between *being a valid executable file* and *being something an OS can actually run*.

---

## TL;DR

**No.** `kernel.o` is a structurally valid i386 ELF executable, but it can't be run as a normal program on a Linux i386 machine. It's a *kernel* — it expects to *be* the OS, with nothing underneath it. There are four hard blockers, each of which would be enough by itself to kill the program before it ran a single useful instruction.

---

## The four hard blockers

### Blocker 1 — The entry point is 16-bit real-mode code

Look at `start.S:19`:

```asm
.code16
```

The very first instruction the kernel runs is **16-bit real-mode** assembly. Real mode is the CPU mode the i386 boots into — it has:

- 16-bit registers (`%ax`, not `%eax`).
- Segment-shifted addressing (linear address = `segment * 16 + offset`).
- No memory protection, no privilege rings, no paging.
- Different instruction encoding from 32-bit code.

When Linux's ELF loader runs a program, **the CPU is already in 32-bit (or 64-bit) protected mode**. Executing the bytes of a `.code16` instruction sequence in 32-bit mode would decode them as totally different instructions — gibberish that would fault almost immediately.

The `start.S` code we annotated (lines 126–151) does the real-mode → protected-mode transition itself. It expects to *receive* a CPU that's already in real mode, the way the BIOS leaves it after running the boot sector. Linux can't deliver that, because Linux is *already* running in protected mode by the time it's launching your program.

You can prove the 16-bit-ness yourself:

```sh
# Disassemble as 16-bit code → makes sense
objdump -d -M i8086 src/threads/build/kernel.o | head -30

# Disassemble as 32-bit code → nonsense
objdump -d         src/threads/build/kernel.o | head -30
```

The same bytes produce sensible instructions in one mode and gibberish in the other. An i386 CPU in 32-bit mode would do the gibberish version.

### Blocker 2 — It's linked at `0xC0020000` — kernel space

The linker script `kernel.lds.S:9` places the kernel at `LOADER_PHYS_BASE + LOADER_KERN_BASE = 0xC0020000`. Every absolute address inside `kernel.o` (function pointers, jump tables, the addresses patched into `call` instructions, etc.) was computed assuming this layout.

On a 32-bit Linux system, the address space is split:

```
   0x00000000 ──────────────────────────────  ┐
                                              │
              user space (3 GB)               │  Programs you launch
                                              │  live in here.
   0xBFFFFFFF ──────────────────────────────  ┘
   0xC0000000 ──────────────────────────────  ┐
                                              │
              Linux kernel space (1 GB)       │  Where Pintos's
                                              │  kernel.o is linked.
   0xFFFFFFFF ──────────────────────────────  ┘
```

User programs cannot allocate or load code into the top 1 GB — that range is reserved for the Linux kernel itself. When Linux's ELF loader sees a program asking to be mapped at `0xC0020000`, it either refuses (`ENOMEM`) or maps it elsewhere, which would break every absolute address baked into the binary.

This is also *why* Pintos has to set up its own page tables in `start.S:81-124` and create the higher-half mapping by hand — it doesn't have an OS underneath to do this for it. On a Linux box, Linux has already done its own mapping for its own kernel at those addresses, and there's no room for Pintos to coexist.

### Blocker 3 — It uses ring-0 instructions

The kernel calls a long list of **privileged instructions** that only work when the CPU is at ring 0 (kernel privilege level):

- `outb`, `inb` — port I/O (every device driver uses these).
- `cli`, `sti` — disable/enable interrupts.
- `lgdt`, `lidt` — load segment / interrupt descriptor tables.
- `mov %eax, %cr0`, `mov %eax, %cr3` — modify control registers (paging, protection bits).

Linux runs your processes at **ring 3** (user level). Executing any of those privileged instructions at ring 3 raises a **`#GP` (general protection fault)**, which Linux delivers to your process as `SIGSEGV`. So even if you somehow got past blockers 1 and 2, the very first `cli` (early in `start.S`) or `outb` (in any device driver) would kill the process instantly.

This is fundamentally why kernels exist: someone has to be allowed to talk to hardware, and the CPU enforces that "someone" is exactly one entity per machine.

### Blocker 4 — It expects no OS underneath it

A normal Linux executable expects a substantial *runtime environment* to be set up before it starts:

- A C library (`libc`) loaded into the address space.
- `main` to receive `argc`, `argv`, `envp` on the stack.
- A heap (`brk`/`mmap`) it can grow.
- `stdin`/`stdout`/`stderr` already opened.
- `int $0x80` / `syscall` / `sysenter` to be the way to talk to the world.
- Signal handlers, working `fork`/`exec`, etc.

Pintos's `main` (`init.c:77`) expects almost the opposite:

- A working stack (which `start.S` sets up before calling it).
- BSS not yet cleared (it clears it itself at `init.c:147`).
- No paging beyond what `start.S` set up.
- No file descriptors, no syscall mechanism — there's no kernel to call into, *it is the kernel*.
- I/O happens by writing directly to hardware ports (the 16550 UART, the VGA framebuffer at `0xB8000`), **not** through `write(2)`.

When Pintos's `printf` runs, it resolves not to glibc's `printf` but to `lib/stdio.c`'s own implementation, which eventually calls `serial_putc`, which calls `outb(0x3F8, byte)` — and we're back to Blocker 3.

There is no Linux service `kernel.o` could call even if it tried. Its world begins and ends with the i386 instruction set.

---

## A more interesting variant: *bare-metal* i386, not Linux i386

What if you didn't try to run it under Linux, but on a real i386 machine, booting it directly?

The answer is more nuanced:

### `kernel.o` alone, no.

A real PC BIOS doesn't know how to parse an ELF file. The BIOS reads the first 512 bytes of a disk into `0x7c00` and jumps. It has no concept of `e_entry`, sections, program headers, or symbol tables. ELF parsing is firmly in software territory, not firmware territory.

Even if you somehow handed the BIOS the bytes of `kernel.o`, it would just execute the ELF header as machine code (the magic bytes `\x7fELF` and so on) — which would fault essentially immediately.

### `kernel.o` + `loader.bin`, yes — that's exactly the Pintos boot flow.

`loader.bin` is the bridge between the BIOS and the kernel:

1. The BIOS loads `loader.bin` (the MBR) into `0x7c00`.
2. The loader's code parses the ELF header sitting in the first sector of the kernel partition.
3. It pulls `e_entry` out of byte offset `0x18` (`loader.S:163-168`).
4. It copies the kernel from the partition to physical `0x20000`.
5. It far-jumps to `e_entry`, handing the still-real-mode CPU to `start.S`.

In other words, **`loader.bin` is Pintos's own custom ELF loader, hand-coded to fit in 512 bytes**. It plays the same role for Pintos that `fs/binfmt_elf.c` plays for Linux user programs — but with completely different rules:

| | Linux's ELF loader (`binfmt_elf.c`) | Pintos's `loader.bin` |
|---|---|---|
| Loads programs into | user space (`< 0xC0000000`) | low physical RAM (`0x20000`) |
| Leaves the CPU in | 32-bit/64-bit protected, ring 3 | 16-bit real mode, ring 0 |
| Handles paging | yes (via the OS kernel's page tables) | no — kernel does it itself |
| Code size | ~1500 LOC of C | 512 bytes of asm |

Same file format (ELF), totally different execution contracts.

---

## A small experiment

If you're curious, you can try this on any 32-bit-capable Linux machine:

```sh
chmod +x src/threads/build/kernel.o
./src/threads/build/kernel.o
```

You'll get one of:

- `cannot execute binary file: Exec format error` — Linux's ELF loader rejected it for not having loadable program headers in user space, or for some other structural reason.
- `Segmentation fault` — the loader did try, and the CPU faulted on either the first `.code16` instruction or the first ring-0 instruction.

Either way, the kernel doesn't run. Try this with the various inspection commands from `build-artifacts.md` to dig into *why* — `readelf -l kernel.o`, `objdump -h kernel.o`, etc.

---

## The mental model

A "valid ELF executable" is **necessary but not sufficient** for "can run on machine X." There are several extra contracts a binary has to satisfy beyond format-validity, and those contracts are different for different runtime environments:

| Contract | Linux user binary | Pintos kernel |
|---|---|---|
| CPU mode at entry | 32/64-bit protected, ring 3 | **16-bit real mode**, ring 0 |
| Virtual address range | `< 0xC0000000` (user space) | `>= 0xC0000000` (kernel space) |
| Privileged instructions | forbidden (would `#GP`) | required (talks to hardware directly) |
| Talks to the world via | syscalls (`int 0x80`/`syscall`) | port I/O (`outb`/`inb`) |
| Stack/argv/env at entry | provided by Linux | nothing — the kernel sets up its own |
| Loaded by | Linux's `binfmt_elf` | Pintos's hand-rolled `loader.S` |
| Coexists with other code | yes (under the kernel) | no — owns the whole machine |

Same file format. Totally different worlds. `kernel.o` is well-formed for the kernel contract but ill-formed for the user-space contract — and there's no fix for that short of rewriting it as something fundamentally different.

This is also why "an OS" and "a program" feel so different even though they're both "just code." The OS exists *precisely to provide* the runtime environment that ordinary programs need. Without one, your program has to be the OS — and being the OS means meeting the i386 boot environment on its own raw, real-mode, ring-0 terms.

---

## Cross-references

- [build-artifacts.md](build-artifacts.md) — what `kernel.o` *is*, and how it's produced.
- [loader-annotated.md](loader-annotated.md) — how the boot sector finds and loads the kernel ELF.
- [start-annotated.md](start-annotated.md) — what the kernel does in its first hundred-odd instructions to lift itself into a usable runtime.
- [qemu-pintos-seam.md](qemu-pintos-seam.md) — the wider picture: where the kernel ends and the (emulated) hardware begins.
