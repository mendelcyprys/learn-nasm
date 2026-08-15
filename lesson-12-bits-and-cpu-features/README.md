# Lesson 12 — Bit Manipulation, CPU Features, and the Road to SIMD

**Goal:** the bitwise instruction set, runtime CPU feature detection, and
the GDB scripting that makes long sessions bearable. This lesson closes
the course and hands off to SIMD.

## 1. Bitwise logic

```nasm
and rax, rbx        ; bitwise AND — clear bits
or  rax, rbx        ; set bits
xor rax, rbx        ; toggle bits
not rax             ; invert all bits (does NOT affect flags)
```

Standard idioms:

```nasm
and rax, 0xFF           ; keep the low byte
or  rax, 0x80           ; set bit 7
xor rax, 0x80           ; toggle bit 7
and rax, ~0x80          ; clear bit 7 (NASM computes ~ at assembly time)
test rax, 0x80          ; TEST bit 7 without modifying rax
and rax, rbx            ; ...where rbx is a mask
```

`test` is `and` that throws the result away — same relationship as `cmp`
to `sub`. Use it for bit checks.

## 2. Shifts and rotates

```nasm
shl rax, 3          ; shift left 3 — multiply by 8, MSBs fall into CF
shr rax, 3          ; shift right LOGICAL — zeros shifted in (unsigned /8)
sar rax, 3          ; shift right ARITHMETIC — sign bit replicated (signed /8)

rol rax, 8          ; rotate left — bits wrap around
ror rax, 8          ; rotate right
rcl / rcr           ; rotate through the carry flag

shl rax, cl         ; variable shift count — must be in CL specifically
shld rax, rbx, 4    ; double-precision shift: shift rax left, filling from rbx
shrd rax, rbx, 4
```

**`shr` vs `sar` is a real bug source.** For signed division by a power of
two, `sar` is *almost* right — it rounds toward negative infinity, whereas
C's `/` rounds toward zero. `-7 >> 1` gives `-4` with `sar`, but `-7 / 2`
is `-3` in C. Compilers add a correction; be careful if you hand-roll it.

The variable-count forms require the count in `cl` (the low byte of
`rcx`) — a hardware restriction inherited from the 8086. BMI2 lifts it
(see below).

## 3. Bit test and scan

```nasm
bt  rax, 5          ; copy bit 5 into CF
bts rax, 5          ; copy to CF, then SET bit 5
btr rax, 5          ; copy to CF, then RESET (clear) bit 5
btc rax, 5          ; copy to CF, then COMPLEMENT bit 5

bsf rbx, rax        ; Bit Scan Forward: index of lowest set bit; ZF=1 if rax==0
bsr rbx, rax        ; Bit Scan Reverse: index of highest set bit
```

`bsf`/`bsr` leave the destination **undefined** when the source is zero —
check ZF, don't assume. `tzcnt`/`lzcnt` (below) fixed this by defining the
zero case as "operand width."

## 4. Population count and the modern extensions

These are not part of baseline x86-64 — they're extensions with their own
CPUID feature bits. **Using them on a CPU that lacks them causes SIGILL**
(illegal instruction).

```nasm
popcnt rax, rbx     ; count set bits          [SSE4.2 feature bit]
lzcnt  rax, rbx     ; count leading zeros     [LZCNT/ABM]
tzcnt  rax, rbx     ; count trailing zeros    [BMI1]
```

**BMI1:**
```nasm
andn rax, rbx, rcx  ; rax = (~rbx) & rcx
blsi rax, rbx       ; extract lowest set bit:  rbx & -rbx
blsr rax, rbx       ; clear lowest set bit:    rbx & (rbx-1)
blsmsk rax, rbx     ; mask up to lowest set bit
bextr rax, rbx, rcx ; extract a bitfield
```

**BMI2:**
```nasm
shlx rax, rbx, rcx  ; shift with count in ANY register — not just cl
shrx / sarx
mulx rax, rbx, rcx  ; unsigned multiply that doesn't touch flags
pdep rax, rbx, rcx  ; parallel bit deposit
pext rax, rbx, rcx  ; parallel bit extract
```

`blsr` (clear lowest set bit) is the heart of the fast "iterate over set
bits" loop:

```nasm
.loop:
    test rbx, rbx
    jz .done
    tzcnt rcx, rbx        ; rcx = index of lowest set bit
    ; ... process bit rcx ...
    blsr rbx, rbx          ; clear it
    jmp .loop
.done:
```

## 5. `cpuid` — runtime feature detection

Since these instructions may not exist, real code checks first. `cpuid`
takes a "leaf" in `eax` (and a subleaf in `ecx`) and returns feature bits
across `eax`/`ebx`/`ecx`/`edx`.

