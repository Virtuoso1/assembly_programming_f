# DIV Instruction: Flag Analysis

---

## Program 1: div1.asm (8-bit division)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF | Undefined | DIV does not define CF. |
| OF | Undefined | DIV does not define OF. |
| SF | Undefined | DIV does not define SF. |
| ZF | Undefined | DIV does not define ZF. |
| AF | Undefined | DIV does not define AF. |
| PF | Undefined | DIV does not define PF. |

### Explanation

An 8-bit `div` divides the 16-bit AX by an 8-bit operand. The quotient goes in AL and the remainder goes in AH. The quotient 14 fits in AL (max 255), so no divide error happens. The program runs to the exit call. The flags shown in GDB after this line are not the result of the division.

---

## Program 2: div2.asm (16-bit division)


### Flags

| Flag | Status | Reason |
|------|--------|--------|
| CF | Undefined | DIV does not define CF. |
| OF | Undefined | DIV does not define OF. |
| SF | Undefined | DIV does not define SF. |
| ZF | Undefined | DIV does not define ZF. |
| AF | Undefined | DIV does not define AF. |
| PF | Undefined | DIV does not define PF. |

### Explanation

A 16-bit `div` divides the 32-bit value DX:AX by a 16-bit operand. DX holds the upper 16 bits and AX holds the lower 16 bits. The quotient goes in AX and the remainder goes in DX.

The program sets DX to 0 before the division. This step matters. If DX held leftover data, the CPU would treat it as the upper half of the dividend. The dividend would become a much larger number, and the quotient would overflow AX and crash the program with SIGFPE.

The quotient 166 fits in AX (max 65535), so no divide error happens. As with Program 1, the flags shown in GDB after this line carry no meaning.

---

## Comparison

| | div1 | div2 |
|---|------|------|
| Operand size | 8-bit | 16-bit |
| Dividend | AX = 100 | DX:AX = 50000 |
| Divisor | BL = 7 | BX = 300 |
| Quotient | AL = 14 | AX = 166 |
| Remainder | AH = 2 | DX = 200 |
| Flags | All undefined | All undefined |
| Divide error | No | No |
