# The Qemu ⇄ Pintos Seam

## 1. Boot Handshake — from BIOS to C

Qemu boots an emulated i386 exactly like a real PC. The "kernel" you build is an ELF image written into a partition on a virtual disk image (`build/os.dsk`). Qemu loads the **first 512 bytes of that disk** (the MBR — Pintos's `loader.S`) into physical address `0x7c00` and jumps there in 16-bit real mode.

Sequence of handoffs:

| Stage | File | What happens |
|---|---|---|
| BIOS (Qemu's SeaBIOS) → Loader | `threads/loader.S:23-27` | BIOS jumps to `0x7c00`. Loader sets `ds/ss=0`, `esp=0xf000`. |
| Loader scans partitions | `loader.S:50-86` | Walks MBR partition table for type `0x20` (Pintos kernel). |
| Loader loads kernel image | `loader.S:113-148` | Reads up to 512 KB (1024 sectors) starting at physical `0x20000` (128 KB) using BIOS `int 0x13` ext-read. |
| Loader → kernel entry | `loader.S:163-168` | Far-jumps to the ELF entry address (still real-mode). |
| `start` in kernel | `threads/start.S:23` | First kernel-side instruction. Sets segment regs, queries memory size, enables A20, builds an identity+higher-half page directory, loads GDT, flips CR0 PE+PG, far-jumps into 32-bit protected mode. |
| Protected-mode trampoline | `start.S:180` | Executes `call main`. |
| **First C function** | `threads/init.c:77` — `int main (void)` | This is the answer to your question. `main()` clears BSS, initializes the thread system, console, paging, interrupts, timer, etc. |

So the chain is: **SeaBIOS → `loader.S` → `start.S` → `init.c:main()`**.

## 2. I/O Port Communication — `outb`/`inb`

The wrappers live at `threads/io.h:66-70`:

```c
static inline void outb (uint16_t port, uint8_t data) {
  asm volatile ("outb %b0, %w1" : : "a" (data), "Nd" (port));
}
```

These compile to the x86 `OUT`/`IN` instructions. When the guest CPU executes them, Qemu's CPU emulator traps them and dispatches to the *device model* registered on that port number — there is no real hardware involved.

**Concrete example: the timer (8254 PIT).** `devices/pit.c:79-81` programs Qemu's emulated 8254 to fire at `TIMER_FREQ` Hz:

```c
outb (PIT_PORT_CONTROL, (channel << 6) | 0x30 | (mode << 1));  // port 0x43
outb (PIT_PORT_COUNTER (channel), count);                       // port 0x40
outb (PIT_PORT_COUNTER (channel), count >> 8);
```

`devices/timer.c:38-39` then wires it up: `pit_configure_channel(0, 2, TIMER_FREQ); intr_register_ext(0x20, timer_interrupt, ...)`.

**Another example: the 16550A UART** (`devices/serial.c:203-205`):

```c
while ((inb (LSR_REG) & LSR_THRE) == 0)   // read port 0x3FD until "transmit holding empty"
  continue;
outb (THR_REG, byte);                      // write byte to port 0x3F8
```

That `inb/outb` pair is *the* seam — it's exactly the place where a Pintos instruction stops being computation and becomes a Qemu device-model side effect.

## 3. The Console Path — `printf` to your terminal

When you run `pintos --qemu --`, the launcher (`utils/pintos`) invokes Qemu with `-serial stdio`, which connects the guest's COM1 to the host's stdout — i.e. your SSH session on the Droplet. The kernel doesn't know or care; it just writes bytes to UART port `0x3F8`.

```
printf("Boot complete.\n")              lib/stdio.c
        │
        ▼  vsnprintf → __vprintf → putchar_func
console_putc / vprintf_helper           lib/kernel/console.c:142-146
        │
        ▼
putchar_have_lock(c)                    lib/kernel/console.c:185
        ├── serial_putc(c)              devices/serial.c:99
        │       └── putc_poll: poll LSR_THRE, then  outb(0x3F8, byte)
        │                                                │
        │                                                ▼
        │                                       Qemu UART model
        │                                                │
        │                                                ▼
        │                                       host stdio (the -serial stdio chardev)
        │                                                │
        │                                                ▼
        │                                       your DigitalOcean SSH terminal
        │
        └── vga_putc(c)                 devices/vga.c   (writes to text-mode framebuffer at 0xB8000)
                                          → invisible over SSH; matters in graphical mode
```

So every character actually goes out **two** paths in parallel; the one you see in your SSH session is the serial path.

## 4. Interrupt Mapping — Qemu signaling the kernel

Interrupts cross the seam in the **opposite direction** of `outb`: Qemu's device models raise IRQs on the emulated 8259 PIC, the PIC vectors them through the IDT, and the kernel's handler runs.

Setup, in order:

1. **`pic_init()`** — `threads/interrupt.c:238-258` — remaps the legacy PC IRQs (which originally collided with CPU exception vectors 0x08–0x0F) to vectors `0x20–0x2F` by writing ICW1–ICW4 to PIC ports `0x20/0x21` (master) and `0xA0/0xA1` (slave).
2. **`intr_init()`** populates the IDT with 256 stub entries from `intr-stubs.S`. Each stub pushes the vector number and jumps to a common `intr_entry` that builds an `intr_frame` and calls C `intr_handler`.
3. **`timer_init()`** — `devices/timer.c:39` — calls `intr_register_ext(0x20, timer_interrupt, "8254 Timer")`, storing `timer_interrupt` in `intr_handlers[0x20]`.
4. **`thread_start()`** runs `sti`, enabling interrupts.

Runtime path of one tick:

```
Qemu's 8254 model decrements its counter
        │
        ▼ raises IRQ0 on Qemu's 8259 model
Qemu injects vector 0x20 into the guest CPU
        │
        ▼ CPU pushes EFLAGS/CS/EIP, looks up IDT[0x20]
intr20_stub  (intr-stubs.S, generated)
        │  pushes 0 (no error code) and 0x20 (vec)
        ▼
intr_entry   builds intr_frame, calls →
        │
        ▼
intr_handler(frame)               threads/interrupt.c:345
        │   dispatches via intr_handlers[0x20]
        ▼
timer_interrupt(frame)            devices/timer.c:170-175
        │   ticks++;  thread_tick();   ← may set yield_on_return
        ▼
back in intr_handler: pic_end_of_interrupt()  (outb 0x20 → port 0x20: EOI)
        │
        ▼ iret → resumes interrupted code (or schedules a new thread)
```

The same pattern carries the keyboard (vector `0x21`), serial (`0x24`), and IDE (`0x2E/0x2F`) — only the device model on the Qemu side and the registered handler on the Pintos side change.

## 5. Layered ASCII view

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                Physical Droplet (KVM guest on DigitalOcean)                 │
│  Real x86_64 silicon · Linux host kernel · your SSH session ── stdout ───┐  │
└──────────────────────────────────────┬──────────────────────────────────┘   │
                                       │ host syscalls (write/read on tty)   │
┌──────────────────────────────────────▼──────────────────────────────────┐   │
│                         qemu-system-i386 process                        │   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────┐  │   │
│  │ TCG / KVM    │  │ 8259 PIC     │  │ 8254 PIT     │  │ 16550 UART  │◀─┼───┘ -serial stdio
│  │ CPU emulator │  │ model        │  │ model        │  │ model       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬──────┘  │
│         │ traps on        │ INT vector      │ IRQ0            │ port    │
│         │ IN/OUT/IRET     │ injection       │                 │ 0x3F8   │
└─────────┼─────────────────┼─────────────────┼─────────────────┼─────────┘
          │                 │                 │                 │
          │  ── port I/O ── seam (the line everything crosses) ──
          │                 │                 │                 │
┌─────────▼─────────────────▼─────────────────▼─────────────────▼─────────┐
│                  Pintos drivers  (src/devices, src/threads/io.h)        │
│  pit.c  ── outb 0x40/0x43           timer.c ── intr 0x20 handler        │
│  serial.c ── inb/outb 0x3F8..3FF     kbd.c   ── intr 0x21 handler       │
│  interrupt.c ── pic_init outb 0x20/0x21/0xA0/0xA1, EOI                  │
│  intr-stubs.S ── 256 IDT trampolines → intr_handler()                   │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │ function calls
┌──────────────────────────────────────▼──────────────────────────────────┐
│                Pintos kernel logic   (src/threads, src/lib)             │
│  init.c:main()  thread.c (scheduler)  synch.c (locks)  palloc/malloc    │
│  printf → console.c → serial_putc / vga_putc                            │
└─────────────────────────────────────────────────────────────────────────┘
```

The two arrows that cross the seam:
- **Downward (kernel → Qemu):** `inb`/`outb` instructions trap into Qemu's device models.
- **Upward (Qemu → kernel):** the 8259 model injects an interrupt vector, the CPU dispatches via the IDT, and a registered `intr_handler_func` runs.

Everything else in Pintos (scheduling, locks, paging, the file system) is just C running on the emulated CPU — invisible to Qemu.