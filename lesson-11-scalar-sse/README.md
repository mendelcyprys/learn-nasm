# Lesson 11 — Scalar SSE Floating Point

**Goal:** learn the XMM register file and the SSE instruction naming
scheme using *scalar* (one-value-at-a-time) operations. This is the
direct on-ramp to the SIMD course — same registers, same mnemonics, same
encoding rules. The only thing SIMD adds is doing all four (or eight, or
sixteen) lanes at once instead of one.

## 2. The XMM registers

Sixteen 128-bit registers: `xmm0` through `xmm15`.

For **scalar** operations, only the low 32 or 64 bits are used and the
upper bits are ignored (or preserved, depending on the instruction). For
**packed** operations — the SIMD course — all 128 bits hold multiple
values at once:

```
xmm0 as 2 doubles:   [  double  ][  double  ]
xmm0 as 4 floats:    [ f ][ f ][ f ][ f ]
xmm0 as 16 bytes:    [b][b][b][b][b][b][b][b][b][b][b][b][b][b][b][b]
```

Same 128 bits, different interpretation chosen by which instruction you
use. That idea *is* SIMD.

**All XMM registers are caller-saved.** There are no callee-saved vector
registers in the System V ABI. Anything in an XMM register is gone after a
`call`.

## 2. The naming scheme — learn this, not a list

Every SSE arithmetic mnemonic decomposes into `<operation><type>`:

| Suffix | Meaning |
|---|---|
| `ss` | **S**calar **S**ingle — one 32-bit float |
| `sd` | **S**calar **D**ouble — one 64-bit double |
| `ps` | **P**acked **S**ingle — four 32-bit floats |
| `pd` | **P**acked **D**ouble — two 64-bit doubles |

So:

```nasm
addss xmm0, xmm1        ; add one float
addsd xmm0, xmm1        ; add one double
addps xmm0, xmm1        ; add FOUR floats at once     ← SIMD
addpd xmm0, xmm1        ; add TWO doubles at once      ← SIMD
```

Learn the scalar forms now and the packed forms are free later. That is
the entire reason this lesson exists before the SIMD course.

## 3. Moving data

```nasm
movss xmm0, [x]         ; load one float (zeroes the rest of xmm0)
movsd xmm0, [x]         ; load one double
movss [x], xmm0         ; store one float
movsd [x], xmm0         ; store one double

movaps xmm0, xmm1       ; move all 128 bits, ALIGNED address required
movups xmm0, xmm1       ; move all 128 bits, unaligned OK
movdqa / movdqu         ; same, integer-flavoured

movq  rax, xmm0         ; move 64 bits from xmm to a GP register
movq  xmm0, rax         ; and back
movd  eax, xmm0         ; 32-bit version
```

**Alignment:** `movaps`/`movdqa` require the address to be 16-byte
aligned and **fault** otherwise. `movups`/`movdqu` don't. On modern CPUs
the unaligned versions are just as fast when the data happens to be
aligned, so prefer them unless you're certain — but you'll meet the
aligned forms constantly in compiler output, and in the SIMD course.

Note the mnemonic collision: `movsd` is *move scalar double* in SSE, and
also *move string doubleword* from Lesson 07. NASM disambiguates by
operands. Annoying but harmless.

## 4. Arithmetic

```nasm
addsd xmm0, xmm1        ; xmm0 += xmm1
subsd xmm0, xmm1
mulsd xmm0, xmm1
divsd xmm0, xmm1
sqrtsd xmm0, xmm1       ; xmm0 = sqrt(xmm1)
minsd xmm0, xmm1        ; branchless minimum
maxsd xmm0, xmm1        ; branchless maximum
```

Note `minsd`/`maxsd`: branchless by construction. In SIMD there is no
per-element branching at all, so min/max instructions are how conditionals
get expressed. Getting used to that mindset now pays off later.

There is no `negsd`. Negation is done by flipping the sign bit with a
bitwise XOR against a mask:

```nasm
section .data
    align 16
    signmask dq 0x8000000000000000, 0

section .text
    xorpd xmm0, [signmask]      ; flip the sign bit
```

The bitwise ops (`andpd`, `orpd`, `xorpd`, `andnpd`) exist precisely for
this kind of bit-level manipulation of floats — and they are *heavily*
used in SIMD for masking.

