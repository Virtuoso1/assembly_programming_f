# SUB Instruction: Flag Analysis


## Program 1: sub1.asm (8-bit subtraction)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF (Carry) | Set | As unsigned numbers, 50 is smaller than 80. The CPU borrows from beyond bit 7 to complete the subtraction. CF records this borrow. The unsigned result 226 is wrong, and CF warns about it. |
| OF (Overflow) | Cleared | Both operands are positive. Subtracting two positives always gives a result between -127 and 127, which fits in the signed 8-bit range. The signed answer -30 is correct. |
| SF (Sign) | Set | Bit 7 of 1110 0010 is 1. The CPU reads the result as negative, which matches -30. |
| ZF (Zero) | Cleared | The result 0xE2 is not zero. The operands are not equal. |
| PF (Parity) | Set | The result 1110 0010 has four 1-bits. Four is even. |
| AF (Auxiliary Carry) | Cleared | The low nibbles are 0010 (2) and 0000 (0). 2 - 0 = 2, so no borrow from bit 4 was needed. |

### Explanation

This program shows CF and OF giving different answers. As unsigned numbers, 50 - 80 has no valid answer, so CF is set. As signed numbers, 50 - 80 = -30 is correct, so OF is cleared. A program treating the data as signed ignores CF and checks OF.

---

## Program 2: sub2.asm (16-bit subtraction)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF (Carry) | Set | As unsigned numbers, 1000 is smaller than 2000. The CPU borrows from beyond bit 15. The unsigned result 64536 is wrong, and CF records the borrow. |
| OF (Overflow) | Cleared | Both operands are positive. The signed answer -1000 fits in the signed 16-bit range of -32768 to 32767. |
| SF (Sign) | Set | Bit 15 of 0xFC18 is 1. The CPU reads the result as negative, which matches -1000. |
| ZF (Zero) | Cleared | The result 0xFC18 is not zero. |
| PF (Parity) | Set | PF checks only the low byte, 0x18 = 0001 1000. It has two 1-bits. Two is even. |
| AF (Auxiliary Carry) | Cleared | The low nibbles are 1000 (8) and 0000 (0). 8 - 0 = 8, so no borrow from bit 4 was needed. |

### Explanation

This program shows the same pattern as sub1 at 16 bits. The negative result -1000 is stored in two's complement form as 0xFC18. To check: 65536 - 1000 = 64536, which is 0xFC18. CF is set because of the unsigned borrow, and OF is cleared because the signed answer is valid.

---

## Comparison

| Flag | sub1 (50 - 80, 8-bit) | sub2 (1000 - 2000, 16-bit) |
|------|-----------------------|----------------------------|
| CF | 1 | 1 |
| OF | 0 | 0 |
| SF | 1 | 1 |
| ZF | 0 | 0 |
| PF | 1 | 1 |
| AF | 0 | 0 |

Both programs subtract a larger positive number from a smaller one. Both set CF because of the unsigned borrow, and both set SF because the result is negative. Neither sets OF because the signed result fits in the register. The register size changes the stored value but not the flag pattern.

IF (Interrupt Flag) appears in both GDB outputs. The operating system sets it, and `sub` does not change it.
