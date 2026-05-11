# Thread Creation & Context Switching in Pintos — Deep Dive

This document walks through the mechanics of `thread_create()` and the
context-switching machinery in `switch.S`. The goal is to understand, at
the level of individual stack words and `ret` instructions, how Pintos
bootstraps a brand-new thread into running its own function — even though
that thread has never actually executed before.

The relevant source files are:

- `src/threads/thread.c` — `thread_create()`, `kernel_thread()`,
  `schedule()`, `alloc_frame()`.
- `src/threads/switch.S` — the actual assembly that swaps stacks.
- `src/threads/switch.h` — the C declarations of the stack-frame
  structs and the `SWITCH_CUR` / `SWITCH_NEXT` offsets.
- `src/threads/thread.h` — `struct thread` and the page-layout comment.

## 1. The Stack Layout — The "Frame Sandwich"

When `thread_create()` runs, a new page is allocated with
`palloc_get_page(PAL_ZERO)`. The `struct thread` sits at the **bottom**
of the page (offset 0); the kernel stack grows **downward** from the top
of the page (offset 4 kB). See the big comment at `thread.h:33-55`.

`init_thread()` sets `t->stack = (uint8_t *) t + PGSIZE;` — i.e., the
stack pointer starts at the very top of the page (`thread.c:463`). This
is the only "real" stack the thread will ever use until/unless it traps
into a user program.

Then `alloc_frame()` (`thread.c:475-483`) is called three times. Each
call does:

```c
t->stack -= size;
return t->stack;
```

So each frame is carved out of the stack, downward, one after the other.
The three calls in order are:

1. `kf = alloc_frame(t, sizeof *kf);`  → `kernel_thread_frame`
   (12 bytes: `eip`, `function`, `aux`)
2. `ef = alloc_frame(t, sizeof *ef);`  → `switch_entry_frame`
   (4 bytes: `eip`)
3. `sf = alloc_frame(t, sizeof *sf);`  → `switch_threads_frame`
   (28 bytes: `edi`, `esi`, `ebp`, `ebx`, `eip`, `cur`, `next`)

Because the stack grows down and each `alloc_frame` decrements `t->stack`
*before* returning, the last frame allocated (`switch_threads_frame`)
ends up at the **lowest address** — which is the **top of stack** in
stack-discipline terms. So at the end, `t->stack` points at the
`switch_threads_frame`. That saved pointer is what `switch.S` will load
into `%esp` the first time this thread is scheduled.

```
                     HIGH ADDRESS  (top of page, offset 4096)
       +---------------------------------+  <-- initial t->stack (PGSIZE)
       |   kernel_thread_frame           |
       |     +0  eip   = NULL            |   "fake return addr" (never used)
       |     +4  function   <-- arg1     |
       |     +8  aux        <-- arg2     |
       +---------------------------------+  <-- after 1st alloc_frame
       |   switch_entry_frame            |
       |     +0  eip = kernel_thread     |   <-- ret in switch_entry jumps HERE
       +---------------------------------+  <-- after 2nd alloc_frame
       |   switch_threads_frame          |
       |     +0   edi (garbage)          |
       |     +4   esi (garbage)          |
       |     +8   ebp = 0                |
       |     +12  ebx (garbage)          |
       |     +16  eip = switch_entry     |   <-- ret in switch_threads jumps HERE
       |     +20  cur  (unused slot)     |
       |     +24  next (unused slot)     |
       +---------------------------------+  <-- final t->stack (saved in struct thread)
       |          unused stack           |
       |               ...               |
       |               ...               |
       +---------------------------------+
       |   struct thread (tid, status,   |
       |   name, stack ptr, magic, ...)  |
       +---------------------------------+  <-- LOW ADDRESS (offset 0, page base)
```

The "sandwich" is three return frames stacked so that **each `ret` peels
off one slice and lands in the next layer**. The bottom slice (lowest
address, top of stack) drives the first jump; the topmost slice
(highest address, deepest in the stack discipline) drives the last.

