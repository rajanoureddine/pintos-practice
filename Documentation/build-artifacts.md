# Pintos Build Artifacts — What's in `build/` and Where It Comes From

When you run `make` inside `src/threads`, a `build/` directory fills up with files. This document explains what each one is, what tool produced it, and — for readers newer to systems programming — *why* the build is structured this way at all. The relevant Makefile is `src/Makefile.build`.

---

## 1. The 30-second mental model

Compiling C is really two jobs:

1. **Compilation** — turn one source file into one *object file*. The object file contains machine code, but with **holes** wherever the source mentioned something defined in another file (`extern` variables, functions in other `.c` files, etc.). These holes are called **relocations**.
2. **Linking** — paste many object files together, decide where each function/variable lives in memory, and **fill in the holes** with the right addresses.

Most projects you've used (`./a.out`) do this implicitly. The kernel build is unusual because it cares deeply about *exact memory layout* — where things live in physical and virtual RAM — so it does each step explicitly and feeds the linker a custom *linker script*.

---

## 2. The build directory at a glance

After running `make` in `src/threads`:

```
threads/build/
├── Makefile                ← copy of src/Makefile.build, so you can run make from here
├── devices/                ← per-source object files
│   ├── timer.o
│   ├── serial.o
│   └── ...
├── lib/
│   └── ...
├── threads/
│   ├── start.o
│   ├── init.o
│   ├── loader.o            ← intermediate object file for the boot sector
│   ├── kernel.lds.s        ← preprocessed linker script
│   └── ...
├── tests/
│   └── ...
├── kernel.o                ← FULLY LINKED kernel ELF (with debug info)
├── kernel.bin              ← stripped version of kernel.o — what runs at boot
└── loader.bin              ← raw 512-byte MBR — what the BIOS loads first
```

The two **deliverables** are `kernel.bin` and `loader.bin` (line 5 of `Makefile.build`: `all: kernel.bin loader.bin`). Everything else is intermediate.

---

## 3. Quick file-by-file summary

| File | Size (approx) | Format | Purpose |
|---|---|---|---|
| `**/*.o` (per source) | small | ELF object | One unit of compiled code, with relocation holes. |
| `threads/loader.o` | ~3.7 KB | ELF object | The boot sector's assembly, compiled but not yet final. |
| `threads/kernel.lds.s` | small | text | The linker script after running through the C preprocessor. |
| `loader.bin` | **exactly 512 B** | raw bytes | The Master Boot Record — *no ELF wrapper*. |
| `kernel.o` | ~370 KB | ELF, **with debug info** | Fully linked kernel. The file you point GDB at. |
| `kernel.bin` | ~100 KB | ELF, **stripped** | Same kernel without debug info. Written to disk for boot. |

> Don't be fooled by the `.o` extension on `kernel.o`. It's a *Pintos convention*, not a hint about the file's structure. `kernel.o` is a fully linked, executable ELF file. `file kernel.o` confirms it: `ELF 32-bit LSB executable`. Most projects would call this `kernel.elf` or simply `kernel`.

---

## 4. The full dependency graph

```
   src/threads/start.S ┐
   src/threads/init.c  │
   src/devices/*.c     ├──── gcc -c ────► per-file .o files
   src/lib/*.c         │                  (each has unresolved
   src/lib/kernel/*.c  │                   external references)
        ...            ┘
                                                  │
                                                  ▼
                  ld -T threads/kernel.lds.s   (the linker)
                                                  │
                                                  ▼
                                            kernel.o
                                       (linked ELF, debug-info-bearing,
                                        symbol table intact — for you/GDB)
                                                  │
                                                  ▼
                  objcopy -R .note -R .comment -S
                                                  │
                                                  ▼
                                            kernel.bin
                                       (linked ELF, debug info stripped —
                                        for the kernel partition on disk)


   src/threads/loader.S ── gcc -c ──► threads/loader.o
                                                  │
                                                  ▼
                  ld -N -e 0 -Ttext 0x7c00 --oformat binary
                                                  │
                                                  ▼
                                            loader.bin
                                       (raw bytes, no ELF wrapper, exactly
                                        512 bytes, MBR signature 0xAA55 at
                                        offset 510 — for the BIOS)
```

