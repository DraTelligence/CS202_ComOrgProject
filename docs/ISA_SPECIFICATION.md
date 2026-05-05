# ISA Specification - CS202 SystemVerilog CPU Project

## Overview

This CPU implements a **modified RISC-V subset** optimized for the Xilinx XC7A35T FPGA. The ISA is designed to be simple yet powerful enough for course-level understanding and the required computational tasks.

**Hardware Target:** Xilinx Artix-7 XC7A35T-1CSG324  
**Word Size:** 32-bit  
**Register File:** 32 registers (x0-x31)  
**Pipeline Depth:** 5 stages (IF-ID-EX-MEM-WB)  
**Language:** SystemVerilog (IEEE 1800-2017)

---

## Registers

| Name   | Number | Purpose             | Notes                     |
| ------ | ------ | ------------------- | ------------------------- |
| x0     | 0      | Zero register       | Always 0 (writes ignored) |
| x1     | 1      | Return address (ra) | For JAL/JALR              |
| x2     | 2      | Stack pointer (sp)  | Convention only           |
| x3-x31 | 3-31   | General purpose     | Free to use               |
| PC     | -      | Program counter     | 32-bit address            |

---

## Instruction Formats

All instructions are **32-bit** fixed-length:

### R-Type: Register-Register Operations
`
[31:25] funct7
[24:20] rs2 (second source register)
[19:15] rs1 (first source register)
[14:12] funct3
[11:7]  rd (destination register)
[6:0]   opcode (0110011 for R-type)
`

### I-Type: Immediate Operations
`
[31:20] imm[11:0] (sign-extended)
[19:15] rs1 (source register)
[14:12] funct3
[11:7]  rd (destination register)
[6:0]   opcode
`

### U-Type: Load Upper Immediate
`
[31:12] imm[31:12]
[11:7]  rd (destination register)
[6:0]   opcode (0110111 for LUI)
`

### J-Type: Jump and Link
`
[31:20] imm[20|10:1]
[11:7]  rd (link register)
[6:0]   opcode (1101111 for JAL)
`

---

## Base Instructions

**Arithmetic (R-Type):**
- ADD x3, x1, x2 → rd = rs1 + rs2
- SUB x3, x1, x2 → rd = rs1 - rs2
- AND x3, x1, x2 → rd = rs1 & rs2 (Required)
- OR x3, x1, x2 → rd = rs1 | rs2
- XOR x3, x1, x2 → rd = rs1 ^ rs2
- SLL x3, x1, x2 → rd = rs1 << rs2[4:0] (Required: shift amount in low 5 bits)
- SRL x3, x1, x2 → rd = rs1 >> rs2[4:0]
- SRA x3, x1, x2 → rd = rs1 >>> rs2[4:0] (Required: arithmetic right shift)

**Immediate (I-Type):**
- ADDI, ANDI, ORI, XORI, SLLI, SRLI, SRAI

**Load/Store:**
- LW, LH, LB, SW, SH, SB (word/halfword/byte)

**Control Flow:**
- JAL x1, offset → rd = PC+4; PC = PC+offset (Required)
- JALR x1, offset(x2) → rd = PC+4; PC = (rs1+offset)&~1 (Required)
- BEQ, BNE, BLT, BGE, BLTU, BGEU (branch comparisons)

**Upper Immediate:**
- LUI x1, 0x12345 → rd[31:12] = imm, rd[11:0] = 0 (Required)
- AUIPC x1, 0x12345 → rd = PC + (imm<<12) (Required)

---

## Special Instructions (Custom)

**Bit Count:** POPCNT x2, x1 → x2 = count of 1-bits in x1[7:0]

**IEEE-754 fp16 Classification:** FCLASS x2, x1 → x2 = classification code (0-4)
- 0: ±Zero
- 1: ±Infinity  
- 2: NaN
- 3: Normalized
- 4: Denormalized

**IEEE-754 fp16 → Q3.4 Quantization:** FQUANT x2, x1 → x2[7:0] = Q3.4 fixed-point

**System:** ECALL → Halt simulation

---

## Memory Layout

| Address                   | Size  | Purpose                   |
| ------------------------- | ----- | ------------------------- |
| 0x0000_0000 - 0x0000_7FFF | 32 KB | Instruction memory (IMEM) |
| 0x0000_8000 - 0x0000_FFFF | 32 KB | Data memory (DMEM)        |

**Total BRAM: 64 KB** (XC7A35T limit ~100 KB leaves room for registers, logic)

---

## Instruction Encoding Reference

**Opcode Summary:**
- R-type (register): 0110011
- I-type (immediate): 0010011
- S-type (store): 0100011
- B-type (branch): 1100011
- U-type (upper imm): 0110111
- J-type (JAL): 1101111
- JALR: 1100111
- LUI: 0110111
- AUIPC: 0010111

See ISA manual for complete encoding reference.
