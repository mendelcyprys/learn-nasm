# Lesson 07 — Addressing Modes, Arrays, and String Instructions

**Goal:** master how x86 computes memory addresses, and learn watchpoints
— the debugging technique for "something is changing this and I don't know
what."

## 1. The general addressing form

Every memory operand on x86-64 has this shape:

```
[ base + index*scale + displacement ]
```

- **base** — any general-purpose register
- **index** — any GPR except `rsp`
- **scale** — 1, 2, 4, or 8 (nothing else)
- **displacement** — a signed 32-bit constant

All parts are optional. Examples:

```nasm
mov rax, [rbx]                    ; base only
mov rax, [rbx + 8]                ; base + displacement
mov rax, [arr + rcx*8]            ; displacement + index*scale
mov rax, [rbx + rcx*4 + 12]       ; the full form
mov rax, [rbx + rcx]              ; scale defaults to 1
```

The scale values 1/2/4/8 are exactly the sizes of byte/word/dword/qword —
that's not a coincidence. `[arr + rcx*8]` indexes an array of 64-bit
values with `rcx` as the element index, which is why array access is one
instruction on x86 rather than a multiply plus an add.

**The displacement is 32-bit signed**, even in 64-bit mode. You cannot
put a full 64-bit constant address in a memory operand. This is why
position-independent code uses RIP-relative addressing.

## 2. RIP-relative addressing

```nasm
mov rax, [rel msg]        ; address computed relative to the instruction pointer
lea rsi, [rel buffer]
```

