# 07_stack / 05.asm – Push/pop of full 64-bit values and byte order (64-bit)

## What the program does

```nasm
mov rax, 0x1122334455667788
mov rbx, 0x1234

push rax            ; save RAX
push rbx            ; save RBX
nop

pop rbx             ; restore RBX (in reverse order)
pop rax             ; restore RAX

mov rax, 60
xor rdi, rdi
syscall
```

The program pushes a large 64-bit constant and a small one, then pops them back **in reverse order** into the same registers. It demonstrates that each slot is a full 8 bytes, how the bytes are laid out in memory (little-endian), and that correct LIFO ordering restores the original registers.

## Build, run, trace

```bash
nasm -f elf64 -g -F dwarf 05.asm -o 05.o
ld 05.o -o 05
./05 ; echo $?             

gdb ./05
(gdb) break _start
(gdb) run
(gdb) stepi 4              
(gdb) p/x $rsp
(gdb) x/2gx $rsp          
(gdb) x/16xb $rsp          
```

## Execution walkthrough

At `_start`: RSP = `0x7fffffff9370` (holds `argc`).

| # | Instruction | RSP after | What happens |
|---|-------------|-----------|--------------|
| 1 | `mov rax, 0x1122334455667788` | `...9370` | RAX loaded (assembled as `movabs`, a 10-byte instruction). |
| 2 | `mov rbx, 0x1234` | `...9370` | RBX = 0x1234. |
| 3 | `push rax` | `...9368` | RSP -= 8; `[...9368] = 0x1122334455667788`. |
| 4 | `push rbx` | `...9360` | RSP -= 8; `[...9360] = 0x0000000000001234`. |
| 5 | `nop` | `...9360` | Snapshot point (below). |
| 6 | `pop rbx` | `...9368` | RBX = `[...9360]` = 0x1234; RSP += 8. |
| 7 | `pop rax` | `...9370` | RAX = `[...9368]` = 0x1122334455667788; RSP += 8. |
| 8-10 | `mov rax,60` / `xor rdi,rdi` / `syscall` | `...9370` | Exit. |

### Stack at the `nop`, as 64-bit values

```
 higher addresses
 0x7fffffff9370 | 0x0000000000000001 |   argc                      <- RSP at start
 0x7fffffff9368 | 0x1122334455667788 |   pushed RAX
 0x7fffffff9360 | 0x0000000000001234 |   pushed RBX                <- RSP now
 lower addresses
```

### The same memory as bytes (`x/16xb $rsp`)

Offsets are relative to RSP (the absolute address changes between runs):

```
 RSP+0  : 34 12 00 00 00 00 00 00      <- RBX (0x1234), 8 bytes
 RSP+8  : 88 77 66 55 44 33 22 11      <- RAX (0x1122334455667788), 8 bytes
```

## Observations from running it

1. **Little-endian layout.** The *least* significant byte is stored at the *lowest* address, so `0x1122334455667788` appears in memory as `88 77 66 55 44 33 22 11`, and `0x1234` as `34 12 00 ...`. gdb's `x/gx` shows the logical value; `x/xb` shows the real byte order.
2. **Each push uses a full 8 bytes** regardless of how small the value is: RBX held only `0x1234`, but a whole quadword (padded with zeros) was written and RSP still moved by 8.
3. **Pop order mirrors push order.** `pop rbx` first because RBX was pushed last. After the pops, RBX is `0x1234` and RAX is `0x1122334455667788` again. If the pops were swapped (`pop rax` first), RAX would have received `0x1234` and RBX the big constant. This is the correct "save and restore" pattern, and it is the contrast with `01.asm`, where the destinations differed.
4. **Nothing visibly changes in the registers** (they are restored to what they were), so the only evidence is RSP: `9370 -> 9368 -> 9360 -> 9368 -> 9370`, which ends balanced.
5. The pushed data is **still in memory below RSP** after the pops (gdb still shows `0x1234` and `0x1122334455667788` at `...9360` / `...9368`), just not protected.
6. The program exits with status `0`. A side observation is that at exit RAX holds `0x3c` (60), overwriting the restored value, as it is reused for the syscall number.
