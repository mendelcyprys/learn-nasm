# Lesson 06 — Functions and the System V AMD64 ABI

**Goal:** learn the calling convention properly. This is the most
practically important lesson in the course — it's what lets your assembly
call C, C call your assembly, and your own functions compose reliably.

The ABI is a *contract*, not a law. The CPU won't enforce it. But every
compiled function on your system obeys it, so the moment you interoperate
with anything, you obey it too.

## 1. Where arguments go

**Integer and pointer arguments**, in order:

| Arg | Register |
|---|---|
| 1st | `rdi` |
| 2nd | `rsi` |
| 3rd | `rdx` |
| 4th | `rcx` |
| 5th | `r8` |
| 6th | `r9` |

Beyond six: pushed onto the stack, **in right-to-left order**, so the 7th
argument ends up nearest the return address.

**Floating-point arguments** go in `xmm0`–`xmm7`, counted separately.
A function taking `(int a, double b, int c)` gets `a` in `rdi`,
`c` in `rsi`, and `b` in `xmm0` — the two sequences don't interleave.

**Return values:**

| Type | Register |
|---|---|
| Integer / pointer | `rax` |
| 128-bit integer | `rdx:rax` |
| Float / double | `xmm0` |
| Large struct | Caller allocates; hidden pointer in `rdi`, returned in `rax` |

Memorize the first table. Write it on something. You will use it
constantly.

## 2. Who saves what

This is the part people get wrong.

**Callee-saved** (a.k.a. *non-volatile*) — if your function uses these,
it must restore them before returning:

```
rbx  rbp  rsp  r12  r13  r14  r15
```

**Caller-saved** (a.k.a. *volatile*) — a called function may destroy
these freely; if you need a value across a `call`, save it yourself:

```
rax  rcx  rdx  rsi  rdi  r8  r9  r10  r11
all xmm registers
```

Two practical consequences:

1. **All XMM registers are caller-saved.** Every single one. This becomes
   a real cost in SIMD code where you're juggling many vector registers
   across calls — worth remembering for the SIMD course.
2. **The argument registers are all caller-saved.** So a function is free
   to clobber its own incoming arguments, and routinely does.

A useful mnemonic: the callee-saved set is `rbx`, `rbp`, `rsp`, and the
*high-numbered* registers `r12`–`r15`. Everything else is fair game.

## 3. Stack alignment — the rule that bites hardest

**At the point a `call` instruction executes, `rsp` must be 16-byte
aligned.**

Equivalently: on entry to a function (after `call` has pushed the 8-byte
return address), `rsp % 16 == 8`.

Why: SSE instructions like `movaps` fault on unaligned addresses, and
compiled code assumes the guarantee. Break the rule and your program
crashes inside `printf` with no obvious connection to what you did wrong.

The standard prologue keeps you aligned automatically:

```nasm
my_func:
    push rbp            ; rsp was ≡8 (mod 16) on entry; now ≡0
    mov rbp, rsp        ; aligned — safe to call from here
    sub rsp, 32         ; keep sub amounts a multiple of 16
    ; ...
    leave
    ret
```

If you push an *odd* number of registers, add a dummy `sub rsp, 8` to
re-align before calling out.

**Special note for `_start`:** when the kernel starts your process,
`rsp` is 16-byte aligned but there's *no return address on the stack* —
`_start` was jumped to, not called. So if you `call` a libc function
directly from `_start`, you must `sub rsp, 8` first. This trips up
everyone exactly once.

## 4. A well-behaved function

`funcs.asm`:

```nasm
global _start

section .data
    values  dq  10, 20, 30, 40, 50
    count   equ 5

section .text

; ============================================================
; sum_array(rdi = pointer, rsi = count) -> rax = sum
; Clobbers: rax, rcx (caller-saved — allowed)
; Preserves: everything else, including rbx and r12
; ============================================================
sum_array:
    push rbp
    mov rbp, rsp
    push rbx                  ; callee-saved: we use it, so we save it

    xor rax, rax               ; running sum
    xor rcx, rcx                ; index

.loop:
    cmp rcx, rsi
    jge .done
    mov rbx, [rdi + rcx*8]      ; load element
    add rax, rbx
    inc rcx
    jmp .loop

.done:
    pop rbx                      ; restore before returning
    leave
    ret

; ============================================================
; scale(rdi = value, rsi = factor) -> rax = value * factor
; A leaf function: calls nothing, so no frame needed at all.
; ============================================================
scale:
    mov rax, rdi
    imul rax, rsi
    ret

_start:
    ; --- call sum_array(values, 5) ---
    mov rdi, values
    mov rsi, count
    call sum_array               ; rax = 150

    ; --- call scale(rax, 2) ---
    mov rdi, rax
    mov rsi, 2
    call scale                    ; rax = 300

    mov rdi, rax                   ; exit with the result as the status
    mov rax, 60
    syscall
```

