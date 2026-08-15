# Lesson 01 — The Toolchain and Your First Binary

**Goal:** understand what each tool in the pipeline actually does, build a
working program, and take your first steps in GDB.

## 1. The registers you'll meet immediately

x86-64 has 16 general-purpose 64-bit registers:

```
rax rbx rcx rdx rsi rdi rbp rsp r8 r9 r10 r11 r12 r13 r14 r15
```

The first eight have names inherited from the 16-bit 8086 era (`ax` =
accumulator, `cx` = counter, and so on). Those historical meanings are
mostly dead — with a few exceptions we'll flag as they come up (`rcx` for
`loop`, `rdx:rax` for `div`) — so treat them as general-purpose.

Each register can be accessed at four widths. Using `rax` as the example:

| Width | Name | Bits |
|---|---|---|
| 64-bit | `rax` | 63:0 |
| 32-bit | `eax` | 31:0 |
| 16-bit | `ax`  | 15:0 |
| 8-bit  | `al`  | 7:0 |

`r8` through `r15` use a suffix scheme instead: `r8` / `r8d` / `r8w` /
`r8b`.

**One rule that surprises everyone:** writing to a 32-bit register
**zeroes the upper 32 bits** of the full 64-bit register. Writing to a
16-bit or 8-bit register does *not*. So `mov eax, 5` sets all of `rax` to
5, but `mov ax, 5` leaves the top 48 bits untouched. This is a hardware
behaviour, not an assembler convenience, and it is a very common source
of bugs. You will verify it yourself in the exercises.

## 2. The three sections

A NASM source file is divided into sections that become segments in the
final binary, each with different memory permissions:

| Section | Contents | Permissions |
|---|---|---|
| `.text` | Your instructions | read + execute |
| `.data` | Initialized data (values you specify) | read + write |
| `.bss`  | Uninitialized data (zero-filled at load) | read + write |

`.bss` costs nothing in the file on disk — it only records "reserve N
bytes," and the kernel zeroes them at load time. `.data` bytes are
physically stored in the file.

## 3. The program

Create `hello.asm`:

```nasm
; hello.asm — write a string to stdout and exit

global _start                   ; export this symbol so `ld` can find it

section .data
    msg     db  "Hello from assembly!", 10   ; 10 = newline (\n)
    msglen  equ $ - msg          ; assemble-time length calculation

section .text
_start:
    ; write(1, msg, msglen)
    mov rax, 1                   ; syscall number 1 = sys_write
    mov rdi, 1                   ; arg1: fd 1 = stdout
    mov rsi, msg                 ; arg2: pointer to the bytes
    mov rdx, msglen              ; arg3: how many bytes
    syscall

    ; exit(0)
    mov rax, 60                  ; syscall number 60 = sys_exit
    xor rdi, rdi                 ; arg1: status 0 (xor r,r is the idiomatic zero)
    syscall
```

Three things worth naming now:

- **`db`** means "define byte(s)". `10` is a raw byte value — ASCII
  newline. NASM has no `"\n"` escape inside strings by default, so this
  is how newlines are done.
- **`$`** means "the current address, here, right now, at assembly time."
  So `$ - msg` is the number of bytes between the start of `msg` and the
  current position — i.e. the string's length, computed by the assembler.
  `equ` binds that number to a name. Nothing about this exists at
  runtime; it's pure assemble-time arithmetic.
- **`xor rdi, rdi`** sets `rdi` to zero. Anything XORed with itself is
  zero. It is one byte shorter than `mov rdi, 0` and is what you'll see
  in essentially all real x86 code, so learn to read it as "= 0".

## 4. Build it

```sh
nasm -f elf64 hello.asm -o hello.o     # assemble
ld hello.o -o hello                     # link
./hello                                  # run
```

**What `nasm` did:** turned mnemonics into machine-code bytes and produced
an *object file* — machine code plus a symbol table plus a list of
addresses it couldn't resolve yet.

**What `ld` did:** decided the final memory layout, resolved those
addresses, wrote the ELF headers (including the entry point, taken from
`_start`), and produced a runnable file.