Now in detail.

---

## 5. Step-by-step: what happens when you type `make`

### Step 1: compile each `.c` and `.S` source to a `.o` (object file)

`Makefile.build` has long lists of source files at lines 15–73, grouped by directory:

```makefile
threads_SRC  = threads/start.S
threads_SRC += threads/init.c
...
devices_SRC  = devices/pit.c
devices_SRC += devices/timer.c
...
```

The makefile then generates the corresponding object-file paths (line 76: `OBJECTS = $(patsubst %.c,%.o,$(patsubst %.S,%.o,$(SOURCES)))`) and uses implicit pattern rules to compile each one with `gcc -c`. A simplified version of what runs:

```sh
gcc -c devices/timer.c    -o devices/timer.o
gcc -c threads/init.c     -o threads/init.o
gcc -c threads/start.S    -o threads/start.o
... (one per source)
```

**What `-c` means.** "Compile, but stop short of linking." The output is an **object file** (`.o`), which contains:

- **Machine code** for the functions defined in this source file.
- A **symbol table** listing what this file *defines* (e.g. `timer_init`) and what it *needs from elsewhere* (e.g. `printf`, `intr_register_ext`).
- **Relocations** — placeholders, one per "needs from elsewhere" reference, with instructions like *"the 4 bytes at offset 0x14 should be patched with the address of `printf`."*

You can inspect any of these:

```sh
nm devices/timer.o          # symbol table
objdump -d devices/timer.o  # disassembly (with placeholder zeros)
objdump -r devices/timer.o  # relocation entries
```

This is exactly the mechanism we walked through in `start-annotated.md` for `call main` — `start.o` has a relocation entry saying "patch in `main`'s address here," and the linker fills it in later.

### Step 2: preprocess the linker script

```makefile
threads/kernel.lds.s: CPPFLAGS += -P
threads/kernel.lds.s: threads/kernel.lds.S threads/loader.h
```

`src/threads/kernel.lds.S` (uppercase `S`) is the linker script. Linker scripts are written in their own little language (you saw a piece of it earlier: `_start = LOADER_PHYS_BASE + LOADER_KERN_BASE;`). Pintos wants to use C `#define` constants like `LOADER_PHYS_BASE` from `loader.h` inside the script, so the script is run through the C preprocessor first to expand those constants.

The `-P` flag suppresses `# linenumber filename` markers that `cpp` would normally emit, since `ld`'s parser can't handle them.

The output, `threads/kernel.lds.s` (lowercase), is a "real" linker script that `ld` can read.

### Step 3: link everything into `kernel.o` (the linked ELF)

`Makefile.build:82-83`:
```makefile
kernel.o: threads/kernel.lds.s $(OBJECTS)
        $(LD) -T $< -o $@ $(OBJECTS)
```

This expands to roughly:

```sh
ld -T threads/kernel.lds.s -o kernel.o \
    threads/start.o threads/init.o threads/thread.o \
    devices/timer.o devices/serial.o ... \
    lib/stdio.o lib/string.o ...
```

What the linker does:

1. **Reads every `.o`'s symbol table.** Builds a single global map of symbol → defining file.
2. **Lays out memory** according to the linker script's `SECTIONS` block. For Pintos (`kernel.lds.S:6-22`):
   - Start at virtual address `0xC0020000` (i.e. `LOADER_PHYS_BASE + LOADER_KERN_BASE`).
   - First place the `.start` section (containing `start.S`) — this is why the kernel's entry point sits at the very beginning of the image.
   - Then `.text` (code from every other source).
   - Then `.rodata` (string literals, const tables).
   - Then `.data` (writable globals with initial values).
   - Then `.bss` (writable globals initialised to zero, recorded as size only — no actual zero bytes in the file).
3. **Resolves every relocation.** Walks each `.o`'s relocation list; for each one, finds the target symbol, computes the right address (or PC-relative offset), and patches the placeholder bytes.
4. **Emits one ELF executable** with full debug info (DWARF), full symbol tables, the works.