```sh
nasm -f elf64 funcs.asm -o funcs.o -l funcs.lst
ld funcs.o -o funcs
./funcs
echo $?          # should print 300 mod 256 = 44
```

(Exit statuses are truncated to 8 bits by the OS — hence 44, not 300.
Worth knowing.)

## 5. Documenting the contract

Get into this habit now. Above every function, write:

```nasm
; name(arg1 = what, arg2 = what) -> return
; Clobbers: <registers destroyed>
; Preserves: <callee-saved registers used and restored>
```

In assembly there is no type system and no compiler to check you. The
comment *is* the interface. Every exercise in this course from here on
specifies a contract, and you should write it above your code.

## 6. GDB: working with functions

### `finish` — run to the end of the current function

```gdb
break sum_array
run
finish
```

GDB runs until `sum_array` returns, then prints **"Value returned is …"**
along with `rax`. This is the fastest way to test a function: break on
it, `finish`, read the result. No stepping required.

### `until` — escape a loop without leaving the function

Inside `.loop`, `until` runs to the first instruction at a higher address,
i.e. past the loop. Compare with `finish`, which exits the whole function.

### Navigating frames

```gdb
bt              # the call chain
frame 1         # select the caller
info frame      # frame layout: saved registers, return address location
info args       # (needs debug info — will be empty for raw NASM)
up / down       # move one frame
```

With `frame 1` selected, `p $rsp` and `x/8gx $rbp` show you the *caller's*
view, which is how you check "did my function preserve what it promised?"

### Verifying the ABI contract yourself

This is the workflow worth internalizing:

```gdb
break sum_array
run

# snapshot the callee-saved registers on entry
p/x $rbx
p/x $r12
p/x $rsp

finish

# compare after return — these three must be identical
p/x $rbx
p/x $r12
p/x $rsp
```

If any differ, your function violates the ABI. This check catches real
bugs, and it's exactly what you'd do to debug a crash inside a library
you called.

### Calling your function directly from GDB

```gdb
p sum_array
p $rax
```

GDB can even invoke functions in the running process, though with raw
NASM and no type info it needs casts and is fiddly. Mostly you'll use
`set $rdi = …` before a breakpointed call instead — simpler and more
predictable.

## Exercises

**6.1 — Verify the contract.** Take `sum_array` and deliberately remove
the `push rbx` / `pop rbx`. Set `rbx` to a recognizable value before the
call, then use the entry/exit comparison workflow above to catch the
violation. Note that the program still *runs correctly* — the bug is
invisible until something else depends on `rbx`.

**6.2 — Contract: `strlen`.**

> `strlen(rdi = ptr to NUL-terminated string) -> rax = length`
> Clobbers: `rax`, `rcx` only. Must preserve all callee-saved registers.

Test with `finish` and check the returned value.

**6.3 — Contract: `memcpy`.**

> `memcpy(rdi = dest, rsi = src, rdx = count) -> rax = dest`
> Byte-by-byte is fine. Must preserve callee-saved registers.

Verify with `x/16xb` on both buffers before and after.

**6.4 — Seven arguments.** Write a function taking seven integer
arguments that returns their sum. The seventh must be passed on the stack.
Work out where it lands relative to `rbp` in the callee (hint: `[rbp+16]`
if you use the standard prologue — draw the stack to confirm) and verify
with `x/4gx $rbp` inside the function.

**6.5 — Alignment failure, deliberately.** Write a function that pushes
an odd number of registers and then calls another function. Before and
after the `call`, check `p $rsp % 16` in GDB. Then add the `sub rsp, 8`
fix and confirm the value changes. (You won't crash yet — you will in
Lesson 09 when real libc code assumes the guarantee.)

**6.6 — Contract: recursive factorial.**

> `fact(rdi = n) -> rax = n!`
> Must be genuinely recursive (calls itself). n ≤ 20.

Break inside it, let it recurse a few levels, and run `bt` to see the
stack of frames. Then use `frame 3` to inspect the value of `rdi` at that
depth.

**6.7 — Contract: `qsort`-style comparator.**

> `apply(rdi = array ptr, rsi = count, rdx = function ptr) -> rax = accumulated result`
> `apply` should call the function in `rdx` once per element, passing the
> element in `rdi`, and sum the returned values.

This forces you to handle caller-saved registers correctly: `rdi`, `rsi`,
and `rdx` are all destroyed by the call you're making inside your loop.
Where do you keep the loop state? (Answer: callee-saved registers — which
is exactly what they're for.)
