# `threads/start.S` — Annotated Walkthrough

This document annotates `src/threads/start.S`, the kernel-side bootstrap that runs immediately after `loader.S` hands off control. Its job is to take a CPU still running in **16-bit real mode** and prepare everything required to call `main()` in C: it queries memory size, enables A20, builds page tables, loads a GDT, switches to **32-bit protected mode with paging enabled**, and finally runs `call main`.

This file is linked into the kernel ELF at virtual address `LOADER_PHYS_BASE + 0x20000` (= `0xC0020000`), but **it executes from physical `0x20000`** until paging is on. That mismatch is why several instructions need explicit `LOADER_PHYS_BASE - 0x20000` arithmetic — the assembler's symbol values are all in the higher-half virtual world, which doesn't yet exist.

---

## State on entry (from `loader.S`'s `ljmp *start`)

| Register | Value | Meaning |
|---|---|---|
| `%cs` | `0x2000` | code segment of the loaded kernel |
| `%eip` | low 16 bits of ELF `e_entry` | first instruction of `start` |
| `%ss` | `0x0000` | stack segment from loader |
| `%esp` | `0xf000` | stack at linear `0x0000:0xf000` = 60 KB |
| `%dl` | original BIOS boot drive | unused here |
| CPU mode | 16-bit real mode | no paging, no segments-as-selectors |

---

## Lines 21–33 — Entry point and segment setup

```asm
21  .func start
22  .globl start
23  start:
24
25  # CS = 0x2000, SS = 0x0000, ESP = 0xf000 — initialize the rest.
28      mov $0x2000, %ax
29      mov %ax, %ds            ; DS = 0x2000  (data segment matches code)
30      mov %ax, %es            ; ES = 0x2000
31
32  # Set string instructions to go upward.
33      cld                      ; clear direction flag → REP advances upward
```

After this:
- `%ds = %es = %cs = 0x2000`, so `mov %al, foo` and `mov foo, %al` reach memory at linear `0x20000 + offset(foo)`.
- `cld` matters for `rep stosl` later when we zero out the page directory.

---

## Lines 35–48 — Determining physical memory size

```asm
41      movb $0x88, %ah
42      int $0x15               ; BIOS: returns AX = (KB of RAM) - 1024
43      addl $1024, %eax        ; EAX = total KB of RAM
44      cmp $0x10000, %eax      ; cap at 65536 KB = 64 MB
45      jbe 1f
46      mov $0x10000, %eax
47  1:  shrl $2, %eax           ; KB → 4-KB pages (divide by 4)
48      addr32 movl %eax, init_ram_pages - LOADER_PHYS_BASE - 0x20000
```

**`int 15h / AH=88h`** is the legacy "Get Extended Memory Size" BIOS service. It returns the count of contiguous 1-KB blocks **above the 1 MB boundary** in `%ax`. So adding `1024` (the first MB) gives the total RAM in KB. The BIOS service tops out at 64 MB; anything beyond that requires the more modern `int 15h / E820h`, which Pintos doesn't use because it caps memory at 64 MB anyway (the page-table-building code below only prepares mappings for that much).

**`shrl $2, %eax`** divides KB by 4 to get the number of 4-KB pages.

**The `addr32` prefix and the weird symbol math.** We're a 16-bit code segment but want to write to the symbol `init_ram_pages`, which the linker places at virtual address `0xC0020000 + offset_of_init_ram_pages`. The actual physical address right now is `0x20000 + offset_of_init_ram_pages`. So the source code says: `init_ram_pages - LOADER_PHYS_BASE - 0x20000` — strip off the higher-half base **and** strip off the `0x20000` ELF base, leaving just the offset within the segment, which combines with `%ds = 0x2000` to produce the correct linear address. The `addr32` prefix forces a 32-bit address operand because the symbol arithmetic might exceed 16 bits.

**State after this block:** `%eax` = pages of RAM; the global `init_ram_pages` is now populated (the C kernel will read it from `init.c`).

---

## Lines 50–79 — Enabling the A20 line

```asm
55  1:  inb $0x64, %al          ; read keyboard controller status
56      testb $0x2, %al         ; bit 1 = "input buffer full"
57      jnz 1b                  ; spin while busy
58
61      movb $0xd1, %al
62      outb %al, $0x64         ; command D1: "write output port"
63
66  1:  inb $0x64, %al          ; spin until controller ready again
67      testb $0x2, %al
68      jnz 1b
69
72      movb $0xdf, %al
73      outb %al, $0x60         ; data 0xDF: A20 on, system reset off
74
77  1:  inb $0x64, %al          ; final spin
78      testb $0x2, %al
79      jnz 1b
```

