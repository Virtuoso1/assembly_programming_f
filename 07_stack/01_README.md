# 07_stack / 01.asm – Basic `push` and `pop` (64-bit)

## What the program does

```nasm
mov rax, 100
mov rbx, 200
push rax        ; save 100
push rbx        ; save 200
nop
pop rcx         ; rcx <- 200
pop rdx         ; rdx <- 100
nop
mov rax, 60     ; sys_exit
xor rdi, rdi    ; exit code 0
syscall
```

It loads two values, pushes both onto the stack, then pops them back into two *different* registers. The program prints nothing, so everything below was observed by single-stepping in `gdb`.

## Build, run, trace

```bash
nasm -f elf64 -g -F dwarf 01.asm -o 01.o
ld 01.o -o 01
./01 ; echo $?            

gdb ./01
(gdb) break _start
(gdb) run
(gdb) stepi               
(gdb) p/x $rsp
(gdb) x/4gx $rsp         
```

## Key facts about the stack

- The stack **grows downward**: `push` *subtracts* 8 from RSP and then writes; `pop` reads and then *adds* 8 to RSP.
- RSP always points at the **top** (the most recently pushed value, i.e. the lowest address in use).
- In 64-bit mode every push/pop moves exactly **8 bytes**.
- It is **LIFO**: the last value pushed is the first one popped.

## Execution walkthrough

When `_start` begins, the kernel has already built a stack: RSP = `0x7fffffff9370` and the 8 bytes there hold `argc = 1` (followed by the `argv` pointers). Our program never touches that memory; it only uses the space *below* it. (Absolute addresses will differ on your machine / outside gdb because of the environment size and ASLR, but the *differences* are always the same.)

| # | Instruction | RSP after | What happens on the stack |
|---|-------------|-----------|---------------------------|
| 1 | `mov rax, 100` | `...9370` | No stack use. RAX = 0x64. |
| 2 | `mov rbx, 200` | `...9370` | No stack use. RBX = 0xC8. |
| 3 | `push rax` | `...9368` | RSP -= 8, then `[0x...9368] = 0x64` (100). |
| 4 | `push rbx` | `...9360` | RSP -= 8, then `[0x...9360] = 0xC8` (200). |
| 5 | `nop` | `...9360` | Breakpoint position to inspect the stack (see below). |
| 6 | `pop rcx` | `...9368` | RCX = `[RSP]` = 0xC8 (200), then RSP += 8. |
| 7 | `pop rdx` | `...9370` | RDX = `[RSP]` = 0x64 (100), then RSP += 8. |
| 8 | `nop` | `...9370` | Stack is back to where it started. |
| 9-11 | `mov rax,60` / `xor rdi,rdi` / `syscall` | `...9370` | Exit syscall; the stack is not used. |

### Stack picture at the first `nop` (both values pushed)

```
 higher addresses
 0x7fffffff9378 | 0x00007fffffff9cb5 |   argv[0] pointer (from kernel)
 0x7fffffff9370 | 0x0000000000000001 |   argc            <- RSP at program start
 0x7fffffff9368 | 0x0000000000000064 |   pushed RAX (100)
 0x7fffffff9360 | 0x00000000000000c8 |   pushed RBX (200)  <- RSP now
 lower addresses
```

Net effect of the whole push/pop section: RSP went `9370 -> 9368 -> 9360 -> 9368 -> 9370`, i.e. two pushes (-16 bytes) were exactly balanced by two pops (+16 bytes).

## Observations from running it

1. **The values come out swapped.** After the pops, `RCX = 200` and `RDX = 100`, while the original registers were `RAX = 100`, `RBX = 200`. That is LIFO in action: 200 was pushed last, so it is popped first (into RCX). If the pops were `pop rbx` then `pop rax` (as in `05.asm`) the original registers would be restored; here the destination registers are different, so the order shows up in the results.
2. **`pop` does not erase memory.** After both pops, gdb still shows `0xc8` and `0x64` at `...9360` / `...9368`. They are now *below* RSP, so they are logically free and would be overwritten by the next push or `call`, but the data is physically still there.
3. **RSP is only changed by push/pop** in this program; the `mov`s and the `nop`s never move it, and RSP ends exactly where it began (`0x...9370`), which is what a balanced program should do.
4. `push rax` assembles to a single byte (`0x50`) and `push rbx` to `0x53`. Also, NASM encodes `mov rax, 100` as `mov eax, 0x64` (writing a 32-bit register zero-extends into the full 64-bit register), which gdb shows as `mov eax,0x64`; the result in RAX is identical.
5. The program exits with status `0` (`xor rdi, rdi` sets the exit code).
