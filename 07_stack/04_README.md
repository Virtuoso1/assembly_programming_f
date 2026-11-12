# 07_stack / 04.asm – A stack frame with local variables (32-bit)

## What the program does

```nasm
BITS 32
_start:
    call work
    mov eax, 1           ; sys_exit (i386)
    xor ebx, ebx         ; exit code 0
    int 0x80

work:
    push ebp
    mov ebp, esp
    sub esp, 8           ; room for two 4-byte locals

    mov dword [ebp - 4], 100
    mov dword [ebp - 8], 200

    mov eax, [ebp - 4]
    mov ebx, [ebp - 8]
    nop

    mov esp, ebp
    pop ebp
    ret
```

This is the 32-bit version of `02.asm`: same prologue/epilogue pattern, but with the 32-bit registers (`ESP`, `EBP`, `EIP`) and **4-byte** stack slots instead of 8.

## Build, run, trace

```bash
nasm -f elf32 -g -F dwarf 04.asm -o 04.o
ld -m elf_i386 04.o -o 04
./04 ; echo $?             

gdb ./04
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) p/x $esp  ;  p/x $ebp
(gdb) x/6wx $esp-12        
```

## Execution walkthrough

At `_start`: ESP = `0xffff8800` (holds `argc = 1`), EBP = 0.

| # | Instruction | ESP after | EBP after | What happens |
|---|-------------|-----------|-----------|--------------|
| 1 | `call work` | `...87fc` | `0` | ESP -= **4**; pushes the return address `0x08049005` at `[...87fc]`; jumps to `work`. |
| 2 | `push ebp` | `...87f8` | `0` | ESP -= 4; saves caller's EBP (0) at `[...87f8]`. |
| 3 | `mov ebp, esp` | `...87f8` | `...87f8` | EBP = base of this frame. |
| 4 | `sub esp, 8` | `...87f0` | `...87f8` | Reserves 8 bytes for two dword locals. |
| 5 | `mov dword [ebp-4], 100` | `...87f0` | `...87f8` | Writes 100 to `0x...87f4`. ESP unchanged. |
| 6 | `mov dword [ebp-8], 200` | `...87f0` | `...87f8` | Writes 200 to `0x...87f0`. ESP unchanged. |
| 7 | `mov eax, [ebp-4]` | `...87f0` | `...87f8` | EAX = 100. |
| 8 | `mov ebx, [ebp-8]` | `...87f0` | `...87f8` | EBX = 200. |
| 9 | `nop` | `...87f0` | `...87f8` | Snapshot point. |
| 10 | `mov esp, ebp` | `...87f8` | `...87f8` | Releases the locals (ESP back up to EBP). |
| 11 | `pop ebp` | `...87fc` | `0` | EBP = saved value (0); ESP += 4. |
| 12 | `ret` | `...8800` | `0` | EIP = `[ESP]` = `0x08049005`; ESP += 4. |
| 13-15 | `mov eax,1` / `xor ebx,ebx` / `int 0x80` | `...8800` | `0` | Exit through the 32-bit syscall interface. |

### Stack frame at the `nop`

```
 higher addresses
 0xffff8800 | 0x00000001 |   argc (kernel / _start)
 0xffff87fc | 0x08049005 |   return address (pushed by CALL)
 0xffff87f8 | 0x00000000 |   saved EBP (pushed by PUSH EBP)    <- EBP = here
 0xffff87f4 | 0x00000064 |   local 1  [ebp-4] = 100
 0xffff87f0 | 0x000000c8 |   local 2  [ebp-8] = 200            <- ESP = here
 lower addresses
```

ESP over the program: `8800 -> 87fc -> 87f8 -> 87f0 -> 87f8 -> 87fc -> 8800`. Steps are always multiples of 4.

## Observations from running it

1. **Everything is half the size of 64-bit.** `call` and `push` subtract 4 from ESP, the return address is 4 bytes (`0x08049005`), and the locals are at `[ebp-4]` and `[ebp-8]`. In `02.asm` the same layout used 8-byte slots, `[rbp-8]` and `[rbp-16]`, and RSP moved in steps of 8.
2. **Same mechanics:** `call` pushes the address of the next instruction, `push ebp` / `mov ebp, esp` creates a fixed base for the frame, `sub esp, 8` allocates, `mov esp, ebp` + `pop ebp` frees it, and `ret` pops the return address back into EIP. gdb shows execution resuming at `_start+5` (`0x08049005`) after `ret`.
3. **Locals do not move ESP.** The two `mov dword [ebp-N]` writes leave ESP at `...87f0`; only the `sub esp, 8` allocated the space.
4. **Memory is not erased on release.** After `ret`, `0xc8` and `0x64` are still readable at `...87f0` / `...87f4`, below ESP, i.e. dead but not zeroed.
5. **Exit code is set explicitly.** Right before `int 0x80`, gdb shows `ebx = 0xc8` (left over from loading the local); `xor ebx, ebx` is what makes the exit status `0`. The `eax = 100` value from the function is also overwritten by `mov eax, 1` (the syscall number).
6. Differences from `02.asm` outside the stack: the exit uses `eax = 1` + `int 0x80` instead of `rax = 60` + `syscall`.
