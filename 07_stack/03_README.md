# 07_stack / 03.asm – `call` and `ret` with no frame (64-bit)

## What the program does

```nasm
_start:
    mov rax, 10
    call calculate       ; push return address, jump
after_call:
    ; RAX should now be 15
    nop
    mov rax, 60
    xor rdi, rdi
    syscall

calculate:
    add rax, 5
    ret                  ; pop return address into RIP
```

This is the smallest possible use of the stack by a function: the **only** thing placed on the stack is the **return address**. There is no `push rbp`, no locals, and no `push`/`pop` written by the programmer. The function takes its argument and returns its result in RAX.

## Build, run, trace

```bash
nasm -f elf64 -g -F dwarf 03.asm -o 03.o
ld 03.o -o 03
./03 ; echo $?             

gdb ./03
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) p/x $rsp
(gdb) x/2gx $rsp           
```

## Execution walkthrough

At `_start`: RSP = `0x7fffffff9370` (holds `argc`).

| # | Instruction | RAX after | RSP after | What happens |
|---|-------------|-----------|-----------|--------------|
| 1 | `mov rax, 10` | 10 | `...9370` | No stack use. |
| 2 | `call calculate` | 10 | `...9368` | RSP -= 8, then writes the address of the next instruction, `0x40100a` (`after_call`), at `[...9368]`. RIP = `calculate` (`0x401015`). |
| 3 | `add rax, 5` | 15 | `...9368` | Does the work; no stack change. |
| 4 | `ret` | 15 | `...9370` | RIP = `[RSP]` = `0x40100a`, then RSP += 8. |
| 5 | `nop` (at `after_call`) | 15 | `...9370` | Back in `_start`; stack balanced. |
| 6-8 | `mov rax,60` / `xor rdi,rdi` / `syscall` | | `...9370` | Exit. |

### Stack while inside `calculate`

```
 higher addresses
 0x7fffffff9370 | 0x0000000000000001 |   argc                 <- RSP before the call
 0x7fffffff9368 | 0x000000000040100a |   return address       <- RSP inside calculate
 lower addresses
```

RSP over the program: `9370 -> 9368 (call) -> 9368 (add) -> 9370 (ret)`.

## Observations from running it

1. **`call` = push + jump.** The return address saved is `0x40100a`: `call` sits at `0x401005` and is 5 bytes long, so the next instruction (the `after_call` label) is at `0x40100a`. gdb confirms RIP lands exactly on `after_call` after `ret`.
2. **`ret` = pop + jump.** It reads the 8 bytes at RSP into RIP and adds 8 to RSP. This only works because RSP pointed at the return address when `ret` ran, i.e. `calculate` left the stack exactly as it found it. If `calculate` had pushed something and not popped it, `ret` would have jumped to that value instead.
3. **RAX is the result.** The comment in the source says "RAX should now be 15", and gdb shows `rax=0xf` at `after_call`, so the call worked. The data travelled through the register, not the stack.
4. **Stale data remains.** After `ret`, memory at `...9368` still contains `0x40100a`; the value is not cleared but lies below RSP, so it is dead data and would be overwritten by the next push/call.
5. Because there is no frame setup, RBP stays `0` throughout, and a debugger cannot walk this "frame" using RBP. It only has the return address to go on.
6. The program exits with status `0`.
