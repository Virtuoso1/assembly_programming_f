# ADD Instruction: Flag Analysis


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF (Carry) | Cleared | CF tracks unsigned overflow. The unsigned sum 120 + 10 = 130 fits in 8 bits (max 255). No carry left bit 7. |
| OF (Overflow) | Set | OF tracks signed overflow. The signed 8-bit range is -128 to 127. Both operands are positive, but the true sum 130 is above 127. The result 1000 0010 has bit 7 set, so the CPU reads it as -126. Two positives gave a negative, so OF is set. |
| SF (Sign) | Set | SF copies bit 7 of the result. 0x82 is 1000 0010, so bit 7 is 1. |
| ZF (Zero) | Cleared | The result 0x82 is not zero. |
| PF (Parity) | Set | PF checks the low byte of the result. 1000 0010 has two 1-bits. Two is even, so PF is set. |
| AF (Auxiliary Carry) | Set | AF checks for a carry from bit 3 into bit 4. The low nibbles are 1000 (8) and 1010 (10). 8 + 10 = 18, which is above 15, so a carry moved into bit 4. |

### Key Point

This program shows CF and OF giving different answers for the same bits. As unsigned numbers, 120 + 10 = 130 is correct, so CF is cleared. As signed numbers, the answer -126 is wrong, so OF is set. The programmer picks which flag to check based on whether the data is signed or unsigned.

---

## Program 2: add2.asm (16-bit addition)



### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF (Carry) | Cleared | The unsigned sum 32000 + 500 = 32500 fits in 16 bits (max 65535). No carry left bit 15. |
| OF (Overflow) | Cleared | The signed 16-bit range is -32768 to 32767. Both operands are positive and the sum 32500 is below 32767. The result stays positive, so no signed overflow happened. |
| SF (Sign) | Cleared | SF copies bit 15 of the result. 0x7EF4 starts with 0111, so bit 15 is 0. |
| ZF (Zero) | Cleared | The result 0x7EF4 is not zero. |
| PF (Parity) | Cleared | PF checks only the low byte, 0xF4 = 1111 0100. It has five 1-bits. Five is odd, so PF is cleared. |
| AF (Auxiliary Carry) | Cleared | The low nibbles are 0000 (0) and 0100 (4). 0 + 4 = 4, which fits in 4 bits. No carry moved from bit 3 into bit 4. |

### Key Point

The sum 32500 is close to the signed limit of 32767 but stays under it. Adding 268 more would cross the limit and set OF. This program shows a clean addition with no carry, no overflow, and a positive result.

---

## Comparison

| Flag | add1 (120 + 10, 8-bit) | add2 (32000 + 500, 16-bit) |
|------|------------------------|----------------------------|
| CF | 0 | 0 |
| OF | 1 | 0 |
| SF | 1 | 0 |
| ZF | 0 | 0 |
| PF | 1 | 0 |
| AF | 1 | 0 |

Both programs add two positive numbers and neither sets CF. The difference is the register size. In add1, the 8-bit register pushes 130 past the signed limit of 127, so OF and SF are set. In add2, the 16-bit register has room for 32500, so OF and SF stay cleared.

IF (Interrupt Flag) appears in both GDB outputs. The operating system sets it, and `add` does not change it.
