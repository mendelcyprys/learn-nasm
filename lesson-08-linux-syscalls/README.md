# Lesson 08 — Linux Syscalls: What the Kernel Gives You

**Goal:** understand the kernel interface as its own ABI, distinct from
the function-call ABI, and use it to do real I/O.

## 1. The syscall ABI — note the differences

| | Function calls (Lesson 06) | **Syscalls** |
|---|---|---|
| Selector | — | **`rax` = syscall number** |
| Arg 1 | `rdi` | `rdi` |
| Arg 2 | `rsi` | `rsi` |
| Arg 3 | `rdx` | `rdx` |
| Arg 4 | `rcx` | **`r10`** ← different! |
| Arg 5 | `r8` | `r8` |
| Arg 6 | `r9` | `r9` |
| Return | `rax` | `rax` |
| Destroyed | caller-saved set | **`rcx` and `r11` only** |

Two things to burn in:

1. **Argument 4 is `r10`, not `rcx`.** This isn't arbitrary: the `syscall`
   instruction itself uses `rcx` to save the return address and `r11` to
   save `rflags`, so `rcx` is unavailable. Every syscall wrapper in libc
   does `mov r10, rcx` for exactly this reason.
2. **Syscalls preserve everything else.** Unlike a function call, the
   kernel restores all your registers except `rax`, `rcx`, `r11`. You do
   not need to save `rsi`/`rdi`/`rdx` around a syscall.

Also: there is **no stack alignment requirement** for `syscall`, and no
red zone protection — the kernel switches to its own stack.

## 2. Error handling

The kernel returns errors as **small negative numbers** in `rax`. There is
no `errno` variable at this level — libc creates that by checking the
return and storing the negated value.

The convention: a return value in the range `-1` to `-4095` is an error
code.

```nasm
    syscall
    cmp rax, -4096
    ja  .error              ; unsigned above → in the error range
```

(`ja` works because as *unsigned* values, `-1`…`-4095` are the largest
possible numbers.)

Common error codes: `-1` EPERM, `-2` ENOENT, `-9` EBADF, `-13` EACCES,
`-22` EINVAL. Full list in `/usr/include/asm-generic/errno-base.h` — worth
opening in the VM.

## 3. The syscalls you'll actually use

| # | Name | Signature |
|---|---|---|
| 0 | `read` | `(fd, buf, count)` → bytes read, 0 = EOF |
| 1 | `write` | `(fd, buf, count)` → bytes written |
| 2 | `open` | `(path, flags, mode)` → fd |
| 3 | `close` | `(fd)` → 0 |
| 8 | `lseek` | `(fd, offset, whence)` → new offset |
| 9 | `mmap` | `(addr, len, prot, flags, fd, off)` → address |
| 11 | `munmap` | `(addr, len)` → 0 |
| 12 | `brk` | `(addr)` → new break |
| 39 | `getpid` | `()` → pid |
| 60 | `exit` | `(status)` → does not return |
| 231 | `exit_group` | `(status)` → does not return |
| 257 | `openat` | `(dirfd, path, flags, mode)` → fd |

Find any number yourself in the VM:

```sh
grep -w __NR_write /usr/include/asm/unistd_64.h
```

**Use `exit_group` (231), not `exit` (60), in real programs** — `exit`
terminates only the calling thread. For single-threaded code they behave
identically, which is why every tutorial uses 60.

The standard file descriptors: **0 = stdin, 1 = stdout, 2 = stderr**.

Flags for `open`: `O_RDONLY` 0, `O_WRONLY` 1, `O_RDWR` 2, `O_CREAT` 0o100
(64), `O_TRUNC` 0o1000 (512), `O_APPEND` 0o2000 (1024). Combine with `|`.

## 4. Program: read a file, write it out

`fileio.asm` — a minimal `cat`:

```nasm
default rel
global _start

section .data
    filename    db  "/etc/hostname", 0
    errmsg      db  "open failed", 10
    errlen      equ $ - errmsg

section .bss
    buf         resb 4096

section .text
_start:
    ; ---- fd = open("/etc/hostname", O_RDONLY) ----
    mov rax, 2                  ; sys_open
    lea rdi, [filename]
    xor rsi, rsi                 ; O_RDONLY = 0
    xor rdx, rdx                  ; mode unused without O_CREAT
    syscall

    cmp rax, -4096
    ja  .openfail
    mov r12, rax                   ; save fd in a callee-saved register

.readloop:
    ; ---- n = read(fd, buf, 4096) ----
    mov rax, 0                     ; sys_read
    mov rdi, r12
    lea rsi, [buf]
    mov rdx, 4096
    syscall

    cmp rax, 0
    jle .finish                     ; 0 = EOF, negative = error

    ; ---- write(1, buf, n) ----
    mov rdx, rax                     ; n bytes
    mov rax, 1                        ; sys_write
    mov rdi, 1
    lea rsi, [buf]
    syscall

    jmp .readloop

.finish:
    ; ---- close(fd) ----
    mov rax, 3
    mov rdi, r12
    syscall

    xor rdi, rdi
    jmp .exit

.openfail:
    mov rax, 1
    mov rdi, 2                        ; stderr
    lea rsi, [errmsg]
    mov rdx, errlen
    syscall
    mov rdi, 1                         ; exit status 1

.exit:
    mov rax, 231                       ; exit_group
    syscall
```

