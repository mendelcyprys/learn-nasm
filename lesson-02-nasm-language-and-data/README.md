# Lesson 02 — The NASM Language and Data in Memory

**Goal:** learn NASM's syntax properly, define and lay out data, and
learn to read and modify memory in GDB.

## 1. Anatomy of a NASM line

```
label:    instruction   operand, operand    ; comment
```

Every part is optional. NASM is case-sensitive for labels (`Foo` and
`foo` are different) but not for instruction mnemonics.

Operand order is **destination first**:

```nasm
mov rax, rbx        ; rax = rbx    (Intel syntax — NASM's style)
```

If you have ever read AT&T syntax (what `gcc -S` and, by default,
`objdump` produce), the order is reversed there and registers carry `%`.
You can make GDB and objdump match NASM:

```sh
objdump -d -M intel hello        # Intel syntax from objdump
```
```gdb
set disassembly-flavor intel     # Intel syntax in GDB
```

Do both. Mixing syntaxes while learning is needless confusion. Put the
GDB line in `~/.gdbinit` so it's permanent.

## 2. Labels: global and local

```nasm
global my_function          ; visible to the linker, appears in the symbol table
my_function:
.loop:                       ; local label — belongs to my_function
    dec rcx
    jnz .loop                 ; refers to my_function.loop
    ret

other_function:
.loop:                        ; a *different* label, no conflict
    ret
```

A label starting with `.` is scoped to the most recent non-local label
above it. This lets you use `.loop` and `.done` in every function without
inventing unique names. Local labels do not appear in the symbol table,
so GDB can't break on them by name — which is exactly why you'll learn
address-based breakpoints later.

## 3. Defining data

In `.data` — values physically stored in the file:

| Directive | Size | Example |
|---|---|---|
| `db` | 1 byte | `db 0x41, 'B', 10` |
| `dw` | 2 bytes (word) | `dw 1000, 0xFFFF` |
| `dd` | 4 bytes (doubleword) | `dd 123456, 3.14` |
| `dq` | 8 bytes (quadword) | `dq 1234567890123, 2.718` |
| `dt` | 10 bytes (x87 extended) | `dt 3.14159265358979` |

In `.bss` — reserve space, zero-filled at load, costs nothing on disk:

| Directive | Reserves |
|---|---|
| `resb n` | n bytes |
| `resw n` | n words (2n bytes) |
| `resd n` | n doublewords |
| `resq n` | n quadwords |

Useful extras:

```nasm
zeros:    times 64 db 0        ; repeat a directive n times
pattern:  times 8 dq 0xDEADBEEF
name:     db "alice", 0         ; C-style NUL-terminated string
count:    equ 10                ; assemble-time constant, not memory
```

`equ` does **not** allocate anything. It's a named number the assembler
substitutes at build time. `dq 10` allocates 8 bytes containing 10;
`equ 10` allocates nothing.

## 4. Reading and writing memory

Square brackets mean **dereference** — "the contents at this address":

```nasm
mov rax, msg          ; rax = the ADDRESS of msg
mov rax, [msg]        ; rax = the 8 BYTES STORED AT msg
```

This distinction is the single most common beginner error in NASM. In
`.data`, `msg` is a label — a number, an address. Brackets go and fetch.

### Size ambiguity

When neither operand tells NASM how many bytes to touch, you should say
so explicitly:

```nasm
mov [counter], 5              ; AMBIGUOUS — see below
mov qword [counter], 5        ; 8 bytes
mov dword [counter], 5        ; 4 bytes
mov byte  [counter], 5        ; 1 byte
```

The ambiguity is real because **a label has no type in NASM**. Even if
`counter` was declared `resq 1` or `dq 0`, the label is just an address by
the time you write `[counter]` — NASM does not carry the declaration
forward the way MASM does.

What happens without a suffix depends on your NASM version, and the
modern behaviour is the dangerous one:

- **NASM 2.x** rejects it: `error: operation size not specified`.
- **NASM 3.x** accepts it and silently defaults to **byte**. The listing
  shows opcode `C6` (`MOV r/m8, imm8`) and a one-byte immediate.

So on NASM 3, `mov [result], 5` against a `resq 1` writes **one** byte
and leaves the other seven whatever they were. No error, no warning. This
is precisely the bug the suffix prevents, and it fails silently — always
write the suffix.

With a register operand there's no ambiguity — the register's width
decides:

```nasm
mov [counter], rax            ; fine: rax is 64-bit, so 8 bytes
mov [counter], al             ; fine: al is 8-bit, so 1 byte
```

### Moving smaller values into bigger registers

```nasm
movzx rax, byte [flag]        ; zero-extend: upper bits = 0
movsx rax, byte [temp]        ; sign-extend: upper bits = copy of sign bit
```

Use `movzx` for unsigned quantities, `movsx` for signed ones. Getting
this wrong turns `-1` (byte `0xFF`) into `255` or vice versa.

## 5. A worked program

`data.asm` — stores values, reads them back, prints one digit:

```nasm
global _start

section .data
    number      dq  7                  ; 8-byte integer
    letters     db  "ABCDE"             ; 5 bytes
    small       db  200                  ; unsigned 200, or signed -56

section .bss
    result      resq 1                   ; 8 reserved bytes, zeroed at load
    outbuf      resb 16

section .text
_start:
    mov rax, [number]        ; load the value 7
    add rax, 3                ; rax = 10
    mov [result], rax          ; store 10 back into .bss

    movzx rbx, byte [small]    ; rbx = 200
    movsx rcx, byte [small]    ; rcx = -56  (same byte, different reading!)

    ; print the third letter, 'C'
    mov al, [letters + 2]      ; byte at offset 2
    mov [outbuf], al
    mov byte [outbuf + 1], 10   ; newline

    mov rax, 1
    mov rdi, 1
    mov rsi, outbuf
    mov rdx, 2
    syscall

    mov rax, 60
    xor rdi, rdi
    syscall
```

```sh
nasm -f elf64 data.asm -o data.o -l data.lst
ld data.o -o data
./data
```

## 6. GDB: examining memory

This is the lesson's core new skill. The command is `x` (examine), and it
takes a format:

```
x/NFU address
```

- **N** = how many units
- **F** = format: `x` hex, `d` signed decimal, `u` unsigned, `c` char,
  `s` string, `i` instruction, `f` float, `t` binary
- **U** = unit size: `b` byte, `h` half (2), `w` word (4), `g` giant (8)

The unit letters are a historical wart — note that GDB's `w` is 4 bytes
while NASM's "word" is 2. Learn GDB's set separately.

Try all of these against your program:

```gdb
break _start
run

x/1gd &number          # one giant, signed decimal   -> 7
x/8xb &number          # the same 8 bytes in hex     -> 07 00 00 00 ...
x/5c &letters          # five characters             -> 65 'A', 66 'B', ...
x/s &letters           # as a string (runs past the end — no NUL!)
x/1xb &small           # -> 0xc8
x/16xb &outbuf         # the .bss buffer, all zeros before you write to it
```

**Always write `&symbol`, not bare `symbol`.** Your NASM binary has no
debug info, so GDB knows these names only as *minimal symbols* — a name
and an address, with no type attached. Bare `letters` asks GDB for the
symbol's **value**, which it can't work out without knowing the type, and
you'll get an error along the lines of `'letters' has unknown type; cast
it to its declared type`. `&letters` asks for its **address**, which is
exactly what the minimal symbol table does know, and what `x` wants
anyway.

If you ever do want the bare form, cast it explicitly:

```gdb
p (char[5]) letters
p *(long*)&number
```

Notice `x/8xb &number` shows `07 00 00 00 00 00 00 00` — the least
significant byte first. That's **little-endian** byte order, and x86 is
little-endian throughout. It matters constantly once you start looking at
raw memory.

### Printing and setting

```gdb
p $rax                 # print a register
p/x $rax               # in hex
p/t $rax               # in binary
p (char) $al           # cast

