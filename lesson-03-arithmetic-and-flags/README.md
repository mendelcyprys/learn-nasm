# Lesson 03 — Arithmetic and the FLAGS Register

**Goal:** the integer arithmetic instructions, and the invisible register
that every one of them writes to.

## 1. The basic operations

```nasm
add rax, rbx        ; rax = rax + rbx
sub rax, rbx        ; rax = rax - rbx
inc rax             ; rax = rax + 1
dec rax             ; rax = rax - 1
neg rax             ; rax = -rax  (two's complement negation)

adc rax, rbx        ; rax = rax + rbx + carry-flag
sbb rax, rbx        ; rax = rax - rbx - carry-flag
```

`adc` and `sbb` exist to chain arithmetic across multiple registers —
that's how you add two 128-bit numbers on a 64-bit machine.

All of these accept the usual operand combinations: register/register,
register/immediate, register/memory, memory/register, memory/immediate.
What you **cannot** do is memory-to-memory — no x86 arithmetic
instruction has two memory operands. Load into a register first.

## 2. Multiplication

Two families, and the difference matters:

```nasm
; ---- one-operand form: result is DOUBLE width ----
mov rax, 1000000000000
mov rbx, 1000000000000
mul rbx             ; unsigned: rdx:rax = rax * rbx  (128-bit result!)
imul rbx            ; signed version of the same thing
```

The one-operand form implicitly uses `rax` as one input and writes the
full 128-bit product across `rdx:rax` — high half in `rdx`, low half in
`rax`. This is how you multiply without losing bits.

```nasm
; ---- two- and three-operand forms: result is SAME width, signed ----
imul rax, rbx           ; rax = rax * rbx        (truncated to 64 bits)
imul rax, rbx, 10       ; rax = rbx * 10
```

There is no two-operand `mul` — only `imul` has the convenient forms.
For the low 64 bits of a product, signed and unsigned multiplication give
identical bit patterns, so using `imul` for both is correct and standard.

## 3. Division — the instruction with sharp edges

```nasm
; unsigned: divides the 128-bit value rdx:rax by the operand
xor rdx, rdx        ; ***REQUIRED***: clear the high half first
mov rax, 100
mov rbx, 7
div rbx             ; rax = quotient (14), rdx = remainder (2)
```

Three things will bite you:

1. **`div` reads `rdx:rax` as a single 128-bit number.** If `rdx` contains
   garbage from earlier, your result is garbage. Always `xor rdx, rdx`
   before an unsigned `div`.
2. **For signed division use `idiv` — and sign-extend, don't zero.** The
   correct preamble is `cqo` (sign-extend `rax` into `rdx`), not
   `xor rdx, rdx`:
   ```nasm
   mov rax, -100
   cqo                 ; rdx = 0xFFFF... (all sign bits)
   mov rbx, 7
   idiv rbx            ; rax = -14, rdx = -2
   ```
3. **Divide-by-zero, or a quotient too large to fit, raises `#DE`** — the
   process dies with SIGFPE. There is no flag to check; it's a hardware
   exception. Guard your divisors.

`div` is also *slow* — tens of cycles, versus one for `add`. Dividing by a
power of two? Use a shift (Lesson 12).

## 4. FLAGS: the register you never name

Every arithmetic instruction sets bits in the 64-bit `RFLAGS` register as
a side effect. The six that matter:

| Flag | Name | Set when |
|---|---|---|
| **CF** | Carry | Unsigned overflow — result didn't fit |
| **OF** | Overflow | *Signed* overflow — sign came out wrong |
| **ZF** | Zero | Result was exactly zero |
| **SF** | Sign | Result's top bit is 1 (negative, if signed) |
| **PF** | Parity | Low byte has an even number of set bits |
| **AF** | Adjust | BCD carry out of bit 3 (you can ignore this) |

The key insight: **the hardware doesn't know whether your numbers are
signed.** It computes the bits and sets *both* CF and OF, describing what
the result would mean under each interpretation. You decide which flag to
believe by choosing which conditional jump to use (Lesson 04).

Example — `mov al, 0xFF` then `add al, 1`:
- The result is `0x00`. ZF is set.
- Read as unsigned: 255 + 1 = 256, doesn't fit in 8 bits → **CF set**.
- Read as signed: -1 + 1 = 0, perfectly correct → **OF clear**.

Same bits. Two truths. Both flags recorded.

## 5. `cmp` and `test`: arithmetic you throw away

```nasm
cmp rax, rbx        ; computes rax - rbx, sets flags, DISCARDS the result
test rax, rax       ; computes rax AND rax, sets flags, DISCARDS the result
```

