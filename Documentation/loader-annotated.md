# `threads/loader.S` — Annotated Walkthrough

This document annotates the Pintos boot sector (`src/threads/loader.S`). The loader is a **512-byte** real-mode program planted in the MBR. Qemu's BIOS loads it at physical address `0x7c00` and jumps to it. Its single job is to (a) find the Pintos kernel partition on a disk, (b) read it into RAM, and (c) jump to the kernel's ELF entry point.

This file is **16-bit real mode** (`.code16`). Memory addresses are formed as `segment:offset`, where the linear address = `segment * 16 + offset`. So segment `0x2000` corresponds to linear `0x20000` (128 KB).

---

## Register conventions used in this file

| Register | Role in loader.S |
|---|---|
| `%dl` | Disk number being scanned (`0x80` = first hard disk, `0x81` = second…). Set by BIOS on entry. |
| `%dh` | Unused here; BIOS would put head number in it for legacy `int 13h`. |
| `%si` | Pointer to current partition table entry (relative to `%es`). Each entry is 16 bytes. |
| `%ax` | Scratch / segment value being loaded into `%es`. Also temporarily holds `int 13h` results. |
| `%al` | The ASCII digit (`'1'`..`'4'`) of the partition currently being printed. |
| `%bx` / `%ebx` | Sector index (LBA). `%bl`'s low 4 bits double as a "print a dot every 16 sectors" counter. |
| `%cx` / `%ecx` | Loop counter (number of sectors remaining to read). Used by `loop`. |
| `%es` | Segment of the buffer that disk reads land in. Continually advanced as the kernel grows in memory. |
| `%esp` | Stack pointer at `0xf000` (60 KB). The kernel later inherits this stack as its initial thread's stack. |

---

## Lines 80–168 — finding and loading the kernel

### Lines 72–86 — `check_partition`: is this entry the Pintos kernel?

```asm
72  check_partition:
73      # Is it an unused partition?
74      cmpl $0, %es:(%si)         ; first 4 bytes of entry = 0 → unused
75      je next_partition
76
77      # Print [1-4].
78      call putc                   ; print '1'..'4'
79
80      # Is it a Pintos kernel partition?
81      cmpb $0x20, %es:4(%si)     ; partition type byte (offset 4)
82      jne next_partition
83
84      # Is it a bootable partition?
85      cmpb $0x80, %es:(%si)      ; status byte (offset 0): 0x80 = active
86      je load_kernel
```

**State at entry:** `%es:%si` points at a 16-byte partition table entry. The MBR partition entry layout is:

| Offset | Size | Meaning |
|---|---|---|
| 0 | 1 | Status (`0x80` = bootable, `0x00` = not) |
| 1–3 | 3 | CHS of first sector (legacy, ignored) |
| **4** | 1 | **Partition type** — Pintos uses `0x20` |
| 5–7 | 3 | CHS of last sector (legacy) |
| **8** | 4 | **LBA of first sector** |
| **12** | 4 | **Number of sectors** |

Two checks are required for a match: type byte = `0x20` **and** status byte = `0x80`. Anything else falls through to `next_partition`.

### Lines 88–98 — advance through partitions and drives

```asm
88  next_partition:
89      add $16, %si               ; %si += sizeof(partition entry)
90      inc %al                    ; '1' → '2' → '3' → '4'
91      cmp $510, %si              ; past 4th entry?
92      jb check_partition
93
94  next_drive:
95      inc %dl                    ; try next disk (0x80 → 0x81 → ...)
96      jnc read_mbr               ; until %dl wraps from 0xFF to 0x00
```

`%si` walks from `446` (start of partition table inside the MBR) through `446+16*3 = 494`; the `cmp $510` check stops it cleanly before the `0xAA55` boot signature at offset 510. `%dl` is incremented across all hard disks; when it overflows past `0xFF`, the carry flag fires and `jnc` falls through to `no_such_drive`.

### Lines 100–107 — give up

```asm
100 no_such_drive:
101 no_boot_partition:
102     call puts
103     .string "\rNot found\r"
104     int $0x18                  ; BIOS "boot failed" service
```