## 5. Conversion

```nasm
cvtsi2sd xmm0, rax      ; signed integer -> double
cvtsi2ss xmm0, rax      ; signed integer -> float
cvtsd2si rax, xmm0      ; double -> integer, using current rounding mode
cvttsd2si rax, xmm0     ; double -> integer, TRUNCATING (extra 't')
cvtsd2ss xmm0, xmm1     ; double -> float
cvtss2sd xmm0, xmm1     ; float -> double
```

The doubled `t` in `cvttsd2si` means truncate. C's `(int)` cast truncates,
so this is what compilers emit — one of the most common instructions in
compiled float code.

## 6. Comparison

```nasm
ucomisd xmm0, xmm1      ; compare, set ZF/PF/CF directly; NaN-safe
comisd  xmm0, xmm1      ; same, but signals on quiet NaN
ja  .greater
jb  .less
je  .equal
jp  .unordered          ; PF set = at least one operand was NaN
```

`ucomisd` sets the *unsigned* flags (CF, ZF, PF), so you use the unsigned
jumps `ja`/`jb`/`jae`/`jbe` — never `jg`/`jl` — even though you're
comparing signed floating-point values. Using the signed jumps here is a
classic bug.

As with x87: NaN sets PF and makes both `ja` and `jb` fall through to
"below." Check `jp` first if NaN matters.

There is also a packed comparison family (`cmppd`, `cmpps`) that produces
a *mask* of all-ones or all-zeros per lane rather than setting flags —
that's the SIMD way, and the SIMD course starts there.

## 7. Program

`sse.asm`:

```nasm
default rel
global main
extern printf

section .data
    align 16
    a           dq  3.0
    b           dq  4.0
    nums        dq  1.5, 2.5, 3.5, 4.5, 5.5
    count       equ 5
    signmask    dq  0x8000000000000000, 0
    fmt         db  "sum=%f mean=%f hyp=%f neg=%f trunc=%ld", 10, 0

section .text

; ============================================================
; sum_doubles(rdi = ptr, rsi = count) -> xmm0 = sum
; ============================================================
sum_doubles:
    xorpd xmm0, xmm0            ; xmm0 = 0.0 (xor with self)
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .done
    addsd xmm0, [rdi + rcx*8]
    inc rcx
    jmp .loop
.done:
    ret

main:
    push rbp
    mov rbp, rsp
    sub rsp, 32                  ; scratch, keeps alignment

    ; ---- sum the array ----
    lea rdi, [nums]
    mov rsi, count
    call sum_doubles              ; xmm0 = 17.5
    movsd [rsp], xmm0              ; save it — xmm0 is caller-saved!

    ; ---- mean = sum / count ----
    mov rax, count
    cvtsi2sd xmm1, rax              ; integer 5 -> 5.0
    movsd xmm2, [rsp]
    divsd xmm2, xmm1                 ; 17.5 / 5.0 = 3.5
    movsd [rsp + 8], xmm2

    ; ---- hypotenuse of 3,4 using SSE instead of x87 ----
    movsd xmm3, [a]
    mulsd xmm3, xmm3                  ; 9.0
    movsd xmm4, [b]
    mulsd xmm4, xmm4                   ; 16.0
    addsd xmm3, xmm4                    ; 25.0
    sqrtsd xmm3, xmm3                    ; 5.0
    movsd [rsp + 16], xmm3

    ; ---- negate it via sign-bit xor ----
    movsd xmm5, xmm3
    xorpd xmm5, [signmask]                ; -5.0
    movsd [rsp + 24], xmm5

    ; ---- truncate the mean to an integer ----
    cvttsd2si rsi, xmm2                    ; 3.5 -> 3 (not 4!)

    ; ---- print: 4 doubles + 1 integer ----
    lea rdi, [fmt]
    movsd xmm0, [rsp]
    movsd xmm1, [rsp + 8]
    movsd xmm2, [rsp + 16]
    movsd xmm3, [rsp + 24]
    mov rdx, rsi                            ; the integer arg
    mov rsi, 0                               ; placeholder, unused
    mov eax, 4                                ; FOUR vector registers used
    call printf

    xor eax, eax
    leave
    ret
```