```sh
nasm -f elf64 fileio.asm -o fileio.o -l fileio.lst
ld fileio.o -o fileio
./fileio
```

Note `r12` holding the fd across syscalls — safe because syscalls preserve
it, and it would *also* be safe across function calls, which is why it's
the right choice.

## 5. Command-line arguments

At `_start`, the stack holds:

```
[rsp]        argc
[rsp + 8]    argv[0]   (pointer to program name)
[rsp + 16]   argv[1]
[rsp + 24]   argv[2]
...
             NULL
             envp[0], envp[1], ..., NULL
```

There is no `main(argc, argv)` here — the kernel hands you a raw stack.
Reading argv:

```nasm
_start:
    mov r12, [rsp]              ; argc
    lea r13, [rsp + 8]           ; argv
    mov rdi, [r13 + 8]            ; argv[1]
```

Do **not** `pop` these unless you mean to — you'd lose the layout.

## 6. GDB: syscall debugging

### `catch syscall` — break on kernel entry

```gdb
catch syscall write         # break on every write
catch syscall               # break on every syscall (noisy!)
catch syscall 1             # by number
catch syscall read write open close
```

GDB stops **twice** per syscall: once on entry (showing arguments) and
once on return (showing the value). The output distinguishes them:

```
Catchpoint 1 (call to syscall write), ...
Catchpoint 1 (returned from syscall write), ...
```

### The practical session

```gdb
break _start
run
catch syscall open
continue
i r rax rdi rsi rdx         # inspect args at entry
continue                     # now at the return
p $rax                       # the fd, or a negative error
```

`i r rax rdi rsi rdx` (short for `info registers`) is the syscall
debugging command — it shows the number and the first three arguments in
one line.

### Reading the buffer after a read

```gdb
catch syscall read
continue                     # entry
continue                     # return — rax = bytes read
x/s &buf                     # see what actually landed
x/64xb &buf                  # raw bytes
```

### Decoding errors

```gdb
p $rax
p (int)$rax                  # as a signed 32-bit value: -2, -13, etc.
```

A returned `-2` is ENOENT. Cross-check against
`/usr/include/asm-generic/errno-base.h`.

### Compare against `strace`

```sh
apk add strace
strace ./fileio
```

`strace` shows every syscall with decoded arguments and symbolic error
names. Use it to get the overview, then GDB to dig into one call. They're
complementary: `strace` for *what happened*, GDB for *why*.

## Exercises

**8.1 — Handle a real error.** Change the filename to something that
doesn't exist. Run under GDB with `catch syscall open` and read the
negative return value. Confirm it matches ENOENT in the errno header.

**8.2 — Write to a file.** Extend the program to `open` an output file
with `O_WRONLY | O_CREAT | O_TRUNC` (flags = 1 | 64 | 512 = 577) and mode
`0644` (decimal 420), and copy input to it instead of stdout. You now have
`cp`.

**8.3 — Contract: `write_all`.**

> `write_all(rdi = fd, rsi = buf, rdx = count) -> rax = 0 on success, negative errno on failure`
> Must handle short writes by looping until all bytes are written.

Short writes are real — `write` may transfer fewer bytes than requested.
Nearly every tutorial ignores this; don't.

**8.4 — Read argv.** Write a program that prints its own `argv[1]` to
stdout, or an error if `argc < 2`. You'll need `strlen` from Lesson 06.
Test with `./prog hello` under GDB, inspecting `x/gx $rsp` at the very
first instruction.

**8.5 — Contract: `read_line`.**

> `read_line(rdi = fd, rsi = buf, rdx = maxlen) -> rax = length read`
> Reads bytes until a newline or maxlen. The newline is not stored.

Test it on stdin: run the program and type input.

**8.6 — Explore with strace.** Run `strace /bin/echo hi` and count the
syscalls. Identify which are the program doing its job versus the dynamic
loader setting up. Then compare with `strace ./fileio` — your static
binary should be dramatically shorter. Explain why.

**8.7 — `mmap` a file.** Use `mmap` (syscall 9) with `PROT_READ` (1) and
`MAP_PRIVATE` (2) to map a file into memory, then read bytes from it
directly with `mov` instead of using `read`. Inspect the returned address
in GDB with `x/64xb $rax`. This is the mechanism behind every fast file
reader.
