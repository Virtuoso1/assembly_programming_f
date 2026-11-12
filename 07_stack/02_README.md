# 07_stack / 02.asm – A stack frame with local variables (64-bit)

## What the program does

```nasm
_start:
    call work            ; push return address, jump to work
    mov rax, 60
    xor rdi, rdi
    syscall

work:
    push rbp             ; save caller's frame pointer
    mov rbp, rsp         ; new frame pointer
    sub rsp, 16          ; reserve 16 bytes for two locals

    mov qword [rbp - 8],  100
    mov qword [rbp - 16], 200

    mov rax, [rbp - 8]   ; rax = 100
    mov rbx, [rbp - 16]  ; rbx = 200
    nop

    mov rsp, rbp         ; throw away the locals
    pop rbp              ; restore caller's frame pointer
    ret                  ; pop return address into RIP
```

This is the classic **function prologue / epilogue**: the function `work` builds a *stack frame*, stores two local variables in it, reads them back, and tears the frame down before returning.

## Build, run, trace

```bash
nasm -f elf64 -g -F dwarf 02.asm -o 02.o
ld 02.o -o 02
./02 ; echo $?             

gdb ./02
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) p/x $rsp  ;  p/x $rbp
(gdb) x/6gx $rsp-16
```

## Execution walkthrough

At `_start`: RSP = `0x7fffffff9370` (holds `argc`), RBP = 0 (the kernel gives a zeroed RBP).

| # | Instruction | RSP after | RBP after | What happens |
|---|-------------|-----------|-----------|--------------|
| 1 | `call work` | `...9368` | `0` | `call` **pushes the return address** (`0x401005`, the instruction after the call) at `[...9368]`, then jumps to `work`. |
| 2 | `push rbp` | `...9360` | `0` | RSP -= 8; saves the caller's RBP (value 0) at `[...9360]`. |
| 3 | `mov rbp, rsp` | `...9360` | `...9360` | RBP now marks the base of `work`'s frame. No memory change. |
| 4 | `sub rsp, 16` | `...9350` | `...9360` | RSP -= 16 reserves space for locals. Nothing is written yet. |
| 5 | `mov qword [rbp-8], 100` | `...9350` | `...9360` | Writes 100 to `0x...9358`. **RSP unchanged**. |
| 6 | `mov qword [rbp-16], 200` | `...9350` | `...9360` | Writes 200 to `0x...9350`. RSP unchanged. |
| 7 | `mov rax, [rbp-8]` | `...9350` | `...9360` | RAX = 100 (read, not a pop). |
| 8 | `mov rbx, [rbp-16]` | `...9350` | `...9360` | RBX = 200. |
| 9 | `nop` | `...9350` | `...9360` | Snapshot point (below). |
| 10 | `mov rsp, rbp` | `...9360` | `...9360` | RSP jumps back up to RBP, so the locals are *released* in one instruction. |
| 11 | `pop rbp` | `...9368` | `0` | RBP = `[RSP]` = 0 (the saved value); RSP += 8. |
| 12 | `ret` | `...9370` | `0` | **Pops the return address** `0x401005` into RIP, RSP += 8. Execution resumes right after the `call`. |
| 13-15 | `mov rax,60` / `xor rdi,rdi` / `syscall` | `...9370` | `0` | Exit. |

### Stack frame at the `nop` (inside `work`)

```
 higher addresses
 0x7fffffff9370 | 0x0000000000000001 |   argc (belongs to the kernel / _start)
 0x7fffffff9368 | 0x0000000000401005 |   return address (pushed by CALL)
 0x7fffffff9360 | 0x0000000000000000 |   saved RBP (pushed by PUSH RBP)   <- RBP = here
 0x7fffffff9358 | 0x0000000000000064 |   local 1  [rbp-8]  = 100
 0x7fffffff9350 | 0x00000000000000c8 |   local 2  [rbp-16] = 200          <- RSP = here
 lower addresses
```

RSP over the whole program: `9370 -> 9368 (call) -> 9360 (push) -> 9350 (sub) -> 9360 (mov rsp,rbp) -> 9368 (pop) -> 9370 (ret)`. It ends where it started.

## Observations from running it

1. **Two different ways of using the stack.** `call`/`push`/`pop`/`ret` move RSP automatically. The locals are written with plain `mov [rbp-N]`, which does **not** change RSP, because space was reserved in advance with `sub rsp, 16`. Locals are addressed relative to RBP because RSP could move, but RBP is fixed for the lifetime of the function.
2. **Return address.** The value `0x401005` on the stack is the address of `mov rax, 60` in `_start`. `ret` is really just `pop rip`, and after it executes, gdb shows execution continuing exactly at `_start+5`. If the epilogue had not put RSP back on the saved return address, `ret` would have jumped to garbage. **Tested:** deleting the `mov rsp, rbp` line makes `pop rbp` take 200 (a local) instead of the saved RBP, and `ret` then pops 100 as the "return address", so the program dies with a segmentation fault (exit status 139).
3. **The saved RBP is 0** because `_start` never sets up a frame pointer; the value that `push rbp` saved is simply the kernel's initial RBP. After `pop rbp` it returns to 0.
4. **`mov rsp, rbp` frees the locals without erasing them.** After the epilogue, gdb still shows `0xc8` and `0x64` in memory below RSP; they are simply no longer protected and will be overwritten by the next use of the stack.
5. **Alignment.** RSP is 16-byte aligned at `_start` (`...70`), becomes 8 mod 16 after `call` (`...68`, as the System V ABI expects at function entry), is aligned again after `push rbp` (`...60`), and stays aligned because 16 was subtracted.
6. After returning, `RAX = 100` and `RBX = 200` are still set: values loaded from the frame outlive the frame. The program exits with status `0`.