set $rax = 100             # change a register mid-run
set {long}&result = 999    # change memory: write 8 bytes at &result
set {char}&outbuf = 88     # write one byte
```

**Use C type names here, not NASM ones.** `set {qword}...` fails with the
confusing message `No symbol table is loaded` — GDB doesn't know the word
`qword`, so it tries to look it up as a symbol and reports the lookup
failure rather than a syntax error. The sizes map like this:

| NASM | GDB / C | Bytes |
|---|---|---|
| `byte` | `char` | 1 |
| `word` | `short` | 2 |
| `dword` | `int` | 4 |
| `qword` | `long` | 8 |

An equivalent form, which some people find clearer:

```gdb
set var *(long*)&result = 999
set var *(char*)&outbuf = 88
```

Being able to *change* state mid-run is what makes a debugger better than
print statements: you can test a branch without editing and rebuilding.

### Watching values as you step

```gdb
display/x $rax         # re-print rax in hex after every step
display/8xb &result    # and these 8 bytes too
info display           # list what's being auto-displayed
undisplay 1            # stop displaying item 1
```

Set up a couple of `display` expressions before you start stepping and
GDB becomes a live dashboard instead of something you have to interrogate.

## Exercises

**2.1 — Endianness by hand.** Define `dq 0x1122334455667788`. Predict the
output of `x/8xb` on it *before* running it. Then check.

**2.2 — Signed vs unsigned.** Using the `small` byte (200) from the
program above, confirm in GDB that `movzx` and `movsx` really do produce
200 and -56. Print `rbx` and `rcx` both as `p/d` and `p/u` and explain the
four numbers you get.

**2.3 — Build a string in `.bss`.** Reserve 16 bytes. Write the
characters `H`, `i`, `!`, newline into it one `mov byte` at a time, then
print it with one `write` syscall. Use `x/16xb` in GDB after each store to
watch the buffer fill.

**2.4 — Size suffix hunting.** Write `mov [result], 5` with no size
suffix and assemble it *with a listing file*:

```sh
nasm -f elf64 data.asm -o data.o -l data.lst
```

On NASM 3.x this assembles without complaint. Find the line in
`data.lst` and decode the bytes yourself. You should see something like:

```
C6 04 25 [00000000] 05
```

- `C6` — the opcode. Look it up: `MOV r/m8, imm8`. **Eight-bit.**
- `04` — ModRM byte; `rm=100` means a SIB byte follows.
- `25` — SIB; encodes "disp32, no base register."
- `[00000000]` — the relocation placeholder for `result`'s address
  (Lesson 01 territory — the linker fills this in).
- `05` — the immediate, one byte.

Confirm the total is 8 bytes by checking the offset of the *next* line in
the listing.

Now write it three ways explicitly (`qword`, `dword`, `byte`), assemble
each, and compare opcodes. Then run each under GDB and use
`x/8xb &result` to see how many of the eight reserved bytes each version
actually touched.

The lesson: `resq 1` reserved eight bytes, and the no-suffix version wrote
**one**, silently leaving seven untouched. `result` is only an address —
the `resq` gave the label no type, so NASM fell back to its own default
rather than to your declaration. (MASM would have inferred qword here.
NASM does not.)

Bonus: try `mov [result], rax` and `mov [result], al` and confirm from the
listing that these need no suffix at all — the register width settles it.

**2.5 — Contract exercise.** Write a code fragment that satisfies this
contract, and verify it in GDB:

> On entry: `rsi` holds the address of a 5-byte array.
> On exit: `rax` holds the value of the *last* byte of that array,
> zero-extended. No other register is modified.

**2.6 — Reverse a value with `set`.** Run your program, break before the
`write` syscall, and use GDB's `set {char}` to change what gets printed
without rebuilding. Confirm the output changes.
