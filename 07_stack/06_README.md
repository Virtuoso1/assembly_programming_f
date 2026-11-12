# 07_stack / 06.asm – Push/pop of 32-bit values and byte order (32-bit)

## What the program does

```nasm
BITS 32
_start:
    mov eax, 0x11223344
    mov ebx, 0x1234

    push eax            ; save EAX
    push ebx            ; save EBX
    nop

    pop ebx             ; restore EBX (reverse order)
    pop eax             ; restore EAX

    mov eax, 1          ; sys_exit (i386)
    xor ebx, ebx
    int 0x80
```

This is the 32-bit counterpart of `05.asm`: two values are pushed and popped back in reverse order, but every stack slot is **4 bytes** and the pointer is ESP.

## Build, run, trace

```bash
nasm -f elf32 -g -F dwarf 06.asm -o 06.o
ld -m elf_i386 06.o -o 06
./06 ; echo $?             # 0

gdb ./06
(gdb) break _start
(gdb) run
(gdb) stepi 4              
(gdb) p/x $esp
(gdb) x/2wx $esp           
(gdb) x/8xb $esp          
```

## Execution walkthrough

At `_start`: ESP = `0xffff8800` (holds `argc = 1`).

| # | Instruction | ESP after | What happens |
|---|-------------|-----------|--------------|
| 1 | `mov eax, 0x11223344` | `...8800` | EAX loaded. |
| 2 | `mov ebx, 0x1234` | `...8800` | EBX loaded. |
| 3 | `push eax` | `...87fc` | ESP -= **4**; `[...87fc] = 0x11223344`. |
| 4 | `push ebx` | `...87f8` | ESP -= 4; `[...87f8] = 0x00001234`. |
| 5 | `nop` | `...87f8` | Snapshot point (below). |
| 6 | `pop ebx` | `...87fc` | EBX = `[...87f8]` = 0x1234; ESP += 4. |
| 7 | `pop eax` | `...8800` | EAX = `[...87fc]` = 0x11223344; ESP += 4. |
| 8-10 | `mov eax,1` / `xor ebx,ebx` / `int 0x80` | `...8800` | Exit. |

### Stack at the `nop`

```
 higher addresses
 0xffff8800 | 0x00000001 |   argc                    <- ESP at start
 0xffff87fc | 0x11223344 |   pushed EAX
 0xffff87f8 | 0x00001234 |   pushed EBX              <- ESP now
 lower addresses
```

### The same memory as bytes (`x/8xb $esp`)

Offsets relative to ESP:

```
 ESP+0 : 34 12 00 00      <- EBX (0x00001234)
 ESP+4 : 44 33 22 11      <- EAX (0x11223344)
```

## Observations from running it

1. **4-byte slots.** Each `push` lowers ESP by 4 (`8800 -> 87fc -> 87f8`), and each `pop` raises it by 4. Compared with `05.asm`, where RSP moved by 8, the same pattern takes half the stack space.
2. **Little-endian again.** `0x11223344` is stored as `44 33 22 11` (lowest byte at the lowest address), and `0x1234` as `34 12 00 00`.
3. **LIFO restore.** EBX was pushed last, so it is popped first. After both pops, EAX = `0x11223344` and EBX = `0x1234` are exactly as before the pushes, and ESP is back at `0xffff8800`.
4. **Stale values remain in memory.** After the pops, gdb still shows `0x00001234` and `0x11223344` at `...87f8` / `...87fc`, below ESP, i.e. free but not erased.
5. **Registers at exit.** Before the syscall, `mov eax, 1` overwrites the restored EAX with the syscall number, and `xor ebx, ebx` clears EBX to give exit status `0`. Before the `xor`, gdb shows `ebx = 0x1234`, so without it the exit code would have been `0x34` (52). **Tested:** removing the `xor ebx, ebx` line and running gives exit status 52.
6. The default operand size of `push`/`pop` follows the mode: in this `BITS 32` program `push eax` is a 4-byte operation. In 64-bit code (`05.asm`) a 32-bit `push eax` is not encodable at all; you push the full `rax`.