`rel` tells NASM to encode the operand as an offset from `rip` rather
than an absolute address. This produces smaller code, and it's *required*
for position-independent executables and shared libraries (where the load
address isn't known until runtime).

You can make it the default for a whole file:

```nasm
default rel               ; put this near the top
```

Then plain `[msg]` becomes RIP-relative automatically, and you write
`[abs msg]` on the rare occasion you want absolute. Most modern NASM code
does this. Get in the habit.

## 3. `lea` — address arithmetic without memory access

```nasm
lea rax, [rbx + 8]              ; rax = rbx + 8    (no memory touched!)
lea rax, [rbx + rcx*4 + 100]    ; rax = rbx + rcx*4 + 100
```

`lea` (Load Effective Address) computes the address the brackets describe
and puts *the address itself* in the destination. It never reads memory.

Compare:

```nasm
mov rax, [rbx + 8]      ; rax = the VALUE at rbx+8   (memory read)
lea rax, [rbx + 8]      ; rax = the ADDRESS rbx+8    (no read)
```

Because the address unit can do `base + index*scale + disp` in one shot,
`lea` doubles as a fast three-operand arithmetic instruction that doesn't
touch flags:

```nasm
lea rax, [rbx + rbx*2]      ; rax = rbx * 3
lea rax, [rbx + rbx*4]      ; rax = rbx * 5
lea rax, [rax + rax]        ; rax = rax * 2
```

Compilers use it constantly for exactly this. When you see `lea` in
disassembly with no obvious pointer involved, it's arithmetic.

## 4. Arrays and strides

```nasm
section .data
    ints    dq  10, 20, 30, 40, 50        ; 8 bytes each
    bytes   db  1, 2, 3, 4, 5              ; 1 byte each

section .text
    mov rax, [ints + 2*8]         ; third element, constant index
    mov rcx, 2
    mov rax, [ints + rcx*8]        ; third element, variable index
    movzx rax, byte [bytes + rcx]   ; scale 1 for a byte array
```

For a 2D array of `rows × cols` quadwords, stored row-major:

```nasm
; element [i][j] is at base + (i*cols + j)*8
mov rax, rdi                ; i
imul rax, cols               ; i * cols
add rax, rsi                  ; + j
mov rax, [arr + rax*8]         ; load it
```

## 5. String instructions

x86 has dedicated instructions that operate on memory pointed to by `rsi`
(source) and `rdi` (destination), auto-incrementing both:

| Instruction | Action |
|---|---|
| `movsb`/`movsw`/`movsd`/`movsq` | `[rdi] = [rsi]`, advance both |
| `stosb`/`stosw`/`stosd`/`stosq` | `[rdi] = al/ax/eax/rax`, advance `rdi` |
| `lodsb`/`lodsw`/`lodsd`/`lodsq` | `al/ax/eax/rax = [rsi]`, advance `rsi` |
| `scasb`/… | compare `al`/… with `[rdi]`, set flags, advance |
| `cmpsb`/… | compare `[rsi]` with `[rdi]`, set flags, advance both |

Direction is controlled by the **direction flag (DF)**: `cld` = forward
(increment), `std` = backward (decrement). **The ABI requires DF to be
clear on function entry and exit**, so if you ever use `std`, you must
`cld` before returning.

The `rep` prefixes repeat the operation `rcx` times:

```nasm
; memcpy: copy rcx bytes from rsi to rdi
cld
mov rcx, 100
rep movsb

; memset: fill rcx bytes at rdi with al
cld
mov al, 0
mov rcx, 256
rep stosb

; find a byte: scan up to rcx bytes at rdi for al
cld
mov al, 'x'
mov rcx, 100
repne scasb          ; repeat while NOT equal
; on exit: ZF set if found; rdi points one past the match
```

On modern CPUs `rep movsb` and `rep stosb` are specially optimized in
microcode (the "ERMSB" feature) and are genuinely fast for large copies —
often beating a hand-written loop. The other string instructions are
mostly obsolete; a SIMD loop beats them badly, which is a preview of why
the SIMD course exists.

## 6. Program: array processing

`arrays.asm`:

```nasm
default rel
global _start

section .data
    src     dq  5, 3, 9, 1, 7, 8, 2, 6
    n       equ 8
    msg     db  "done", 10
    msglen  equ $ - msg

section .bss
    dst     resq 8
    maxval  resq 1

section .text
_start:
    ; ---- copy src to dst using rep movsq ----
    cld
    lea rsi, [src]
    lea rdi, [dst]
    mov rcx, n
    rep movsq

    ; ---- find the maximum ----
    lea rbx, [src]
    mov rax, [rbx]              ; assume first is max
    mov rcx, 1
.maxloop:
    cmp rcx, n
    jge .maxdone
    mov rdx, [rbx + rcx*8]       ; indexed load
    cmp rdx, rax
    cmovg rax, rdx                ; branchless max
    inc rcx
    jmp .maxloop
.maxdone:
    mov [maxval], rax

    ; ---- double every element of dst in place ----
    lea rbx, [dst]
    xor rcx, rcx
.dblloop:
    cmp rcx, n
    jge .dbldone
    mov rax, [rbx + rcx*8]
    lea rax, [rax + rax]           ; lea as arithmetic: rax *= 2
    mov [rbx + rcx*8], rax
    inc rcx
    jmp .dblloop
.dbldone:

    mov rax, 1
    mov rdi, 1
    lea rsi, [msg]
    mov rdx, msglen
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

```sh
nasm -f elf64 arrays.asm -o arrays.o -l arrays.lst
ld arrays.o -o arrays
gdb ./arrays
```

## 7. GDB: watchpoints

A watchpoint stops the program when a **value changes**, rather than when
execution reaches a place. It's the right tool when you know *what* is
wrong but not *where*.

```gdb
watch  *(long*)&maxval      # break when this memory is WRITTEN
rwatch *(long*)&maxval      # break when it is READ
awatch *(long*)&maxval      # break on either
```

You must tell GDB the type and size — hence the `*(long*)` cast. For
raw NASM symbols with no type info, that cast is mandatory.

Watching a register works too:

```gdb
watch $rax                  # stops whenever rax changes
```

Be warned: register watchpoints are usually implemented by single-stepping
the whole program, which is *slow*. Memory watchpoints use hardware debug
registers and run at full speed — but you only get **four** of them, since
x86 has four hardware debug registers. Exceeding that silently drops GDB
into slow software watchpoints.

### Practical session

```gdb
break _start
run
watch *(long*)&maxval
continue
```

The program runs at full speed and stops the instant `maxval` is written,
printing the old and new values. `bt` then tells you where you are.

Managing them:

```gdb
info watchpoints
delete 2
```

### Watching the copy happen

```gdb
break _start
run
x/8gx &dst                  # all zeros — .bss is zero-filled
break *<address after rep movsq>
continue
x/8gx &dst                  # now a copy of src
```

Stepping *through* `rep movsq` with `si` shows you one iteration at a
time and `rcx` counting down — worth doing once to see that `rep` really
is a loop, not a single magic operation.

## Exercises

**7.1 — `lea` vs `mov`.** Write a fragment that does both
`mov rax, [arr + 8]` and `lea rbx, [arr + 8]`. In GDB, print both and
explain in one sentence why they differ. Then use `x/gx $rbx` to show
they're consistent.

**7.2 — `lea` as arithmetic.** Using only `lea` instructions (no `imul`,
no `add`), compute `rax = rbx * 9`. Then `rax = rbx * 10`. Check your
answers in GDB.

**7.3 — Watchpoint hunt.** Deliberately introduce a bug into
`arrays.asm`: make the doubling loop write one element past the end of
`dst`. Set a watchpoint on the 8 bytes just past `dst` and let GDB find
the offending instruction for you.

**7.4 — Contract: `memset`.**

> `memset(rdi = dest, rsi = byte value, rdx = count) -> rax = dest`

Write it twice: once as an explicit loop, once with `rep stosb`. Compare
instruction counts in `objdump -d`.

**7.5 — Contract: reverse in place.**

> `reverse(rdi = array of quadwords, rsi = count) -> nothing`
> Reverses the array in place. All callee-saved registers preserved.

Use `x/8gx` before and after to verify.

**7.6 — Contract: 2D indexing.**

> `get2d(rdi = base, rsi = row, rdx = col, rcx = ncols) -> rax = element`
> Elements are quadwords, row-major.

Build a 3×4 array in `.data` and test several indices. Verify each by
hand with `x/12gx`.

**7.7 — `repne scasb` string length.** Implement `strlen` using
`repne scasb` rather than a loop. Careful: `rcx` must be set to a maximum
count first, and the result needs adjusting because `rdi` ends one past
the match. Verify against your Lesson 06 version.

**7.8 — Direction flag.** Write a copy that runs *backwards* using `std`
and `rep movsq`. Confirm it works, then confirm you remembered to `cld`
afterwards. What would break if you didn't?