**Why does A20 even exist?** The 8086 had only 20 address lines, and addresses wrapped at 1 MB. The 286 added a 21st line (A20), but for backward compatibility with software that depended on the wrap, IBM tied A20 to a pin on the **keyboard controller (8042)** that the BIOS leaves *disabled* at boot. Until A20 is re-enabled, every memory access has its 21st bit forced to 0 — addresses `0x100000`–`0x1FFFFF` alias `0x000000`–`0x0FFFFF`, which would silently corrupt any kernel that lives above 1 MB.

**The 8042 protocol used here:**

| Port | Direction | Meaning |
|---|---|---|
| `0x64` | read | status byte; bit 1 set = controller is busy |
| `0x64` | write | command byte to the controller |
| `0x60` | write | data byte (parameter to last command) |

Sequence:
1. Wait for "not busy".
2. Send command `0xD1` ("write next byte to output port") to `0x64`.
3. Wait for "not busy" again.
4. Send data `0xDF` to `0x60`. Bit 1 of the output port is the A20 gate; bit 0 is system reset (kept high — anything else would reboot the machine).
5. Wait for "not busy" one more time.

After this, the full 32-bit address space is reachable. (Modern systems often have BIOS/firmware do this, but doing it manually is universal and safe.)

---

## Lines 81–124 — Building temporary page tables