## 2. Why this order? The fake return path

The CPU only knows one mechanism for returning: `ret` pops the top
dword off the stack and jumps to it. So Pintos builds a chain of fake
return addresses, each one redirecting the CPU one step closer to the
user's `function(aux)`. Crucially, the CPU **cannot distinguish** a
legitimate return address pushed by a `call` from one written there by
hand in C — both are just dwords sitting on the stack.

Trace from the moment the scheduler picks this new thread:

### Step A — scheduler calls `switch_threads(cur, next)` (`thread.c:564`)

Inside `switch_threads`, after saving callee-saved registers of the
*old* thread and loading `%esp` from the *new* thread, the new `%esp`
points at `switch_threads_frame.edi`. Then:

```
popl %edi   ; pops garbage
popl %esi   ; pops garbage
popl %ebp   ; pops 0
popl %ebx   ; pops garbage
ret         ; pops eip = switch_entry, jumps there
```

The `ret` consumes `sf->eip`, which `thread_create` set to
`switch_entry` (`thread.c:198`). The CPU jumps to `switch_entry`.

### Step B — `switch_entry` runs (`switch.S:53-65`)

At entry, `%esp` points just above where `sf->eip` used to be — i.e.,
at the `cur` and `next` argument slots. `switch.S` does:

```
addl $8, %esp        ; discard cur, next (those 8 bytes)
pushl %eax           ; %eax holds the old thread (prev) returned by switch_threads
call thread_schedule_tail   ; finalizes the switch
addl $4, %esp        ; pop the pushed prev
ret                  ; pops eip = kernel_thread, jumps there
```

After the `addl $8`, `%esp` points at `ef->eip = kernel_thread`. After
`call/addl`, it's restored to that same spot. So the final `ret`
consumes `ef->eip` and jumps to `kernel_thread`.

### Step C — `kernel_thread(function, aux)` runs (`thread.c:419-426`)

But wait — how does `kernel_thread` find its arguments? Because the
layout puts the `kernel_thread_frame` (with `function` and `aux` in
arg slots) immediately *above* `ef->eip`. After the final `ret` pops
`ef->eip`, `%esp` points at `kf->eip = NULL` — and the *next* two
dwords are `function` and `aux`, exactly where a function called with
the standard cdecl ABI would expect its arguments. So `kernel_thread`
reads `function` at `[esp+4]` and `aux` at `[esp+8]`, and calls
`function(aux)`.

The fake `kf->eip = NULL` is the would-be return address; if
`function` ever returns, `kernel_thread` doesn't fall through into
that NULL — it calls `thread_exit()` instead, so the NULL is never
dereferenced.

So the order **switch_threads_frame → switch_entry_frame →
kernel_thread_frame** (low → high address) is exactly the chain
`ret → ret → call` that the CPU will walk.

## 3. `switch.S` walkthrough

`switch_threads(cur, next)` is a C-callable function. By the time it's
entered, the caller has already pushed `next` and `cur` (cdecl pushes
right-to-left), and `call` pushed the return address. So at entry:

```
   [esp+0]  return address (back into schedule())
   [esp+4]  cur
   [esp+8]  next
```

The function preserves the SysV i386 callee-saved registers
(`%ebx, %ebp, %esi, %edi`) per ABI (`switch.S:18-29`):

```
pushl %ebx
pushl %ebp
pushl %esi
pushl %edi
```

After these four pushes, `%esp` has moved down 16 bytes, so:

```
   [esp+0]   saved edi
   [esp+4]   saved esi
   [esp+8]   saved ebp
   [esp+12]  saved ebx
   [esp+16]  return address       <-- this is the "eip" slot in switch_threads_frame
   [esp+20]  cur                  <-- SWITCH_CUR
   [esp+24]  next                 <-- SWITCH_NEXT
```