> **What is ELF?**
> ELF ("Executable and Linkable Format") is a container format for compiled code on Linux/Unix systems. An ELF file has:
> - A **header** at the top describing the file's layout, machine type, and **entry point** (`e_entry` — the virtual address of the first instruction to run).
> - A list of **sections** (`.text`, `.data`, `.bss`, `.rodata`, debug sections, the symbol table, etc.).
> - A list of **program headers** that tell a loader which chunks of the file to copy into memory and where.
> When you run `./a.out` on Linux, the kernel's ELF loader reads these headers, maps the file into memory, and jumps to `e_entry`. Pintos's `loader.S` does the *same job by hand*, in 512 bytes — see `loader-annotated.md` lines 153–168 for how it pulls `e_entry` straight out of the ELF header at byte offset `0x18`.

The result, `kernel.o`, is the kernel **as you would want to debug it**. Because it has debug info and an intact symbol table, GDB can show you function names, source lines, and variable types when stepping through it. Pintos's `pintos --gdb` workflow points GDB at this file specifically.

### Step 4: strip to `kernel.bin`

`Makefile.build:85-86`:
```makefile
kernel.bin: kernel.o
        $(OBJCOPY) -R .note -R .comment -S $< $@
```

`objcopy` is a binutils tool for "copy this object file to that one, with transformations." Here:

- `-R .note` — remove the `.note` section (build-id, ABI tags — irrelevant at boot).
- `-R .comment` — remove the `.comment` section (compiler version strings).
- `-S` — remove all symbols and **all debug info** (the DWARF sections, which can be huge).

Result: the same code, the same layout, the same ELF entry point, but ~3× smaller. This is `kernel.bin` — the version that gets written to disk and run on the (emulated) machine. Because the entry point and program headers are intact, the loader can still find `e_entry` and the BIOS-style ELF parsing in `loader.S` works identically on stripped or unstripped kernels.

> **Why have both `kernel.o` and `kernel.bin`?** Two audiences:
> - `kernel.o` (with debug info) is for **humans** — debugging in GDB, reading disassembly, mapping crashes back to source lines.
> - `kernel.bin` (stripped) is for **the machine** — smaller, faster to load, no debug info wasting space on disk or in RAM.
> Both contain identical *executable* code; they differ only in metadata.

### Step 5: build the boot sector — `loader.bin`

`Makefile.build:88-92`:
```makefile
threads/loader.o: threads/loader.S
        $(CC) -c $< -o $@ $(ASFLAGS) $(CPPFLAGS) $(DEFINES)

loader.bin: threads/loader.o
        $(LD) -N -e 0 -Ttext 0x7c00 --oformat binary -o $@ $<
```

The first rule compiles `loader.S` to `loader.o` like any other source. **The interesting rule is the second one.** Look at the flags:

| Flag | What it does | Why we need it |
|---|---|---|
| `-N` | Don't page-align sections; pack them tightly. | Every byte counts in a 512-byte sector. |
| `-e 0` | Set the entry point to offset `0`. | The BIOS unconditionally jumps to `0x7c00:0` — i.e. the first byte of our output. |
| `-Ttext 0x7c00` | Place the `.text` section at virtual address `0x7c00`. | Any absolute reference inside the loader (e.g. label addresses) needs to resolve assuming the loader runs at `0x7c00`. |
| `--oformat binary` | **Don't emit an ELF file.** Emit raw bytes. | The BIOS doesn't know what ELF is — it just copies 512 bytes from the disk and executes them. The output must be pure machine code with no header. |

The result is a flat, headerless binary of exactly 512 bytes. The "exactly 512" comes from `loader.S`'s own structure: at the end of the source (lines 251–263), it uses `.org` directives to pad to specific offsets and place `0xAA55` at byte 510:

```asm
.org LOADER_SIG - LOADER_BASE
.word 0xaa55                 ; the BIOS bootable-sector signature
```

If the loader's code grew past 510 bytes, this would either silently overlap or the build would fail. The Pintos loader is hand-tuned to leave just enough room.