This is the densest block. We construct a **page directory** at physical `0xF000` (60 KB — note: this overlaps the loader's stack region but the stack only uses high addresses there) and **page tables** starting at physical `0x10000` (64 KB) that map the first 64 MB of physical RAM into two virtual ranges:

1. **Identity map** — virtual `0x00000000`–`0x04000000` → physical `0x00000000`–`0x04000000`. Required because the instruction *immediately after* enabling paging is still being fetched from low physical addresses; if we only mapped the higher half, paging would page-fault on the very next instruction.
2. **Higher-half map** — virtual `0xC0000000`–`0xC4000000` → physical `0x00000000`–`0x04000000`. This is the kernel's "real" home; once we're in 32-bit protected mode and have done a far jump, we can use higher-half addresses normally.

### Lines 84–90 — zero the page directory

```asm
85      mov $0xf00, %ax         ; segment 0xF00 → linear 0xF000
86      mov %ax, %es            ; ES = 0xF00
87      subl %eax, %eax         ; EAX = 0
88      subl %edi, %edi         ; EDI = 0
89      movl $0x400, %ecx       ; 0x400 = 1024 dwords = 4096 bytes (one page)
90      rep stosl               ; ES:[EDI++] = EAX, repeat ECX times
```

After this: physical `0xF000`–`0xFFFF` is zero — a freshly cleared 4-KB page directory.

### Lines 97–104 — write PDEs that point to page tables

```asm
97      movl $0x10007, %eax     ; first page-table physical addr | flags
98      movl $0x11, %ecx        ; loop 17 times → covers 17 × 4 MB = 68 MB
99      subl %edi, %edi
100 1:  movl %eax, %es:(%di)                              ; identity slot
101     movl %eax, %es:LOADER_PHYS_BASE >> 20(%di)        ; higher-half slot
102     addw $4, %di            ; next PDE (4 bytes per entry)
103     addl $0x1000, %eax      ; next page-table physical addr (+ 4 KB)
104     loop 1b
```

A 32-bit page directory entry encodes a 4-KB-aligned page-table address in its top 20 bits and flags in the low 12. `0x10007` = `0x10000 | 0x007`:
- Top 20 bits: physical `0x10000` (64 KB) — where the first page table lives.
- Low 12 bits: `0x007` = present | writable | user-accessible.

**Why two writes per iteration?** The same PDE value is stored at two locations:
- `%es:(%di)` — the identity-map slot (PDE indices 0..16 cover virtual 0..68 MB).
- `%es:LOADER_PHYS_BASE >> 20(%di)` — `LOADER_PHYS_BASE = 0xC0000000`. A page directory is indexed in 4-byte entries by the top 10 bits of the virtual address; shifting `0xC0000000` right by 20 gives the **byte offset** into the directory of the first higher-half PDE (specifically, entry 768 × 4 = `0xC00`). So we install the same mapping at PDE 0 and PDE 768 simultaneously.

### Lines 111–119 — fill the page tables

```asm
111     movw $0x1000, %ax       ; segment 0x1000 → linear 0x10000
112     movw %ax, %es           ; ES now points at the page tables
113     movl $0x7, %eax         ; physical 0 | flags 0x7
114     movl $0x4000, %ecx      ; 16384 = 16 page tables × 1024 entries
115     subl %edi, %edi
116 1:  movl %eax, %es:(%di)    ; map physical EAX
117     addw $4, %di
118     addl $0x1000, %eax      ; next 4 KB of physical
119     loop 1b
```

Each PTE maps one 4-KB page. We write `0x4000` (= `16384`) PTEs, one for every 4-KB page in the first 64 MB of physical memory. `0x4000 × 4 KB = 64 MB`. Each PTE has flags `0x7` (present + writable + user) and points to the linearly-next physical page.

### Lines 121–124 — point CR3 at the page directory

```asm
123     movl $0xf000, %eax
124     movl %eax, %cr3         ; CR3 = page directory base register (PDBR)
```

`%cr3` is the **page directory base register**: the physical address of the page directory the CPU consults whenever paging is on. We're not yet paging, so this load is a no-op until CR0.PG flips below.

---

## Lines 126–151 — switch to protected mode

### Disable interrupts

```asm
131     cli
```

We haven't built an IDT yet. Any interrupt now would dispatch through whatever garbage `IDTR` points at and crash. The C kernel (`intr_init` in `threads/interrupt.c`) builds the real IDT later; interrupts stay off until `thread_start()` runs `sti`.

### Load GDT

```asm
139     data32 addr32 lgdt gdtdesc - LOADER_PHYS_BASE - 0x20000
```

`lgdt` reads a 6-byte descriptor: 2 bytes of "limit" (= sizeof(GDT) − 1) followed by 4 bytes of base address. The descriptor lives at `gdtdesc` (lines 195–197). Same physical/virtual subtraction trick as before to compute the correct runtime address.

The two prefixes:
- `data32` — force a 32-bit operand size so all 4 bytes of the GDT base are loaded (default in 16-bit code is to load only 24 bits).
- `addr32` — force a 32-bit address. Required because the linker's symbol math may produce a value that doesn't fit in 16 bits, but the CPU itself doesn't strictly need this for the instruction to function.

The GDT itself (lines 190–193):

```asm
190 gdt:
191     .quad 0x0000000000000000    ; null segment (required at index 0)
192     .quad 0x00cf9a000000ffff    ; CS: base 0, limit 4 GB, code, ring 0
193     .quad 0x00cf92000000ffff    ; DS: base 0, limit 4 GB, data, ring 0
```

This is the canonical "**flat**" GDT: base 0, limit 4 GB, so segmentation is effectively disabled — segment registers no longer add to the linear address. From this point on, virtual addresses go straight through segmentation untouched and are translated only by paging.

### Flip the bits in CR0

```asm
149     movl %cr0, %eax
150     orl $CR0_PE | CR0_PG | CR0_WP | CR0_EM, %eax
151     movl %eax, %cr0
```

Four flags, all set in a single write:

| Bit | Name | Effect |
|---|---|---|
| `0` (`PE`) | Protection Enable | Switches CPU from real mode to protected mode. |
| `31` (`PG`) | Paging | Activates the page tables we just built. CR3 finally matters. |
| `16` (`WP`) | Write Protect | Honor write-protect bits in PTEs even in ring 0 (otherwise the kernel could blindly write read-only pages). |
| `2` (`EM`) | Emulation | Trap on FPU instructions. Pintos has no FP support — any FP op should fault, not silently execute. |

This single `mov` to `%cr0` is the real seam: the very next instruction is fetched in **protected mode with paging on**.

---

## Lines 153–176 — Reload segment registers in 32-bit mode

```asm
159     data32 ljmp $SEL_KCSEG, $1f
```

After enabling protected mode, `%cs` still contains the cached real-mode descriptor. A far jump is the only way to reload `%cs`; `ljmp` selects the new code segment (`SEL_KCSEG`, the kernel code selector pointing at GDT entry 1) and clears the prefetch queue. The `data32` prefix promotes the offset to 32 bits.

```asm
164     .code32                ; tell assembler we're 32-bit from here
165
169 1:  mov $SEL_KDSEG, %ax
170     mov %ax, %ds
171     mov %ax, %es
172     mov %ax, %fs
173     mov %ax, %gs
174     mov %ax, %ss            ; all data segs use the kernel data selector
175     addl $LOADER_PHYS_BASE, %esp   ; SS:ESP now references higher half
176     movl $0, %ebp           ; null frame pointer terminates backtraces
```

State after this block:

| Register | Value |
|---|---|
| `%cs` | `SEL_KCSEG` (flat code segment, ring 0) |
| `%ds, %es, %fs, %gs, %ss` | `SEL_KDSEG` (flat data segment) |
| `%esp` | `0xC000F000` — old `0xF000` plus higher-half base |
| `%ebp` | `0` |
| paging | on, identity + higher-half mappings live |

Stack writes still land in physical `0xF000` because both virtual `0x0000F000` and `0xC000F000` map there. But the kernel can now use higher-half addresses everywhere — which matches what the linker baked into every symbol.

---

## Lines 178–184 — call main

```asm
180     call main
181
183 1:  jmp 1b                  ; if main ever returns, hang here
```

This is the handoff to C. `main` is `int main(void)` at `threads/init.c:77`. The protected-mode environment is fully set up: a flat GDT, paging, a clean stack, no live interrupts, and IDT setup waiting for `intr_init()` later in `main` itself.

### How does `call main` know where `main` is?

It doesn't — the **linker** does, after compilation. Three layers cooperate:

**1. The assembler emits a placeholder.** When GAS assembles `call main` in `start.S`, the symbol `main` is undefined in this file (it lives in `init.c`). The assembler can't compute an address, so it does two things:

- Emits the `call` opcode with a **zero displacement** as a placeholder.
- Writes a **relocation entry** into `start.o` that says: *"at this byte offset, please patch in a 32-bit PC-relative reference to the symbol `main`."*

(`objdump -r start.o` would show an `R_386_PC32 main` entry at that offset.)

**2. The linker resolves it.** The link step (`ld -T kernel.lds.S start.o init.o thread.o …`) walks every `.o` file's symbol table and lays sections out at addresses dictated by the linker script `threads/kernel.lds.S:9-15`:

```
_start = LOADER_PHYS_BASE + LOADER_KERN_BASE;   /* = 0xC0020000 */
.text : { *(.start) *(.text) } ...
```

The `.start` section (containing `start` from `start.S`) is placed first inside `.text`, then all the other `.text` from every object file (including `init.c`'s `main`) is concatenated after. By the end of the link, the linker knows the absolute virtual address of every symbol. In the actual built kernel:

```
c00200c0 T start          ← from start.S
c0020225 T main           ← from init.c
```

**3. The displacement is patched in.** `call rel32` is encoded as `E8` followed by a 4-byte signed displacement, applied **relative to the next instruction**. The linker computes:

```
displacement = address_of(main) - address_of(next_instruction)
             = 0xC0020225      - 0xC00201A5
             = 0x00000080
```

…and writes those bytes into the placeholder. Disassembling the built kernel confirms it:

```
c00201a0:  e8 80 00 00 00       call c0020225 <main>
c00201a5:  eb fe                jmp  c00201a5
```

`E8 80 00 00 00` literally means *"call the address that is 0x80 bytes past me."*

**4. At runtime.** The CPU executing this `call` doesn't look up any symbol. It takes the current `%eip`, adds `0x80`, pushes the return address on the stack, and jumps. The "knowledge" of where `main` lives was baked into the instruction stream at link time, then loaded into RAM along with the rest of the kernel image at physical `0x20000`.

| Layer | Knows about `main`? | What it does |
|---|---|---|
| Assembler (`as`) | No — sees an external symbol | Emits opcode + zero displacement + relocation record |
| Linker (`ld`) | **Yes** — has all object files | Computes addresses, patches the displacement |
| CPU at runtime | Doesn't need to | Adds the displacement to `%eip` and jumps |

The same mechanism handles every cross-file function call in the kernel; `main` is unusual only in being the *first* one.

---

## Lines 187–203 — Static data

```asm
189     .align 8
190 gdt:                                       ; the 24-byte GDT
191     .quad 0x0000000000000000
192     .quad 0x00cf9a000000ffff
193     .quad 0x00cf92000000ffff
194
195 gdtdesc:
196     .word gdtdesc - gdt - 1                ; limit  = 23
197     .long gdt                              ; base   = address of gdt
198
201 .globl init_ram_pages
202 init_ram_pages:
203     .long 0                                ; populated at line 48
```

`init_ram_pages` is `.globl` so the C code (`init.c:94`) can read it: `printf("Pintos booting with %u kB RAM...\n", init_ram_pages * PGSIZE / 1024)`.

---

## Summary diagram

```
   loader.S  ──ljmp 0x2000:e_entry──►  start: (16-bit, %cs=0x2000)
                                                │
                                                ▼
   [INT 15h AH=88h] → memory size → init_ram_pages
                                                │
                                                ▼
   [8042 keyboard ctrl] D1/DF dance → A20 line on
                                                │
                                                ▼
   build PDE+PTE at 0xF000 / 0x10000 → identity + higher-half map
                                                │
                                                ▼
   load CR3, lgdt, set CR0.PE|PG|WP|EM         ← THE SWITCH
                                                │
                                                ▼
   far jmp $SEL_KCSEG, $1f                      ← reload CS, now 32-bit
                                                │
                                                ▼
   reload DS/ES/FS/GS/SS, %esp += 0xC0000000
                                                │
                                                ▼
   call main                                    threads/init.c:77
```

Three big transitions happen in this one file: **physical → virtual addressing**, **real mode → protected mode**, and **assembly → C**. Each is reversible only by destroying the system, which is why the order — A20 first, page tables before CR0.PG, GDT before CR0.PE, far-jump before using new segments — is mandatory.