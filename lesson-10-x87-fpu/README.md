# Lesson 10 — The x87 FPU

**Goal:** understand the original floating-point unit — a stack machine
bolted onto a register machine. You will rarely *write* x87 today, but
you will *read* it in old code, and its quirks explain several oddities in
modern floating point.

## 1. Historical context, briefly

The 8087 was a separate physical chip you bought and installed in a socket
on your IBM PC — a coprocessor. It watched the CPU's instruction stream
for its own opcodes (prefixed `ESC`, which is why every x87 mnemonic
starts with `f`). Since it was optional, code had to detect its presence.

It was integrated into the CPU from the 486DX onward. On x86-64 it's
always present — but **almost entirely superseded by SSE** (Lesson 11),
which the ABI mandates for passing float arguments. x87 survives for one
real reason: the 80-bit `long double` type, which SSE cannot do.

## 2. The register stack

Instead of named registers, x87 has **eight 80-bit registers arranged as a
stack**: `st0` through `st7`, where `st0` is always the top.

```
push a value  →  everything shifts down, new value becomes st0
pop           →  st0 discarded, everything shifts up
```

This is genuinely a stack, not an array with an index — `st3` means "three
below the top," and which physical register that is changes as you push
and pop. This is what makes x87 code hard to read and hard for compilers
to schedule.

**Stack overflow is silent-ish:** pushing a 9th value sets the
invalid-operation flag and produces an "indefinite" NaN. Unbalanced x87
code degrades into garbage rather than crashing.

## 3. Loading and storing

```nasm
fld  qword [x]        ; push a 64-bit double onto the stack
fld  dword [y]        ; push a 32-bit float (converted to 80-bit internally)
fild qword [n]        ; push an INTEGER, converting to float ('i' = integer)
fld  st1              ; duplicate st1 onto the top

fst  qword [dst]      ; store st0, leave it on the stack
fstp qword [dst]      ; store st0 and POP
fistp qword [dst]     ; convert to integer, store, and pop

fld1                  ; push the constant 1.0
fldz                  ; push 0.0
fldpi                 ; push π
fldl2e                ; push log₂(e)
fldln2                ; push ln(2)
```

The `p` suffix means "and pop." You'll use `fstp` far more than `fst` —
leaving values on the stack is how you overflow it.

## 4. Arithmetic

```nasm
fadd  st0, st1        ; st0 = st0 + st1
faddp st1, st0        ; st1 = st1 + st0, then pop  (very common)
fadd  qword [x]       ; st0 = st0 + [x]
fiadd dword [n]       ; add an integer from memory

fsub / fsubp / fsubr / fsubrp        ; 'r' = reversed operands
fmul / fmulp / fimul
fdiv / fdivp / fdivr / fdivrp

fsqrt                 ; st0 = sqrt(st0)
fabs                  ; st0 = |st0|
fchs                  ; st0 = -st0
fsin / fcos / fptan / fpatan
f2xm1                 ; 2^st0 - 1
fyl2x                 ; st1 * log₂(st0), pops
fprem / fprem1        ; partial remainder
frndint               ; round to integer per the control word
```

The `r` (reversed) variants exist because subtraction and division aren't
commutative and the stack fixes operand positions. `fsub` gives
`st0 - src`; `fsubr` gives `src - st0`.

## 5. Comparison

Old style — sets the FPU's *own* status word, which you must transfer:

```nasm
fcom  st1             ; compare, set FPU condition codes C0/C2/C3
fnstsw ax             ; copy FPU status word into ax
sahf                  ; load ah into the CPU flags
ja .greater           ; now normal conditional jumps work
```

Modern style (Pentium Pro and later) — sets CPU flags directly:

```nasm
fcomi  st0, st1       ; compare and set ZF/CF/PF directly
fcomip st0, st1       ; ...and pop
fucomi st0, st1       ; 'u' = unordered-safe (NaN-tolerant)
ja .greater
```

Use the `fcomi` family. The `fnstsw`/`sahf` dance is legacy.

**NaN handling:** if either operand is NaN, the comparison is *unordered*
and sets PF (parity flag). Since `ja`/`jb` don't check PF, an unordered
comparison silently answers "below." Robust float comparison checks `jp`
first. This is inherent to IEEE 754, not an x86 wart — SSE has the same
issue.

## 6. The control word — precision and rounding

```nasm
fstcw word [ctrl]     ; store the control word
fldcw word [ctrl]     ; load it
```

Bits 10-11 set the rounding mode: 00 nearest-even (default), 01 down,
10 up, 11 toward zero. Bits 8-9 set precision: 00 single, 10 double,
11 extended (default on Linux).

The classic use is forcing truncation for float→int conversion, since
`fistp` obeys the current rounding mode but C's `(int)` cast requires
truncation. `fisttp` (SSE3) does truncation unconditionally and is the
better answer where available.

## 7. The 80-bit surprise