**`cpuid` destroys all four of those registers**, so save what you need.

Key leaves and bits:

| Leaf | Register | Bit | Feature |
|---|---|---|---|
| 1 | `edx` | 25 | SSE |
| 1 | `edx` | 26 | SSE2 (guaranteed on x86-64) |
| 1 | `ecx` | 0 | SSE3 |
| 1 | `ecx` | 19 | SSE4.1 |
| 1 | `ecx` | 20 | SSE4.2 |
| 1 | `ecx` | 23 | **POPCNT** |
| 1 | `ecx` | 28 | **AVX** |
| 7 (ecx=0) | `ebx` | 3 | **BMI1** |
| 7 (ecx=0) | `ebx` | 5 | **AVX2** |
| 7 (ecx=0) | `ebx` | 8 | **BMI2** |
| 7 (ecx=0) | `ebx` | 16 | AVX-512F |

You'll use this table again on day one of the SIMD course — dispatching to
an AVX2 path or an SSE fallback at runtime is standard practice.

You can also just look:

```sh
cat /proc/cpuinfo | grep flags | tr ' ' '\n' | sort | head -50
```

Under UTM's emulation the flag set may be narrower than your host CPU's —
worth checking before writing anything AVX-dependent.

## 6. Program

`bits.asm`:

```nasm
default rel
global main
extern printf

section .data
    value   dq  0b1011010100000000
    fmt1    db  "value=%lx popcnt=%ld tz=%ld lz=%ld", 10, 0
    fmt2    db  "SSE4.2=%ld AVX=%ld BMI1=%ld BMI2=%ld", 10, 0

section .text

; has_feature_leaf1_ecx(rdi = bit index) -> rax = 0 or 1
feature_leaf1_ecx:
    push rbx
    mov r8, rdi
    mov eax, 1
    xor ecx, ecx
    cpuid                        ; destroys eax ebx ecx edx
    mov rax, rcx
    mov rcx, r8
    shr rax, cl
    and rax, 1
    pop rbx
    ret

; feature_leaf7_ebx(rdi = bit index) -> rax = 0 or 1
feature_leaf7_ebx:
    push rbx
    mov r8, rdi
    mov eax, 7
    xor ecx, ecx
    cpuid
    mov rax, rbx
    mov rcx, r8
    shr rax, cl
    and rax, 1
    pop rbx
    ret

main:
    push rbp
    mov rbp, rsp
    push r12
    push r13
    push r14
    push r15
    sub rsp, 8                   ; realign: 4 pushes + rbp = odd count

    ; ---- bit counting ----
    mov rax, [value]
    popcnt r12, rax
    tzcnt  r13, rax
    lzcnt  r14, rax

    lea rdi, [fmt1]
    mov rsi, [value]
    mov rdx, r12
    mov rcx, r13
    mov r8,  r14
    xor eax, eax
    call printf

    ; ---- feature detection ----
    mov rdi, 20
    call feature_leaf1_ecx        ; SSE4.2
    mov r12, rax
    mov rdi, 28
    call feature_leaf1_ecx         ; AVX
    mov r13, rax
    mov rdi, 3
    call feature_leaf7_ebx          ; BMI1
    mov r14, rax
    mov rdi, 8
    call feature_leaf7_ebx           ; BMI2
    mov r15, rax

    lea rdi, [fmt2]
    mov rsi, r12
    mov rdx, r13
    mov rcx, r14
    mov r8,  r15
    xor eax, eax
    call printf

    add rsp, 8
    pop r15
    pop r14
    pop r13
    pop r12
    xor eax, eax
    leave
    ret
```

```sh
nasm -f elf64 bits.asm -o bits.o -l bits.lst
gcc bits.o -o bits
./bits
```

If this dies with SIGILL, your emulated CPU lacks `popcnt`, `tzcnt`, or
`lzcnt` — which is itself the lesson. Comment them out, run the feature
detection alone, and see what you actually have.

Note the prologue: four `push`es plus `push rbp` is an odd number of
8-byte pushes, so `sub rsp, 8` restores 16-byte alignment before any
`call`. This is exercise 6.5 made real.

## 7. GDB: scripting and persistence

### `~/.gdbinit`

Create it once and stop retyping setup:

```gdb
set disassembly-flavor intel
set print pretty on
set pagination off
set confirm off
set history save on
set history size 10000
```

`set pagination off` alone will improve your life measurably.

### Custom commands

Add to `~/.gdbinit`:

```gdb
define ctx
    printf "---- registers ----\n"
    info registers rax rbx rcx rdx rsi rdi rsp rbp
    printf "---- flags ----\n"
    info registers eflags
    printf "---- stack ----\n"
    x/6gx $rsp
    printf "---- next ----\n"
    x/5i $pc
end
document ctx
Show registers, flags, stack, and upcoming instructions.
end
```

Now `ctx` at any breakpoint gives you a full context dump. Then:

```gdb
define sic
    stepi
    ctx
end
```

`sic` = step one instruction and show everything. This is, roughly, what
the fancy GDB frontends (GEF, pwndbg, peda) do — and you can now build
your own.

### Hooks

```gdb
define hook-stop
    x/3i $pc
end
```

Runs automatically every time the program stops, on any breakpoint or
step.

### Scripted runs

```gdb
gdb -x commands.gdb ./bits          # run a script file
gdb -batch -ex "break main" -ex run -ex "info registers" ./bits
```

`-batch` with `-ex` is how you automate checks — useful for verifying an
exercise's contract repeatedly without an interactive session.

### Logging

```gdb
set logging file session.txt
set logging on
```

Everything GDB prints also goes to the file. Invaluable when stepping
through hundreds of instructions and wanting to diff two runs.

## 8. Where this goes next: SIMD

You now have everything the SIMD course assumes:

- **XMM registers** (Lesson 11) — SIMD widens these to YMM (256-bit, AVX)
  and ZMM (512-bit, AVX-512), but the model is identical.
- **The `ss`/`sd`/`ps`/`pd` naming scheme** (Lesson 11) — you already know
  `addsd`; `addpd` does two at once and `vaddpd` on YMM does four.
- **Branchless thinking** (`cmov`, `minsd`/`maxsd`) — SIMD has no
  per-lane branches at all. Everything is compute-both-sides-and-blend.
- **Bit masks** (this lesson) — SIMD comparisons produce all-ones/all-zeros
  masks per lane, and you select with `andpd`/`andnpd`/`orpd` or
  `blendvpd`. The bitwise reflexes you built here are the mechanism.
- **Alignment** (Lessons 06, 11) — `movaps` faults on unaligned data, and
  aligned loads matter more as vectors get wider.
- **CPUID dispatch** (this lesson) — you cannot ship AVX2 code without
  checking for it.
- **The ABI** (Lesson 06) — all vector registers are caller-saved, which
  shapes how you structure SIMD loops around function calls.
- **GDB vector inspection** (`p $xmm0.v2_double`, Lesson 11) — the same
  syntax extends to `$ymm0.v4_double` and `$zmm0.v8_double`.

## Exercises

**12.1 — `shr` vs `sar`.** Compute `-8 >> 1` both ways and print the
results. Then do `-7 >> 1` and explain why `sar` gives -4 while C's `/2`
gives -3. Write a correction sequence that matches C.

**12.2 — Contract: `count_bits`.**

> `long count_bits(unsigned long x)`
> Return the number of set bits. Write it **three** ways: a naive shift
> loop, the `blsr` loop shown above, and a single `popcnt`.

Time all three on a large loop. The ratio should be dramatic.

**12.3 — Contract: `is_power_of_two`.**

> `long is_power_of_two(unsigned long x)` → 1 or 0.
> Use exactly one arithmetic operation and one test. (Hint: `blsr`, or
> `x & (x-1)`.)

**12.4 — Byte swap.** Write `bswap64` using rotates and masks, then
compare against the dedicated `bswap` instruction. Verify with `p/x` in
GDB that you get the endianness reversal you expect.

**12.5 — Contract: `extract_field`.**

> `extract_field(rdi = value, rsi = start bit, rdx = width) -> rax`
> Extract those bits, zero-extended. Write it with shifts and masks, then
> again with `bextr` if BMI1 is available.

**12.6 — Full CPUID dump.** Extend the feature-detection program to check
and print all sixteen features in the table above. Compare your output
against `/proc/cpuinfo` in the VM.

**12.7 — Build your GDB config.** Write a `~/.gdbinit` with at least three
custom commands you'd actually use: a context dump, a stack dump, and one
that prints all XMM registers as doubles. Use it for the remaining
exercises.

**12.8 — Capstone.** Write a program that:
1. Uses `cpuid` to detect SSE4.2.
2. If present, counts set bits across an array of 1000 quadwords using
   `popcnt`.
3. If absent, falls back to a `blsr` loop.
4. Prints the total and which path it took, via `printf`.
5. Follows the ABI exactly — verify with the entry/exit register
   comparison from Lesson 06.

This is a complete, correct, real-world-shaped assembly program:
runtime dispatch, a fast path, a portable fallback, and clean C interop.
It is also, structurally, exactly what a SIMD kernel looks like — which is
where you go next.
