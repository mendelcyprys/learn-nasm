# Lesson 05 — The Stack

**Goal:** understand the stack as a concrete region of memory with a
pointer, not an abstraction. Everything about function calls in Lesson 06
depends on this.

## 1. What the stack actually is

It's just memory. The kernel maps a region for it when your process
starts and puts its top address in `rsp`. There is no special hardware
"stack unit" — only two facts:

1. **It grows downward.** Pushing *decreases* `rsp`. This is a convention
   baked into the instruction encodings, and it's the opposite of what
   most people guess.
2. **`rsp` always points at the most recently pushed item**, not at free
   space above it.

```nasm
push rax        ; rsp -= 8, then [rsp] = rax
pop  rax        ; rax = [rsp], then rsp += 8
```

On x86-64, pushes and pops are **always 8 bytes**. You cannot push a
32-bit register. `push rax` and `pop rax` are the only widths you'll use.

## 2. Reserving space without pushing

```nasm
sub rsp, 32        ; reserve 32 bytes of scratch space
mov qword [rsp], 5        ; use it like an array
mov qword [rsp + 8], 6
mov qword [rsp + 16], 7
add rsp, 32        ; release it — MUST match, or you corrupt the stack
```

This is how local variables work. The compiler does exactly this for
every C function with locals: subtract at entry, add back at exit.

**Every `sub rsp` needs a matching `add rsp`, and every `push` a matching
`pop`, on every path out of the code.** An unbalanced stack is the single
most common way to make an assembly program crash mysteriously — and the
crash usually happens somewhere far from the actual mistake.

## 3. `rbp` and stack frames

Traditionally `rbp` (base pointer) is used to anchor a fixed reference
point in the frame:

```nasm
my_function:
    push rbp            ; save the caller's rbp
    mov rbp, rsp        ; rbp now marks this frame's base
    sub rsp, 32         ; allocate 32 bytes = four quadwords of locals

    mov qword [rbp - 8],  42    ; local 1 — fixed offset from rbp
    mov qword [rbp - 16], 99    ; local 2
    mov qword [rbp - 24], 7     ; local 3
    mov qword [rbp - 32], 1234  ; local 4 — the last one that fits

    mov rsp, rbp        ; deallocate everything at once
    pop rbp             ; restore caller's rbp
    ret
```

Count the slots: 32 bytes divided by 8 is four, at offsets −8, −16, −24,
and −32. Note that `[rbp - 32]` is the *lowest* valid local — it's exactly
where `rsp` is pointing after the `sub`. Writing to `[rbp - 40]` would be
below `rsp`, outside your frame, and liable to be clobbered by anything
you call.

Here's the memory picture, addresses increasing upward:

```
    higher addresses
    ┌──────────────────────┐
    │ caller's stack ...   │
    ├──────────────────────┤
    │ return address       │  ← pushed by `call`
    ├──────────────────────┤
    │ saved rbp            │  ← rbp points HERE
    ├──────────────────────┤
    │ local 1              │  [rbp - 8]
    ├──────────────────────┤
    │ local 2              │  [rbp - 16]
    ├──────────────────────┤
    │ local 3              │  [rbp - 24]
    ├──────────────────────┤
    │ local 4              │  [rbp - 32]  ← rsp points HERE
    └──────────────────────┘
    lower addresses
```

Two offsets worth memorizing from this diagram: `[rbp + 8]` is the return
address, and `[rbp + 16]` is where a seventh stack-passed argument would
land (Lesson 06 exercise 6.4).

The point of `rbp` is that offsets from it stay constant even if `rsp`
moves during the function (from pushes, or a variable-size allocation).
Offsets from `rsp` would shift.

Modern compilers usually *omit* the frame pointer (`-fomit-frame-pointer`,
on by default at `-O1` and above) and address locals from `rsp` directly,
freeing `rbp` as a general register. But frames make debugging much
easier, and GDB's `backtrace` relies on either frame pointers or DWARF
unwind info — which your hand-written NASM has neither of. Use `rbp`
frames while learning; you'll see the payoff in GDB immediately.

There are also two dedicated instructions:

```nasm
enter 32, 0         ; equivalent to: push rbp / mov rbp, rsp / sub rsp, 32
leave               ; equivalent to: mov rsp, rbp / pop rbp
```

`leave` is genuinely used and worth knowing. `enter` is slow and rarely
used — write the three instructions out instead.

## 4. `call` and `ret` demystified

```nasm
call target
```

does exactly two things:
1. Pushes the address of the *next* instruction onto the stack.
2. Jumps to `target`.

```nasm
ret
```

does exactly one thing: pops an 8-byte value off the stack into `rip`.

That's the whole mechanism. `ret` doesn't "know" where it came from — it
blindly jumps to whatever 8 bytes are on top of the stack. If you push
something and forget to pop it before returning, `ret` will happily jump
to your data and the process will die. Understanding this makes the class
of bugs obvious rather than mysterious.

## 5. The red zone

The System V ABI reserves the **128 bytes below `rsp`** as the "red
zone" — a leaf function (one that calls nothing else) may use it as
scratch space without adjusting `rsp` at all:

```nasm
leaf_function:
    mov [rsp - 8], rax       ; legal, no sub rsp needed
    mov [rsp - 16], rbx
    ret
```

The kernel guarantees signal handlers won't clobber it. **But** anything
you `call` will immediately overwrite it, and — importantly — the red zone
does *not* exist in kernel code or in signal handlers themselves. Use it
only in genuine leaf functions.

## 6. Program: stack in action

`stack.asm`:

```nasm
global _start

section .text
_start:
    mov rbp, rsp             ; remember the original stack top

    mov rax, 0x1111111111111111
    mov rbx, 0x2222222222222222
    mov rcx, 0x3333333333333333

    push rax
    push rbx
    push rcx                  ; three pushes: rsp is now 24 lower

    ; scratch space below the pushes
    sub rsp, 16
    mov qword [rsp], 0xAAAA
    mov qword [rsp + 8], 0xBBBB
    add rsp, 16                ; release it

    pop rdx                     ; rdx = 0x3333... (LIFO: last in, first out)
    pop rsi
    pop rdi

    ; rsp should now equal rbp again — verify this in GDB!

    call helper
    ; execution resumes here after ret

    mov rax, 60
    xor rdi, rdi
    syscall

helper:
    push rbp
    mov rbp, rsp
    sub rsp, 32

    mov qword [rbp - 8], 111
    mov qword [rbp - 16], 222
    mov qword [rbp - 24], 333
    mov qword [rbp - 32], 444      ; lowest slot — equals [rsp] right now

    mov rsp, rbp
    pop rbp
    ret
```

```sh
nasm -f elf64 stack.asm -o stack.o -l stack.lst
ld stack.o -o stack
gdb ./stack
```

## 7. GDB: seeing the stack

### Dumping it

```gdb
x/8gx $rsp          # 8 giant hex values starting at the stack pointer
```

This is the command you'll type hundreds of times. Read it as "show me
the top 8 slots of the stack."

Useful variations:

```gdb
x/16gx $rsp         # deeper
x/4gx $rbp          # around the frame base
x/8dg $rsp          # as signed decimals instead of hex
p $rsp              # just the pointer value
p $rbp - $rsp       # how big is the current frame?
```

### The disciplined way to work

```gdb
break _start
run
display/8gx $rsp
display $rsp
```

Now every `stepi` shows you the stack and pointer together. Step through
each `push` and watch `rsp` drop by 8 while a new value appears at the
top. This is the moment the stack stops being abstract.

### Backtraces

```gdb
bt                  # backtrace: the chain of calls that got you here
frame 1             # switch to the caller's frame
up / down           # move through the frame chain
info frame          # detailed layout of the current frame
```

Break inside `helper` and run `bt`. Because you set up a proper `rbp`
frame, GDB can walk the chain and show you `_start` below it. Then try
commenting out the `push rbp` / `mov rbp, rsp` lines, rebuild, and run
`bt` again — the backtrace degrades or goes wrong. That's a concrete
demonstration of what frame pointers buy you.

### Watching `call` and `ret` directly

Break on the `call helper` line and do:

```gdb
x/2gx $rsp          # note what's on top
stepi               # execute the call
x/2gx $rsp          # a new value appeared: the return address
p/x $rip            # you're now inside helper
```

Compare the value that appeared on the stack against the address of the
instruction *after* `call` in `disassemble` output. They're the same
number. That's the entire mechanism, visible.

## Exercises

**5.1 — Balance check.** In `stack.asm`, break right before `call helper`
and confirm `$rsp == $rbp`. Then deliberately delete one `pop`, rebuild,
and watch what goes wrong. Where does it actually crash, and is that where
the mistake was?

**5.2 — LIFO by hand.** Push five distinct recognizable values
(`0x1111...` through `0x5555...`). Before popping, predict the order
they'll come back in. Verify with `x/5gx $rsp`.

**5.3 — Find the return address.** Break at the first instruction of
`helper`. Using only `x/gx $rsp`, determine the return address. Confirm it
matches `disassemble _start` output.

**5.4 — Corrupt it on purpose.** Break inside `helper` and use
`set {long}$rsp = 0x4141414141414141` to overwrite the return address.
`continue` and observe the crash. Then read the fault address GDB reports.
This is, mechanically, exactly what a stack-smashing exploit does — worth
seeing once so the concept isn't mysterious.

**5.5 — Contract: swap via stack.**

> On entry: `rdi` and `rsi` hold two values.
> On exit: their values are exchanged. No other register modified, and
> `rsp` must be exactly what it was on entry.

**5.6 — Contract: sum an array using stack scratch.**

> On entry: `rdi` = address of an array of 8 quadwords.
> On exit: `rax` = their sum. `rsp` unchanged. `rbx` unchanged.

Use `sub rsp, N` to allocate scratch space, even if you don't strictly
need it, and verify with `p $rsp` before and after that you released it
correctly.

**5.7 — Nested calls.** Write three functions that call each other
(`a` calls `b` calls `c`), each with a proper `rbp` frame. Break in `c`
and run `bt`. Then use `frame 1` and `frame 2` to inspect each caller's
frame with `x/4gx $rbp`.