This is exactly `struct switch_threads_frame` (`switch.h:6-15`), with
the offsets `SWITCH_CUR=20` and `SWITCH_NEXT=24` matching. The
constants in `switch.h` and the field order in the struct are
**coupled** — if you reorder the pushes in `switch.S`, you must update
both the struct and the offset macros.

Now the stack swap (`switch.S:32-41`):

```
mov thread_stack_ofs, %edx       ; %edx = offsetof(struct thread, stack)
movl SWITCH_CUR(%esp), %eax      ; %eax = cur
movl %esp, (%eax,%edx,1)         ; cur->stack = %esp     (SAVE old esp into old thread)
movl SWITCH_NEXT(%esp), %ecx     ; %ecx = next
movl (%ecx,%edx,1), %esp         ; %esp = next->stack    (LOAD new esp from new thread)
```

`thread_stack_ofs` is defined at `thread.c:584` as
`offsetof(struct thread, stack)` — exported as a global so assembly
doesn't have to hard-code the C struct layout. If `struct thread`
grows or its fields are reordered, this variable updates
automatically at compile time and the assembly keeps working.

After the second `movl`, **we are now running on the new thread's
stack.** Same code, same `%eip`, but `%esp` points into a totally
different page. The new thread's stack also happens to hold a
`switch_threads_frame` at top — either because the new thread was
previously preempted *inside* `switch_threads` and is resuming
(symmetric case), or — for a brand-new thread — because
`thread_create()` faked one (asymmetric case, see §4).

Then (`switch.S:44-48`):

```
popl %edi
popl %esi
popl %ebp
popl %ebx
ret
```

These restore the *new* thread's callee-saved registers and return to
wherever its `switch_threads_frame.eip` says. For a thread resuming a
prior preemption, that's just back into `schedule()`. **For a
brand-new thread, that's `switch_entry`** — the magic that makes it
work.

One detail about the return value: `switch_threads` returns a
`struct thread *` (the old/prev thread). Per cdecl, that return value
lives in `%eax`. Notice `%eax` was set to `cur`
(`movl SWITCH_CUR(%esp), %eax`) and never overwritten. So when
control eventually returns to the *new* thread's caller, `%eax` still
holds the *old* thread pointer — that's how `prev` is passed to
`thread_schedule_tail()` (both in the normal return-into-`schedule`
path and in `switch_entry`, which `pushl %eax`s before calling).

## 4. The First-Switch Paradox

**Paradox**: `switch_threads` works by symmetry — it assumes the
thread you're switching *into* is also sitting inside `switch_threads`,
blocked at the `popl %edi … ret` sequence, waiting to be reanimated.
But a brand-new thread has **never executed any code at all**. It has
never called `switch_threads`. So how does the symmetric scheme
bootstrap from nothing?

**Resolution**: `thread_create()` **lies to the CPU**. It hand-builds
a `switch_threads_frame` on the new thread's stack that is
byte-for-byte indistinguishable from the frame a real, preempted
thread would have left — except the saved `eip` points at
`switch_entry` instead of somewhere inside `schedule()`
(`thread.c:196-199`):

```c
sf = alloc_frame (t, sizeof *sf);
sf->eip = switch_entry;
sf->ebp = 0;
```

The other fields (`edi`, `esi`, `ebx`, `cur`, `next`) are
uninitialized garbage — they're never read meaningfully because:

- `popl %edi/%esi/%ebx` pop garbage into registers that
  `switch_entry` doesn't use.
- `popl %ebp` loads the deliberately-zeroed `sf->ebp`, terminating any
  `%ebp`-based backtrace cleanly at the bottom of the new thread.
- The `cur`/`next` slots at `[esp+20]`/`[esp+24]` are discarded by
  `addl $8, %esp` in `switch_entry`.

So when the new thread first runs:

1. `switch_threads` pops 4 dwords (mostly garbage) into edi/esi/ebp/ebx.
2. `ret` pops `sf->eip` — but `sf->eip` is `switch_entry`, not a real
   return address. **The CPU doesn't know the difference between a
   real return and a fake one. `ret` just means "pop and jump."**
3. Execution lands in `switch_entry`, which then finishes the
   bootstrap by calling `thread_schedule_tail` (the cleanup work that
   a normally-returning `switch_threads` caller would do back in
   `schedule()`), then `ret`s a *second* time into `kernel_thread`.

The reason `switch_entry` exists at all (rather than `sf->eip`
pointing directly at `kernel_thread`): the normal path through
`schedule()` calls `thread_schedule_tail(prev)` after `switch_threads`
returns. A new thread can't go through `schedule()` (it's not where
it left off — there's no "where it left off"). So `switch_entry` is a
shim that does the equivalent post-switch work — call
`thread_schedule_tail(prev)` with `prev` taken from `%eax` — before
handing off to `kernel_thread`. This keeps the invariant that
`thread_schedule_tail` runs exactly once per switch, regardless of
whether the incoming thread is new or resumed.

## 5. How `function` and `aux` reach the CPU

Recap the route:

1. **In `thread_create`** (`thread.c:187-190`):
   ```c
   kf = alloc_frame (t, sizeof *kf);
   kf->eip = NULL;
   kf->function = function;
   kf->aux = aux;
   ```
   These are written into stack memory at fixed offsets matching
   `struct kernel_thread_frame` (`thread.c:41-46`): `eip` at +0,
   `function` at +4, `aux` at +8.

2. **When `switch_entry` finally executes its terminating `ret`**,
   `%esp` is sitting at `ef->eip` (which is `kernel_thread`). `ret`
   pops that into `%eip` and jumps. Now `%esp` is one dword higher —
   pointing exactly at `kf->eip` (the NULL "return address"). And
   just above that sit `function` and `aux`.

3. **From `kernel_thread`'s point of view, the stack looks identical
   to a normal C call**: `[esp+0]` = return address (NULL),
   `[esp+4]` = first argument, `[esp+8]` = second argument. The
   cdecl ABI says exactly that. So `kernel_thread(function, aux)`
   reads its parameters normally — no special machinery, just careful
   layout that mimics what `call kernel_thread(function, aux)` would
   have produced.

4. **`kernel_thread` then does** (`thread.c:419-426`):
   ```c
   intr_enable ();
   function (aux);
   thread_exit ();
   ```
   It enables interrupts (because `schedule()` runs with them off,
   and the new thread never went through the part of `schedule()`
   that would have re-enabled them on its way out), invokes the
   user's `function` with `aux`, and on return kills the thread —
   which is why `kf->eip = NULL` is harmless: control never tries to
   use it.

## TL;DR mental model

`thread_create` doesn't start a thread; it **forges a crime scene** —
a stack that looks exactly like one a real, preempted thread would
have left behind, but with the `eip` values chosen so that the chain
of `ret`s walks the CPU through a setup ladder
(`switch_threads → switch_entry → kernel_thread → function`). The CPU
has no way to distinguish a forged return path from a real one,
because a return address is just a number the CPU jumps to.

The whole machinery rests on a few small but load-bearing facts:

- The cdecl calling convention puts arguments on the stack at known
  offsets above the return address.
- `ret` is just "pop and jump"; it has no notion of "valid".
- `struct thread` lives at a page-aligned base, so `pg_round_down(%esp)`
  recovers the current thread from any kernel-stack address.
- The layout of `switch_threads_frame` matches exactly what
  `switch_threads`'s prologue pushes — so a real preemption and a
  forged stack are bit-identical.

Once you see that the scheduler treats "real" and "forged" stacks
uniformly, the first-switch paradox dissolves: there is no first
switch, only a switch that happens to land on a stack someone
prepared earlier.