> **What is an MBR?**
> The Master Boot Record is the first 512 bytes of a hard disk in the legacy PC boot scheme. It contains:
> - 446 bytes of boot code (Pintos's loader).
> - A **partition table** of four 16-byte entries (offsets 446–509).
> - The signature `0x55, 0xAA` at offsets 510–511, which the BIOS checks before agreeing the disk is bootable.
> When the BIOS finds a valid MBR, it loads those 512 bytes to physical address `0x7c00` and jumps. That's the very first piece of *your code* the CPU executes.

### Step 6: assemble a bootable disk

`Makefile.build` has a target for `os.dsk` (line 94) that just concatenates `kernel.bin`, but the *real* disk used by `pintos --qemu` is assembled by the `pintos` Perl script (`src/utils/pintos`). It does the steps a real OS installer would:

1. Allocate a blank disk image file.
2. Write `loader.bin` into sector 0 (the MBR).
3. Write a partition table entry that says "partition 1: type `0x20` (Pintos kernel), bootable, starting at sector 1, length N sectors."
4. Write `kernel.bin` into the data area starting at sector 1.
5. Hand the resulting image file to Qemu: `qemu-system-i386 -hda <image>`.

When Qemu boots, the BIOS:
- Reads sector 0 of the disk (`loader.bin`) into `0x7c00`.
- Jumps to `0x7c00`.
- Loader scans the partition table, finds the type-`0x20` partition, and reads `kernel.bin` (sector by sector via `INT 13h/AH=42h` — see `loader-annotated.md` lines 230–246) into physical `0x20000`.
- Loader pulls the ELF entry point out of the kernel image and far-jumps to `start.S` (see `start-annotated.md`).
- `start.S` switches to protected mode and `call`s `main` (see `start-annotated.md` for that handoff).

Closing the loop with the rest of the documentation in this folder.

---

## 6. Useful commands for poking at the artifacts

You can learn a lot by inspecting these files yourself:

```sh
cd src/threads/build

# What kind of file is each?
file kernel.o kernel.bin loader.bin

# List every symbol in the kernel and its address.
nm kernel.o | sort

# Find a specific symbol (here: main).
nm kernel.o | grep ' main$'

# Disassemble around a specific function.
objdump -d kernel.o | grep -A 30 '<main>:'

# See the section layout (sizes, virtual addresses).
objdump -h kernel.o

# See the program headers (what gets loaded at boot).
readelf -l kernel.o

# Look at relocations in an unlinked .o (interesting for understanding the linker).
objdump -r threads/start.o

# Verify loader.bin is exactly 512 bytes, ending in 0x55 0xAA.
ls -l loader.bin
xxd loader.bin | tail -1
```

---

## 7. The diff between `.o`, `.bin`, and `.elf` (jargon decoder)

You'll see these extensions a lot in low-level projects. They mean different things by convention but the file *format* is determined by the build flags, not the extension.

| Extension | Conventional meaning | What it usually contains |
|---|---|---|
| `*.o` | "Object file" | An ELF object — compiled but **not linked**. Has unresolved relocations. |
| `*.elf` | "Linked ELF" | An ELF executable — linked, with headers and symbol info. |
| `*.bin` | "Binary" | Raw bytes, no header — typically used for boot sectors, ROM images, firmware. |

Pintos breaks one of these conventions: `kernel.o` is not an unlinked object, despite the name. It's a fully linked ELF — the same kind of file most projects would call `kernel.elf`. Once you internalise that it's the *exact same kind of file as a Linux executable, just stripped of OS-loading metadata*, the rest of the build pipeline will feel much less mysterious.

---

## 8. One-line summary

`make` runs `gcc -c` to chop sources into object files (`.o`), runs `ld` to fuse them into a linked kernel (`kernel.o`) at addresses dictated by `kernel.lds.S`, runs `objcopy` to strip debug info into the on-disk image (`kernel.bin`), and separately turns `loader.S` into a raw 512-byte MBR (`loader.bin`). The `pintos` script splices `loader.bin` and `kernel.bin` into a disk image; Qemu's BIOS plays them back in order at boot.