x87 computes internally at **80-bit extended precision** regardless of
whether your values are 32- or 64-bit. This means intermediate results are
more accurate than the declared types — and results change depending on
whether a value got spilled to memory (truncating it to 64 bits) or stayed
in a register.

This is the notorious "x87 excess precision" problem: the *same* C
expression can give different answers at different optimization levels.
SSE, which computes at exactly the declared width, eliminated it. It's one
of the main reasons x87 was abandoned.

## 8. Program

`fpu.asm`:

```nasm
default rel
global main
extern printf

section .data
    a       dq  3.0
    b       dq  4.0
    n       dq  7
    fmt     db  "hypot=%f  int_as_float=%f  sqrt2=%f", 10, 0

section .bss
    hyp     resq 1
    conv    resq 1
    root    resq 1

section .text
main:
    push rbp
    mov rbp, rsp

    ; ---- hypotenuse: sqrt(a² + b²) ----
    fld qword [a]           ; st0 = 3.0
    fmul st0, st0            ; st0 = 9.0
    fld qword [b]             ; st0 = 4.0, st1 = 9.0
    fmul st0, st0              ; st0 = 16.0
    faddp st1, st0              ; st0 = 25.0  (and pop)
    fsqrt                        ; st0 = 5.0
    fstp qword [hyp]              ; store and pop — stack now empty

    ; ---- integer 7 -> float ----
    fild qword [n]
    fstp qword [conv]

    ; ---- sqrt(2) ----
    fld1
    fld1
    faddp st1, st0                 ; 2.0
    fsqrt
    fstp qword [root]

    ; ---- print all three (three doubles => al = 3) ----
    lea rdi, [fmt]
    movsd xmm0, [hyp]
    movsd xmm1, [conv]
    movsd xmm2, [root]
    mov eax, 3
    call printf

    xor eax, eax
    leave
    ret
```

```sh
nasm -f elf64 fpu.asm -o fpu.o -l fpu.lst
gcc fpu.o -o fpu
./fpu
```

Note the ending: results are stored to memory and reloaded into **XMM**
registers for `printf`, because the ABI passes floats in XMM, not on the
x87 stack. Even when you compute with x87, you interoperate with SSE.

## 9. GDB: `info float`

```gdb
info float
```

This is the lesson's new command. It prints all eight `st` registers with
their values, plus the status word, control word, tag word, and — crucially
— **TOP**, the index of which physical register is currently `st0`.

Sample output shape:

```
R7: Valid   0x4001c800000000000000 +25
=>R6: Empty 0x00000000000000000000
Status Word:  0x3800  TOP: 7
Control Word: 0x037f  IM DM ZM OM UM PM  PC: Extended  RC: Nearest
Tag Word:     0x3fff
```

Read the **Tag Word** and the `Valid`/`Empty` markers to see how deep the
stack is. This is how you catch unbalanced push/pop — the single most
common x87 bug.

Individual registers:

```gdb
p $st0
p $st1
p/x $st0            # the raw 80-bit encoding
```

### The workflow

```gdb
break main
run
display/i $pc
info float
```

Then `si` through the hypotenuse calculation, running `info float` after
each step. Watch values shift positions as `faddp` pops — this is the
clearest possible demonstration of what "stack register" means.

Check the stack is empty before returning:

```gdb
info float          # all registers should show Empty
```

If they don't, you have leaked FPU stack slots.

## Exercises

**10.1 — Watch the stack shift.** Step through the hypotenuse code with
`info float` after every instruction. Record, at each step, which physical
register is `st0`. Explain how `faddp` changed TOP.

**10.2 — Leak the stack deliberately.** Write code that does nine `fld1`
instructions in a row. Run `info float` and describe what happened to the
ninth value. Check the status word for the invalid-operation flag.

**10.3 — Contract: `f_average`.**

> `double f_average(const double *arr, long n)`
> Sum with x87, divide by n (use `fild` for the count), return in `xmm0`.
> The x87 stack must be empty on return.

Verify emptiness with `info float` at the `ret`.

**10.4 — `fsubr` vs `fsub`.** Compute `10.0 - 3.0` and `3.0 - 10.0` using
the same stack setup, changing only which subtraction variant you use.
Confirm both results with `p $st0`.

**10.5 — NaN comparison trap.** Load a NaN (`dq 0x7FF8000000000000`) and
compare it against 1.0 with `fcomi`. Check `$eflags` and confirm PF is
set. Then write a comparison that correctly detects the unordered case
with `jp` before checking `ja`/`jb`.

**10.6 — Rounding modes.** Store the control word with `fstcw`, modify
bits 10-11 to "toward zero," reload with `fldcw`, and convert 2.7 to an
integer with `fistp`. Compare against the default nearest-even result.
Inspect the control word in GDB with `info float` before and after.

**10.7 — Excess precision.** Compute `(a*b)/c` two ways: entirely on the
x87 stack, and with the intermediate `a*b` stored to a `qword` in memory
and reloaded. Use values where the difference shows (try `a = 1e300`,
`b = 1e-300`, `c = 3.0`). Explain the discrepancy.