`int $0x18` traditionally invoked ROM BASIC; modern BIOSes reboot or print "no bootable device". Under Qemu it just halts the boot.

### Lines 113–151 — `load_kernel`: pull the kernel into RAM

**State on arrival:**
- `%dl` = drive containing the kernel
- `%es:%si` → the matched partition table entry
- `%es` = `0x2000` (set earlier so the buffer for the partition-scan reads sat at `0x20000`)

```asm
113 load_kernel:
114     call puts
115     .string "\rLoading"
116
117     # Figure out number of sectors to read.
118     mov %es:12(%si), %ecx      ; ECX = partition's sector count
119     cmp $1024, %ecx            ; cap at 1024 sectors = 512 KB
120     jbe 1f
121     mov $1024, %cx
122 1:
123     mov %es:8(%si), %ebx       ; EBX = LBA of partition's first sector
124     mov $0x2000, %ax           ; AX = segment 0x2000 → linear 0x20000
```

After this prologue:

| Register | Value |
|---|---|
| `%ecx` | sectors to read (≤ 1024) — the `loop` counter |
| `%ebx` | LBA of next sector to read (starts at partition's first sector) |
| `%ax` | segment for the next sector's destination buffer |
| `%dl` | source drive (untouched) |

```asm
132 next_sector:
133     mov %ax, %es               ; ES = current load segment
134     call read_sector           ; reads sector EBX into ES:0000
135     jc read_failed
136
137     # Print '.' every 16 sectors == 8 KB.
138     test $15, %bl              ; low 4 bits of BL == 0?
139     jnz 1f
140     call puts
141     .string "."
142 1:
143
144     # Advance memory pointer and disk sector.
145     add $0x20, %ax             ; AX += 0x20 paragraphs = 0x200 bytes = 1 sector
146     inc %bx                    ; next LBA
147     loop next_sector           ; decrement ECX, jump if non-zero
```

**Key arithmetic on the load address.** A real-mode segment is multiplied by 16 to form the linear address, so adding `0x20` to the segment advances the linear pointer by `0x20 * 16 = 512` bytes — exactly one sector. The kernel ends up laid out contiguously starting at linear `0x20000`:

```
   linear address    contents
   0x20000           sector 0 (ELF header)
   0x20200           sector 1
   0x20400           sector 2
   ...
```

`%bx`/`%ebx` is the LBA, advanced by 1 each iteration. The low 4 bits of `%bl` give a free progress indicator: a `.` is printed every 16 sectors (8 KB).

### Lines 153–168 — jump to the kernel

```asm
163     mov $0x2000, %ax
164     mov %ax, %es                ; ES = 0x2000
165     mov %es:0x18, %dx           ; DX = low 16 bits of ELF e_entry
166     mov %dx, start              ; store at "start" label (reusing code bytes!)
167     movw $0x2000, start + 2     ; segment for far jump
168     ljmp *start                 ; far indirect jump to segment:offset
```

This is the most subtle piece in the file.

**Why `%es:0x18`?** The very first sector loaded into `0x20000` is the kernel's ELF header. In the [ELF32 header layout](https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.eheader.html), `e_entry` (the program's entry point) sits at file offset `0x18` (a 4-byte field). So `%es:0x18` reads it directly out of the loaded image. We only take the low 16 bits because the loader is still in real mode where offsets are 16-bit; the high bits of the entry are always 0 anyway because the entry sits inside the first 64 KB of the kernel segment.

**Why store the address in memory at the `start` label?** The x86 has no instruction that does a far jump to a `segment:offset` value held in registers; the only far-indirect form is `ljmp *(mem)`, which reads the target out of memory. To save 4 bytes (every byte counts in a 512-byte sector), the loader **overwrites four bytes of its own code** at the `start` label (lines 171–174) — that code is part of the `read_failed` path and is never executed once we reach line 163. Self-modifying code as a space optimization.

After the `ljmp`:

| Register | Value |
|---|---|
| `%cs` | `0x2000` |
| `%eip` | `e_entry` (low 16 bits) |
| `%ds`, `%es`, `%ss` | as set earlier |
| `%esp` | `0xf000` (loader's stack — kernel inherits it) |
| `%dl` | original boot drive |

Control passes to `start.S`, the next file in the boot chain.

---

## Lines 230–246 — `read_sector`: pulling one sector off disk

```asm
230 read_sector:
231     pusha                       ; save all 8 GP regs (16 bytes)
232     sub %ax, %ax                ; AX = 0
233     push %ax                    ; LBA [48:63] = 0
234     push %ax                    ; LBA [32:47] = 0
235     push %ebx                   ; LBA [0:31]  = sector to read
236     push %es                    ; buffer segment
237     push %ax                    ; buffer offset = 0
238     push $1                     ; sector count = 1
239     push $16                    ; packet size = 16 bytes
240     mov $0x42, %ah              ; INT 13h, function 42h: extended read
241     mov %sp, %si                ; DS:SI → packet on stack
242     int $0x13                   ; BIOS disk service
243     popa                        ; pop 16-byte packet, preserves flags
244 popa_ret:
245     popa                        ; restore caller's registers
246     ret                         ; CF still indicates success/error
```

This is the cleanest example of using BIOS services as a hardware abstraction. We're invoking **INT 13h, AH=42h "Extended Read Sectors From Drive"**, which takes its arguments via a structure called the **Disk Address Packet (DAP)** rather than packing them into registers like the legacy CHS interface did.

### The DAP, built directly on the stack

The DAP is 16 bytes, and the loader builds it in reverse by pushing onto a downward-growing stack. When the dust settles, `%ds:%si` (= `%ss:%sp`) points to:

```
   offset  size  field             pushed at line  value
   ──────  ────  ────────────────  ──────────────  ─────────────
     0      1    packet size               239     16
     1      1    reserved                  239     0   (high byte of pushed $16)
     2      2    sector count              238     1
     4      2    buffer offset             237     0
     6      2    buffer segment            236     %es
     8      4    LBA [0:31]                235     %ebx
    12      4    LBA [32:63]               234,233 0
```

Each 16-bit `push` lays down 2 bytes; each 32-bit `push` lays down 4. Adding them up: `2 + 2 + 2 + 2 + 2 + 4 + 4 = 18`… wait, the math isn't quite that — one of those `push`es is a 32-bit `push %ebx`. Let's redo it: lines 233, 234, 236, 237, 238, 239 each push 2 bytes (= 12 bytes), and line 235 pushes 4 bytes (`%ebx`), totalling **16 bytes** — exactly one DAP.

### Why the trailing `popa`s

Two consecutive `popa`s appear because two different things were pushed onto the stack:

1. The **caller's general-purpose registers** at line 231 (`pusha`).
2. The **16-byte DAP** built from lines 233–239.

After `int $0x13` returns, BIOS sets the **carry flag (CF)** to indicate error or success. We need to:

- Discard the DAP — but **without disturbing the flags**.
- Restore the caller's registers — also without disturbing the flags.
- Return to the caller with CF still meaningful.

The first `popa` (line 243) is the trick: `popa` pops 8 registers × 2 bytes = **exactly 16 bytes** (the DAP's size) and **does not modify the flags register**. So it's used as a "free" 16-byte stack adjustment that preserves CF. The values popped into the GP registers are immediately overwritten by the second `popa` (line 245), which restores the caller's state.

The function returns with:
- All caller GP registers restored to their pre-call values.
- Carry flag = 0 on success, 1 on disk error.
- The buffer at `%es:0000` filled with one sector (512 bytes) of disk data.

---

## Quick reference: the boot-time data flow

```
   BIOS loads MBR (this file) at 0x7c00
            │
            ▼
   loader.S scans hd[a-z] partition tables
            │
            ▼   for each entry: check status==0x80 && type==0x20
   load_kernel:  read up to 1024 sectors into 0x20000+
            │
            ▼   read_sector (INT 13h, AH=42h)  ←──── one sector at a time
   ELF image at 0x20000, e_entry read from byte 0x18
            │
            ▼
   ljmp *start  →  start.S in protected-mode preparation  →  init.c:main()
```
