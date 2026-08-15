# Lesson 09 — Linking with C and Calling libc

**Goal:** stop reimplementing everything. Call `printf`, `malloc`, and
friends from assembly — and let C call your assembly.

This is where Lesson 06's ABI rules stop being theoretical. Real libc code
*assumes* the contract, and violating it produces crashes deep inside
library functions.

## 1. `extern` and `global`

```nasm
extern printf           ; defined elsewhere — the linker will find it
extern malloc
extern free

global my_asm_function  ; defined here — make it visible to others
```

`extern` tells NASM "this symbol exists somewhere else, emit a relocation
and let the linker sort it out."

## 2. Entry point: `main`, not `_start`

When you link against libc, **libc provides `_start`**. It sets up the
runtime (stdio buffers, TLS, locale, `atexit` handlers, argv/envp
processing) and then calls `main`. If you define your own `_start` and
also use libc, you'll get duplicate-symbol errors — or worse, an
uninitialized libc that segfaults inside `printf`.

So:

```nasm
global main
extern printf

section .text
main:
    ...
    ret                 ; returning from main = exit with rax as the status
```

## 3. Linking

Use `gcc` as the linker driver, not `ld` directly:

```sh
nasm -f elf64 prog.asm -o prog.o
gcc prog.o -o prog                    # gcc knows all the libc startup files
./prog
```

Calling `ld` by hand requires naming `crt1.o`, `crti.o`, `crtn.o`, the
dynamic linker path, and the library search paths — `gcc` does all of it.
Use `gcc`.

Two flags worth knowing:

```sh
gcc -static prog.o -o prog        # statically link libc: bigger, no runtime deps
gcc -no-pie prog.o -o prog        # disable PIE — needed if you use absolute addressing
```

**On Alpine specifically:** Alpine uses **musl** libc, not glibc. It's
smaller, stricter, and mostly identical for our purposes. One visible
difference: musl is far less forgiving of ABI violations, so a stack
misalignment that "works" on glibc may crash on musl. That's arguably a
feature while learning.

## 4. Calling a variadic function — the `al` rule

`printf` is variadic, and the ABI has a special requirement for variadic
calls:

> **`al` must contain the number of vector (XMM) registers used to pass
> arguments.**

If you pass no floats, `al` must be 0. If you pass two doubles, `al` must
be 2. Getting this wrong causes `printf` to read garbage or crash, because
the variadic prologue uses `al` to decide whether to spill all 8 XMM
registers to its register-save area.

```nasm
    lea rdi, [fmt]
    mov rsi, 42
    xor eax, eax            ; al = 0: no float arguments
    call printf
```

`xor eax, eax` before `call printf` is one of the most common lines in
hand-written x86-64 assembly. Now you know why.

## 5. Program: using libc

`libc.asm`:

```nasm
default rel
global main
extern printf
extern malloc
extern free
extern strlen

section .data
    fmt_int     db  "The answer is %ld", 10, 0
    fmt_two     db  "%s has length %ld", 10, 0
    fmt_dbl     db  "pi is about %f", 10, 0
    text        db  "hello, world", 0
    pi          dq  3.14159265358979

section .text
main:
    push rbp
    mov rbp, rsp            ; rsp is now 16-byte aligned — safe to call

    ; ---- printf("The answer is %ld\n", 42) ----
    lea rdi, [fmt_int]
    mov rsi, 42
    xor eax, eax             ; zero float args
    call printf

    ; ---- strlen(text), then printf with two args ----
    lea rdi, [text]
    call strlen               ; result in rax
    mov rdx, rax               ; 3rd printf arg
    lea rdi, [fmt_two]
    lea rsi, [text]             ; 2nd printf arg
    xor eax, eax
    call printf

    ; ---- printf with a double ----
    lea rdi, [fmt_dbl]
    movsd xmm0, [pi]            ; floats go in xmm registers (Lesson 11)
    mov eax, 1                   ; al = 1: ONE vector register used
    call printf

    ; ---- malloc / free ----
    mov rdi, 256
    call malloc
    test rax, rax
    jz .nomem
    mov r12, rax                  ; save the pointer (callee-saved!)

    mov qword [r12], 0xCAFEBABE   ; use the memory

    mov rdi, r12
    call free

.nomem:
    xor eax, eax                   ; return 0 from main
    leave
    ret
```

```sh
nasm -f elf64 libc.asm -o libc.o -l libc.lst
gcc libc.o -o libcdemo
./libcdemo
```

Note `r12` holding the malloc pointer. `rax` would have been destroyed by
the next `call` — this is the callee-saved rule doing real work.

