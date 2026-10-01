# MUL Instruction: Flag Analysis

`mul` performs unsigned multiplication. The result is twice the size of the operands, so the CPU stores it across two registers. The upper half goes in AH, DX, or EDX depending on the operand size.

MUL defines only two flags:

- CF and OF are both set if the upper half of the result is non-zero. This means the product did not fit in the lower half alone.
- CF and OF are both cleared if the upper half is zero. This means the full product fits in the lower half.

Intel lists SF, ZF, AF, and PF as undefined after `mul`. GDB still displays them, but their values carry no meaning.

---

## Program 1: mul1.asm (8-bit multiplication)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF | Cleared | AH is 0x00. The product 250 fits in AL alone (max 255). |
| OF | Cleared | MUL sets OF to the same value as CF. AH is 0x00, so OF is cleared. |
| SF | Undefined | MUL does not define SF. |
| ZF | Undefined | MUL does not define ZF. |
| AF | Undefined | MUL does not define AF. |
| PF | Undefined | MUL does not define PF. |

### Explanation

An 8-bit `mul` multiplies AL by an 8-bit operand and stores the 16-bit product in AX. AH holds the upper 8 bits and AL holds the lower 8 bits.

25 x 10 = 250. The largest unsigned 8-bit value is 255, so 250 fits in AL. AH stays 0x00, and the CPU clears CF and OF to signal the upper half is not needed.

---

## Program 2: mul2.asm (16-bit multiplication)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF | Set | DX is 0x0009, which is non-zero. The product 600000 does not fit in AX alone (max 65535). |
| OF | Set | MUL sets OF to the same value as CF. DX is non-zero, so OF is set. |
| SF | Undefined | MUL does not define SF. |
| ZF | Undefined | MUL does not define ZF. |
| AF | Undefined | MUL does not define AF. |
| PF | Undefined | MUL does not define PF. |

### Explanation

A 16-bit `mul` multiplies AX by a 16-bit operand and stores the 32-bit product in DX:AX. DX holds the upper 16 bits and AX holds the lower 16 bits.

3000 x 200 = 600000. The largest unsigned 16-bit value is 65535, so the product overflows AX. The extra bits spill into DX, which ends up as 0x0009. The CPU sets CF and OF to signal the program must read DX to get the full answer.

Reading AX alone gives 10176, which is wrong. The program stores both halves, AX at `result` and DX at `result+2`, so the 32-bit variable holds the correct value 600000.

---

## Comparison

| | mul1 | mul2 |
|---|------|------|
| Operand size | 8-bit | 16-bit |
| Product | 250 | 600000 |
| Lower half max | 255 | 65535 |
| Upper half | AH = 0x00 | DX = 0x0009 |
| CF | Cleared | Set |
| OF | Cleared | Set |
| SF, ZF, AF, PF | Undefined | Undefined |

The two programs show both states of CF and OF. In mul1, the product fits in the lower half, so both flags are cleared. In mul2, the product needs the upper half, so both flags are set.