Wait — that `printf` call has an argument-ordering problem worth working
out yourself. The integer arg needs to be in the correct *integer*
sequence position (`rsi` is integer arg 2), and it collides with the
placeholder. **Fixing this is exercise 11.7.** Build it, see what prints,
and work out why.

```sh
nasm -f elf64 sse.asm -o sse.o -l sse.lst
gcc sse.o -o sse
./sse
```

## 8. GDB: XMM registers

```gdb
info registers xmm          # all 16, in every interpretation at once
p $xmm0                     # one register, all interpretations
```

GDB prints XMM registers as a struct showing the same bits interpreted
as `v4_float`, `v2_double`, `v16_int8`, `v8_int16`, `v4_int32`,
`v2_int64`, and `uint128`. Overwhelming at first, but it's exactly the
right mental model: **one register, many readings**.

To cut through the noise, index the field you want:

```gdb
p $xmm0.v2_double           # as two doubles -> {17.5, 0}
p $xmm0.v2_double[0]        # just the scalar value
p $xmm0.v4_float            # as four floats
p/x $xmm0.v2_int64          # raw bits in hex
```

`p $xmm0.v2_double[0]` is the command you'll use constantly in scalar
float debugging. Learn it now.

### Examining float memory

```gdb
x/1fg &a                    # one giant (8-byte) as a float -> 3
x/5fg &nums                 # five doubles
x/2fw &somefloat            # 4-byte floats use 'w'
x/2xg &nums                 # the same bytes as raw hex
```

Comparing `x/1fg` and `x/1xg` on the same address shows you the IEEE 754
encoding next to the value it represents — worth doing once.

### Setting XMM values mid-run

```gdb
p $xmm0.v2_double[0] = 99.5
set $xmm1 = $xmm0
```

### Session

```gdb
break sum_doubles
run
display $xmm0.v2_double[0]
si
```

Step through the accumulation loop and watch the running sum grow:
1.5, 4.0, 7.5, 12.0, 17.5. Then:

```gdb
finish
p $xmm0.v2_double[0]        # 17.5, the return value
```

### Checking the caller-saved rule

Break right before `call sum_doubles`, note some XMM register's value,
`finish`, and check it again. It's gone. That's why the program stores
results to the stack immediately.

## Exercises

**11.1 — Encoding archaeology.** Put `dq 1.0` in `.data`. Use `x/1xg` to
read its raw bits and `x/1fg` to read its value. Decompose the hex by
hand into sign / 11-bit exponent / 52-bit mantissa and confirm it encodes
1.0.

**11.2 — Truncation vs rounding.** Convert 2.5, -2.5, 2.7, and -2.7 to
integers using both `cvtsd2si` and `cvttsd2si`. Tabulate all eight
results and explain each.

**11.3 — Contract: `f_max`.**

> `double f_max(const double *arr, long n)`
> Returns the largest element. Use `maxsd`, no branches inside the loop.
> Handle n=0 by returning 0.0.

**11.4 — NaN trap.** Compare a NaN against 1.0 with `ucomisd` and check
`$eflags` for PF. Then write a correct three-way comparison that reports
less / equal / greater / unordered.

**11.5 — Signed-jump bug.** Compare -1.0 and 1.0 with `ucomisd` and use
`jg` instead of `ja`. Observe the wrong answer, then explain which flags
`ucomisd` actually sets and why the unsigned jumps are correct.

**11.6 — Contract: `dot_product`.**

> `double dot_product(const double *a, const double *b, long n)`
> Scalar implementation.

Keep this code. In the SIMD course you will rewrite it with `mulpd` and
`addpd` processing two elements per iteration, then benchmark against this
version. This is your baseline.

**11.7 — Fix the printf call.** The program above passes its integer
argument incorrectly. Work out the correct register assignment (remember:
integer and float arguments are counted in *separate* sequences), fix it,
and verify the output. Use `x/s $rdi` and `p $al` at the breakpoint to
diagnose.

**11.8 — Peek at packed.** Load `nums` into an XMM register with
`movupd xmm0, [nums]` — that's *two* doubles at once. Run
`p $xmm0.v2_double` and see both. Then try `addpd xmm0, xmm0` and print
again. You just did SIMD. Note that you changed exactly one letter from
the scalar version.