## 6. Calling assembly from C

`asmlib.asm`:

```nasm
global sum_array
section .text

; long sum_array(const long *arr, long n)
sum_array:
    xor rax, rax
    xor rcx, rcx
.loop:
    cmp rcx, rsi
    jge .done
    add rax, [rdi + rcx*8]
    inc rcx
    jmp .loop
.done:
    ret
```

`driver.c`:

```c
#include <stdio.h>
long sum_array(const long *arr, long n);

int main(void) {
    long v[5] = {10, 20, 30, 40, 50};
    printf("sum = %ld\n", sum_array(v, 5));
    return 0;
}
```

```sh
nasm -f elf64 asmlib.asm -o asmlib.o
gcc driver.c asmlib.o -o driver
./driver
```

This works *only* because both sides follow the ABI. The C prototype and
your register usage must agree — nothing checks this, and a mismatch
produces silent nonsense.

## 7. The PLT and dynamic linking

Disassemble your binary:

```sh
objdump -d libcdemo | grep -A3 'call.*printf'
```

You'll see the call target is something like `printf@plt`, not `printf`.
The **PLT** (Procedure Linkage Table) is a small stub: on the first call
it jumps into the dynamic linker, which resolves the real address of
`printf` in the loaded libc and patches the **GOT** (Global Offset Table);
subsequent calls go straight through. This is lazy binding.

Consequences for debugging:
- A breakpoint on `printf` before the first call may not resolve.
- Stepping into a call with `si` lands you in PLT stub code, not libc.

## 8. GDB: debugging across the library boundary

### `si` vs `ni` — the crucial distinction

```gdb
stepi / si      # step INTO the call — you end up in libc
nexti / ni      # step OVER the call — the whole call runs, you land after it
```

Use `ni` for library calls unless you specifically want to see inside.
Stepping into `printf` with `si` means thousands of instructions.

### Escaping when you've stepped in too far

```gdb
finish          # run until the current function returns
```

### Breaking in libc

```gdb
break printf
run
bt              # see your code below the libc frames
finish          # come back out
```

Alpine's musl is stripped by default, so you may get bare addresses rather
than function names inside libc. Install debug symbols if you want more:

```sh
apk add musl-dbg gdb
```

### Inspecting arguments at a libc call

Break on the `call printf` instruction (not on `printf` itself) and read
the ABI registers directly:

```gdb
x/s $rdi        # the format string
p $rsi          # first variadic arg
p $rdx          # second
p $al           # the vector-register count!
```

Checking `p $al` is how you diagnose the variadic bug described above.

### Checking alignment before a call

```gdb
p $rsp % 16     # must be 0 immediately BEFORE the call executes
```

Make this a reflex. When a program crashes inside libc for no apparent
reason, this is the first thing to check.

## Exercises

**9.1 — Break alignment deliberately.** Add a single `push rax` before a
`call printf` (making `rsp` misaligned) and run it. On musl this may crash
outright. Diagnose it in GDB with `p $rsp % 16`, then fix it.

**9.2 — The `al` bug.** Remove the `xor eax, eax` before a `printf` call
and instead set `eax` to some large garbage value. Observe what happens.
Explain it in terms of what the variadic prologue does with `al`.

**9.3 — Contract: `count_words`, callable from C.**

> `long count_words(const char *s);`
> Returns the number of whitespace-separated words.

Write the assembly, write a C driver that tests it on several strings,
and link them together.

**9.4 — Dynamic array.** Use `malloc` to allocate an array of N longs (N
read from `argv[1]` via `atoi`), fill it with squares, print them with
`printf`, then `free` it. Watch the returned pointer in GDB with
`x/8gx $rax` immediately after `malloc`.

**9.5 — Trace the PLT.** Set a breakpoint on the first `call printf@plt`.
`si` into it and follow the jumps until you reach real libc code. Note how
many hops it takes. Then `continue` past a second `printf` call and `si`
again — it should be shorter now. Explain why (lazy binding).

**9.6 — Contract: comparator for `qsort`.**

> Write `int cmp_long(const void *a, const void *b)` in assembly,
> returning negative/zero/positive.
> Then call libc's `qsort` from assembly to sort an array using it.

This exercises passing a function pointer *to* C and having C call *your*
code — both directions of the ABI at once. Break inside your comparator
and run `bt` to see qsort's frames.

**9.7 — Static vs dynamic.** Build the same program with and without
`-static`. Compare sizes with `ls -l`, compare syscall counts with
`strace`, and compare `objdump -d` around the printf call. What happened
to the PLT?
