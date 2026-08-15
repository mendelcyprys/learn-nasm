# Lesson 04 — Control Flow

**Goal:** turn flags into decisions. Every `if`, `while`, `for`, and
`switch` you have ever written compiles down to what's in this lesson.

## 1. Unconditional jumps

```nasm
jmp label           ; go there, unconditionally
jmp rax             ; indirect: jump to the address in a register
jmp [table + rbx*8] ; indirect through memory — this is how switch works
```

## 2. Conditional jumps

Each one tests flags set by the *previous* flag-setting instruction. The
critical split is **signed vs unsigned** — they read different flags, and
picking the wrong family is a classic silent bug.

### Unsigned comparisons (think "above/below")

| Instruction | Jumps if | Flags |
|---|---|---|
| `ja` / `jnbe` | above (>) | CF=0 and ZF=0 |
| `jae` / `jnb` | above or equal (>=) | CF=0 |
| `jb` / `jnae` | below (<) | CF=1 |
| `jbe` / `jna` | below or equal (<=) | CF=1 or ZF=1 |

### Signed comparisons (think "greater/less")

| Instruction | Jumps if | Flags |
|---|---|---|
| `jg` / `jnle` | greater (>) | ZF=0 and SF=OF |
| `jge` / `jnl` | greater or equal (>=) | SF=OF |
| `jl` / `jnge` | less (<) | SF≠OF |
| `jle` / `jng` | less or equal (<=) | ZF=1 or SF≠OF |

### Sign-agnostic

| Instruction | Jumps if |
|---|---|
| `je` / `jz` | equal / zero (ZF=1) |
| `jne` / `jnz` | not equal / not zero (ZF=0) |
| `js` | sign set (result negative) |
| `jns` | sign clear |
| `jc` / `jnc` | carry set / clear |
| `jo` / `jno` | overflow set / clear |

**Reading direction matters.** `cmp rax, rbx` followed by `jg` means "jump
if rax > rbx". The first operand is the left-hand side. Say it out loud
in that order and you won't invert it.

## 3. The three loop shapes

### While (test at top)

```nasm
    mov rcx, 0
.while:
    cmp rcx, 10
    jge .done              ; exit test first
    ; ... body ...
    inc rcx
    jmp .while
.done:
```

### Do-while (test at bottom) — usually faster, one jump per iteration

```nasm
    mov rcx, 0
.do:
    ; ... body ...
    inc rcx
    cmp rcx, 10
    jl .do
```

### Counted down to zero — the idiomatic x86 form

```nasm
    mov rcx, 10
.loop:
    ; ... body ...
    dec rcx
    jnz .loop              ; dec sets ZF; no cmp needed at all
```

That last form is why counting *down* is so common in hand-written
assembly: `dec` sets ZF for free, so you skip the `cmp` entirely.

There is also a dedicated `loop rcx_label` instruction, which decrements
`rcx` and jumps if non-zero. **Don't use it** — it's slower than
`dec`/`jnz` on every modern CPU, kept only for compatibility. Recognize
it in old code; don't write it.

## 4. Conditional moves — branchless code

```nasm
cmp rax, rbx
cmovg rax, rbx          ; if rax > rbx, rax = rbx    (no branch taken)
```

`cmov` has the same suffix set as `j` (`cmovg`, `cmovle`, `cmovz`...). It
avoids a branch entirely, which matters because a mispredicted branch
costs ~15-20 cycles on modern CPUs. For unpredictable conditions, `cmov`
usually wins; for predictable ones, a branch usually wins.

Caveat: `cmov` always *reads* both operands, so it cannot be used to guard
a possibly-invalid memory load.

This branchless mindset is direct preparation for SIMD, where there are
no branches per-element at all — you compute both sides and select.

## 5. `set` instructions — condition to 0 or 1

```nasm
cmp rax, rbx
setg al             ; al = 1 if rax > rbx, else 0
movzx rax, al       ; widen to the full register
```

Useful for turning comparisons into values rather than jumps.

## 6. Program: FizzBuzz-ish counter

`loops.asm` — counts 1..20, prints `*` for multiples of 3, `#` for
multiples of 5, `.` otherwise:

```nasm
global _start

section .bss
    ch      resb 2

section .text
_start:
    mov r12, 1                    ; loop counter (r12 is callee-saved; see L06)
.loop:
    cmp r12, 20
    jg  .done

    ; --- is it divisible by 3? ---
    mov rax, r12
    xor rdx, rdx
    mov rcx, 3
    div rcx
    test rdx, rdx                 ; remainder zero?
    jz  .star

    ; --- divisible by 5? ---
    mov rax, r12
    xor rdx, rdx
    mov rcx, 5
    div rcx
    test rdx, rdx
    jz  .hash

    mov byte [ch], '.'
    jmp .print
.star:
    mov byte [ch], '*'
    jmp .print
.hash:
    mov byte [ch], '#'

.print:
    mov byte [ch + 1], 10          ; newline
    mov rax, 1
    mov rdi, 1
    mov rsi, ch
    mov rdx, 2
    syscall

    inc r12
    jmp .loop

.done:
    mov rax, 60
    xor rdi, rdi
    syscall
```

```sh
nasm -f elf64 loops.asm -o loops.o -l loops.lst
ld loops.o -o loops
./loops
```

## 7. GDB: breakpoint mastery

Stepping through 20 iterations one instruction at a time is unbearable.
This is what breakpoint control is for.

### Setting them

```gdb
break _start                # by symbol
break *0x401015             # by exact address (needed for local labels!)
break *_start+42            # by offset from a symbol
```

Since NASM local labels (`.loop`) never reach the symbol table, address
breakpoints are how you stop inside a function. Get addresses from
`disassemble` or `objdump -d`.

### Conditional breakpoints — the big one

```gdb
break *0x401020 if $r12 == 15
```

The program runs at full speed and stops only on iteration 15. This
single feature is the difference between debugging a loop and suffering
through one.

You can add a condition to an existing breakpoint too:

```gdb
condition 2 $r12 > 10       # breakpoint 2 now only fires when r12 > 10
condition 2                 # remove the condition
```

### Ignore counts

```gdb
ignore 2 9                  # skip the next 9 hits of breakpoint 2
```

Handy when you know the bug appears "somewhere after a while."

### Temporary breakpoints

```gdb
tbreak *0x401030            # fires once, then deletes itself
```

### Managing them

```gdb
info breakpoints            # list all, with hit counts
disable 2                   # turn off without deleting
enable 2
delete 2                    # remove one
delete                      # remove all
```

`info breakpoints` showing hit counts is quietly useful: it tells you how
many times a branch was actually taken.

### Stepping controls

```gdb
stepi        / si      # one instruction, entering calls
nexti        / ni      # one instruction, stepping OVER calls
until        / u       # run until a higher address — escapes a loop!
finish                 # run until the current function returns (Lesson 06)
```

`until` is the loop-escape command: at the bottom of a loop it runs to
completion rather than jumping back.

### Watching the loop

```gdb
break *0x401015              # address of .loop, from `disassemble`
run
display $r12
display $eflags
continue                     # each `continue` = one iteration
```

## Exercises

**4.1 — Signed/unsigned trap.** Set `rax = -1` and `rbx = 1`. Do
`cmp rax, rbx` and then check both `jg` and `ja`. One says "greater," one
says "not greater." Explain why, in terms of what `-1` looks like as an
unsigned 64-bit number.

**4.2 — Rewrite the loop three ways.** Take the counter loop from
`loops.asm` and rewrite it as (a) a while loop, (b) a do-while loop,
(c) a count-down-to-zero loop using `dec`/`jnz`. Compare the instruction
counts with `objdump -d`.

**4.3 — Conditional breakpoint practice.** Using `loops.asm`, set a
breakpoint that fires only when the character about to be printed is `#`.
(Hint: break after the `.hash` store, or use a condition on `[ch]`.)

**4.4 — Contract: max of three.**

> On entry: `rdi`, `rsi`, `rdx` hold three *signed* 64-bit integers.
> On exit: `rax` holds the largest. No other register modified.

Write it twice: once with branches, once entirely with `cmov`. Compare
the disassembly.

**4.5 — Contract: string length.**

> On entry: `rdi` = address of a NUL-terminated string.
> On exit: `rax` = its length, not counting the NUL.

Test it on a string in `.data`. Use a conditional breakpoint to stop when
`rax == 3` and confirm you're where you expect.

**4.6 — Jump table.** Implement a 4-way switch using
`jmp [table + rax*8]`, where `table` is a `dq` list of label addresses.
Step into the indirect jump in GDB with `si` and watch `rip` land
somewhere the disassembly didn't obviously predict.

**4.7 — Collatz.** Write a program that takes a starting value in `r12`
and counts how many Collatz steps (`n/2` if even, `3n+1` if odd) it takes
to reach 1, leaving the count in `r13`. Use a conditional breakpoint to
stop when `r13 == 10` and inspect the value of `r12` at that moment.
