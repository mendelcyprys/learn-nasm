# x86-64 Assembly with NASM, GDB, and Linux

A hands-on course in twelve lessons. Everything runs inside your Alpine
Linux VM in UTM. Every lesson builds real, runnable binaries; every
lesson teaches new GDB skills; every lesson ends with exercises.

This course is a **precursor to a SIMD course**. Lessons 11 and 12 in
particular lay the groundwork — the XMM register file, the SSE
instruction naming scheme, and CPU feature detection are all introduced
here as scalar concepts before the SIMD course generalizes them to
vectors.

## Prerequisites

Inside the Alpine VM:

```sh
apk update
apk add build-base nasm gdb nano
```

`build-base` supplies `gcc`, `ld`, `objdump`, `readelf`, and `nm`.
`nano` is optional if you already use `vi`.

Verify:

```sh
nasm -v
gdb --version
uname -m          # should print x86_64
```

## Lessons

| # | Topic | New GDB skills |
|---|---|---|
| [01](lesson-01-setup-and-first-program/) | The toolchain, sections, your first binary | `break`, `run`, `stepi`, `info registers`, `layout asm` |
| [02](lesson-02-nasm-language-and-data/) | NASM syntax, data directives, `mov`, operand sizes | `x` (examine memory), `print`, `set var` |
| [03](lesson-03-arithmetic-and-flags/) | `add`, `sub`, `mul`, `div`, and the FLAGS register | Reading `eflags`, `display` |
| [04](lesson-04-control-flow/) | `cmp`, `test`, jumps, loops | Conditional breakpoints, `tbreak`, breakpoint management |
| [05](lesson-05-the-stack/) | `push`, `pop`, `rsp`, stack frames | Examining the stack, `backtrace`, `frame` |
| [06](lesson-06-functions-and-the-abi/) | System V AMD64 ABI: arg registers, callee-saved, alignment | `finish`, `up`/`down`, `info frame` |
| [07](lesson-07-addressing-and-memory/) | `lea`, addressing modes, arrays, string instructions | Watchpoints (`watch`, `rwatch`, `awatch`) |
| [08](lesson-08-linux-syscalls/) | The kernel interface in depth, file I/O, errno | `catch syscall` |
| [09](lesson-09-calling-c-and-libc/) | Linking with `gcc`, calling `printf`/`malloc`, the PLT | `ni` vs `si`, breaking in library code |
| [10](lesson-10-x87-fpu/) | The x87 FPU stack, `fld`/`fadd`/`fstp` | `info float` |
| [11](lesson-11-scalar-sse/) | XMM registers, scalar SSE float math, conversions | `info registers xmm`, format specifiers |
| [12](lesson-12-bits-and-cpu-features/) | Shifts, masks, `popcnt`, BMI, `cpuid` — and the road to SIMD | `.gdbinit`, `define`, scripting |

## How to work through this

Do the exercises. Assembly is not a subject you can absorb by reading —
the whole point is that you build an accurate mental model of what the
machine does with each byte, and that only comes from writing code that
doesn't work and then finding out why in a debugger.

Each lesson's exercises are graded roughly easy → hard. If an exercise
specifies a *contract* (what's in which register on entry and exit),
treat that contract as binding — Lesson 06 explains why that discipline
matters, and every lesson after it assumes you're following it.

## Conventions used throughout

- All code is **NASM syntax**, Intel-style: `mov destination, source`.
- All builds target **x86-64 Linux ELF**.
- Registers written lowercase (`rax`), instructions lowercase (`mov`).
- Where a lesson says "verify this yourself," actually do it. Those
  are the moments the concept becomes concrete rather than asserted.