`_start` is the default entry-point symbol `ld` looks for. If you rename
it, you must tell `ld` with `-e yourname` or linking will fail.

## 5. Look at what you built

Add a listing file — NASM's own record of which bytes it emitted for each
source line:

```sh
nasm -f elf64 hello.asm -o hello.o -l hello.lst
cat hello.lst
```

Columns are: line number, byte offset, hex bytes, source.

Then compare the object file against the linked binary:

```sh
objdump -d hello.o      # the "before" — some addresses are placeholders
objdump -d hello        # the "after" — the linker filled them in
```

Find the `mov rsi, msg` line in each. In `hello.o` its address operand is
zeros; in `hello` it's a real address. That difference *is* linking.

Other inspection tools worth knowing now:

```sh
nm hello                # symbol table: names and their addresses
readelf -h hello        # ELF header, including the entry point address
readelf -S hello        # section headers and their addresses
xxd hello | head        # raw bytes, no interpretation at all
```

## 6. GDB: first steps

```sh
gdb ./hello
```

You get a `(gdb)` prompt. The essential first commands:

```gdb
break _start        # set a breakpoint at the symbol _start
run                 # start the program; it stops at the breakpoint
info registers      # dump all general-purpose registers
stepi               # execute exactly one machine instruction
continue            # resume until the next breakpoint or exit
quit                # leave
```

Almost every GDB command has a short form: `b`, `r`, `si`, `c`, `i r`.
Use them.

### The visual layout

```sh
gdb -tui ./hello
```

TUI mode opens on the **source** view by default. Since you assembled
without debug info, it will say **"No Source Available"** — that is
expected, not an error. Switch to the views that work for raw assembly:

```gdb
layout asm          # live disassembly, current instruction highlighted
layout regs         # add a register pane above it
```

Now `stepi` repeatedly and watch `rax`, `rdi`, `rsi`, `rdx` fill in one
at a time as each `mov` executes. Registers that changed since the last
step are highlighted.

If the TUI ever gets visually corrupted (it happens), `Ctrl-L` redraws.

### Without the TUI

The same information, on demand:

```gdb
disassemble         # disassemble the current function, => marks your position
x/8i $pc            # show the next 8 instructions from the program counter
p $rax              # print one register
p/x $rax            # print it in hex
```

### The one thing to actually verify this lesson

Set a breakpoint, run, and compare the address GDB reports against what
`objdump -d hello` printed statically. They should match exactly. That
confirms something real: `ld` baked fixed addresses into this binary
(this is a *non-PIE* executable), so what you see on disk is what runs.

## Exercises

**1.1 — Change the message.** Edit the string and rebuild. Confirm the
length still comes out right without you touching `msglen`. Why does it?

**1.2 — Exit codes.** Change the exit status from 0 to 42. Rebuild, run,
then check it with `echo $?` in the shell. Now set a breakpoint at
`_start`, run, and use GDB to change the value *at runtime* before it
exits: `set $rdi = 7`, then `continue`, then check `echo $?` again.

**1.3 — Prove the 32-bit zero-extension rule.** Write a program that does
this and nothing else, then step through it in GDB watching `rax`:

```nasm
mov rax, 0xFFFFFFFFFFFFFFFF
mov eax, 1                     ; what happens to the top 32 bits?
mov rax, 0xFFFFFFFFFFFFFFFF
mov ax, 1                      ; and now?
```

Write down what you observed. This is the single most useful surprising
fact in the whole instruction set.

**1.4 — Two writes.** Extend `hello.asm` to print two different strings
with two separate `write` syscalls, without using a loop. Note what you
have to re-set between the calls and what you don't.

**1.5 — Read the bytes.** Run `objdump -d hello` and find the encoding of
`mov rax, 1`. It should be seven bytes: `48 c7 c0 01 00 00 00`. Now find
the encoding of `mov eax, 1` from exercise 1.3 — it's five bytes, `b8 01
00 00 00`. Explain to yourself why the assembler picked a shorter
encoding, and why the shorter one is safe given the zero-extension rule.