These exist purely to set flags before a conditional jump. `test rax, rax`
is the idiomatic "is this zero?" check — cheaper than `cmp rax, 0` and
what you'll see everywhere.

## 6. Program: 128-bit addition and division

`arith.asm`:

```nasm
global _start

section .data
    lo_a    dq  0xFFFFFFFFFFFFFFFF
    hi_a    dq  0x0000000000000001
    lo_b    dq  0x0000000000000002
    hi_b    dq  0x0000000000000000

section .text
_start:
    ; ---- 128-bit add: (hi_a:lo_a) + (hi_b:lo_b) ----
    mov rax, [lo_a]
    mov rdx, [hi_a]
    add rax, [lo_b]         ; low halves; sets CF if it wrapped
    adc rdx, [hi_b]         ; high halves PLUS that carry
    ; rdx:rax now holds the true 128-bit sum

    ; ---- unsigned division ----
    mov rax, 1000
    xor rdx, rdx             ; MUST clear
    mov rcx, 7
    div rcx                   ; rax = 142, rdx = 6

    ; ---- signed division ----
    mov rax, -1000
    cqo                        ; sign-extend into rdx
    mov rcx, 7
    idiv rcx                    ; rax = -142, rdx = -6

    mov rax, 60
    xor rdi, rdi
    syscall
```

```sh
nasm -f elf64 arith.asm -o arith.o -l arith.lst
ld arith.o -o arith
gdb ./arith
```

## 7. GDB: reading the flags

```gdb
info registers eflags
```

GDB prints the raw value **and** decodes it into names:

```
eflags   0x246   [ PF ZF IF ]
```

The bracketed list is the flags currently *set*. Anything absent is
clear. `IF` (interrupt enable) is always on in user code — ignore it.

Other ways to reach the flags:

```gdb
p $eflags              # symbolic list
p/x $eflags            # raw hex
p ($eflags >> 0) & 1   # CF, bit 0
p ($eflags >> 6) & 1   # ZF, bit 6
p ($eflags >> 7) & 1   # SF, bit 7
p ($eflags >> 11) & 1  # OF, bit 11
```

You can also **set** flags to force a branch during testing:

```gdb
set $eflags |= (1 << 6)     # force ZF on
set $eflags &= ~(1 << 6)    # force ZF off
```

### The workflow to actually use

Flags change on almost every instruction, so watching them means using
`display`:

```gdb
break _start
run
display $eflags
display/x $rax
display/x $rdx
```

Now every `stepi` reprints all three automatically. Step through the
128-bit addition and watch CF appear on the `add` (because the low half
wrapped) and get consumed by the `adc`.

## Exercises

**3.1 — Watch a carry happen.** Write a program that sets `al` to `0xFF`
and adds 1. Step through it in GDB with `display $eflags` on, and confirm
CF is set and OF is not. Then do `mov al, 0x7F` / `add al, 1` and confirm
the opposite: OF set, CF clear. Explain both.

**3.2 — 128-bit multiply.** Multiply 0x123456789ABCDEF by itself using
one-operand `mul`. Read the full 128-bit answer out of `rdx:rax` in GDB
using `p/x $rdx` and `p/x $rax`. Verify the low half against a two-operand
`imul` — and confirm the high half is what you lose by using it.

**3.3 — Break `div` on purpose.** Write a program that does `div` with
`rdx` deliberately left non-zero, and observe what happens (SIGFPE — GDB
will catch it and tell you). Then fix it with `xor rdx, rdx` and see it
work. This failure mode is worth experiencing once, deliberately.

**3.4 — Contract: modulo.** Write a fragment satisfying:

> On entry: `rdi` = dividend (unsigned), `rsi` = divisor (unsigned,
> guaranteed non-zero).
> On exit: `rax` = `rdi % rsi`. `rdx` may be clobbered. All other
> registers unchanged.

Test it in GDB by setting `rdi` and `rsi` by hand with `set $rdi = ...`
before letting it run.

**3.5 — Signed vs unsigned divergence.** Take the byte `0xFF`. Write code
that computes `0xFF + 1` and then, separately, `movsx` it to 64 bits and
add 1. Show in GDB that one gives 0 and the other gives 0. Then repeat
with `0x80` and explain why the two interpretations diverge there.

**3.6 — Average without overflow.** Write a fragment that computes the
average of two 64-bit unsigned numbers in `rdi` and `rsi`, returning it in
`rax`, that is *correct even when `rdi + rsi` overflows 64 bits*. (Hint:
you have `adc`, and you have a 128-bit dividend for `div`.)
