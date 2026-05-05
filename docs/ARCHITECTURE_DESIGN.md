# Architecture Design - CS202 SystemVerilog CPU Project

## 5-Stage Pipeline Overview

This CPU implements a classic 5-stage RISC pipeline:

\\\
    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
    │ IF   │→ │ ID   │→ │ EX   │→ │ MEM  │→ │ WB   │
    └──────┘  └──────┘  └──────┘  └──────┘  └──────┘
     Fetch     Decode   Execute   Memory    Writeback
\\\

### Pipeline Stages

**1. Instruction Fetch (IF)**
- Read instruction from IMEM at address PC
- Increment PC (or jump target on JAL/JALR)
- Output: Instruction word (instr), updated PC

**2. Instruction Decode (ID)**
- Decode opcode, extract fields (rs1, rs2, rd, imm, funct3, funct7)
- Read register file: A = rf[rs1], B = rf[rs2]
- Generate control signals: ALU operation, memory access type, write-back control
- Output: A, B, imm, rd, control signals

**3. Execute (EX)**
- ALU operations: ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, SLT, SLTU
- Address computation: (rs1 + imm) for branches and load/store
- PC-relative calculation: PC + imm for JAL/AUIPC
- Output: ALU result, memory address, branch condition status

**4. Memory Access (MEM)**
- Load: Read from DMEM at computed address
- Store: Write to DMEM at computed address
- Pass-through: For instructions that don't access memory
- Output: Memory data (for load) or ALU result (for non-mem ops)

**5. Write-Back (WB)**
- Select write-back data (ALU result or memory data)
- Write to register file at rd (if not x0)
- Output: None (terminal stage)

---

## Pipeline Registers

Pipeline registers hold data between stages. They are updated on every clock cycle.

### IF/ID Pipeline Register
`
- instr[31:0]     : Instruction word
- pc[31:0]        : Current PC (for AUIPC, branch offset calculation)
`

### ID/EX Pipeline Register
`
- a[31:0]         : Value from rs1 (register file read)
- b[31:0]         : Value from rs2 (register file read)
- imm[31:0]       : Sign-extended immediate
- rd[4:0]         : Destination register address
- pc[31:0]        : PC from IF/ID stage
- funct3[2:0]     : For ALU operation selection
- funct7[6:0]     : For ALU operation selection (SRA vs SRL)
- ctrl_alu_op[3:0]: Control signal for ALU
- ctrl_mem_read   : Is this a load instruction?
- ctrl_mem_write  : Is this a store instruction?
- ctrl_wb_sel     : Write-back selector (ALU vs MEM data)
- ctrl_branch     : Is this a branch/JAL instruction?
`

### EX/MEM Pipeline Register
`
- alu_result[31:0]: Result from ALU
- mem_addr[31:0]  : Address for memory access
- b[31:0]         : Data to write (for store)
- rd[4:0]         : Destination register address
- pc_next[31:0]   : Next PC (for JAL/branch resolution)
- ctrl_mem_read   : Is this a load instruction?
- ctrl_mem_write  : Is this a store instruction?
- ctrl_wb_sel     : Write-back selector
`

### MEM/WB Pipeline Register
`
- wb_data[31:0]   : Data to write to register file
- rd[4:0]         : Destination register address
`

---

## Data Path Components

### Instruction Memory (IMEM)
- Size: 32 KB (0x0000_0000 - 0x0000_7FFF)
- Interface: 1 read port, no write (ROM at runtime)
- Synchronous read (latency 1 cycle typical with BRAM)

### Data Memory (DMEM)
- Size: 32 KB (0x0000_8000 - 0x0000_FFFF)
- Interface: 1 read port, 1 write port
- Supports byte, halfword, word writes
- Synchronous read/write

### Register File (RF)
- 32 x 32-bit registers (x0-x31)
- 2 read ports (rs1, rs2)
- 1 write port (rd)
- x0 always reads 0 and ignores writes
- Asynchronous read, synchronous write

### ALU
- Inputs: A (32-bit), B (32-bit), operation select (funct3 + funct7)
- Operations:
  - ADD: A + B
  - SUB: A - B
  - AND: A & B
  - OR: A | B
  - XOR: A ^ B
  - SLL: A << B[4:0]
  - SRL: A >> B[4:0] (logical)
  - SRA: A >>> B[4:0] (arithmetic)
  - SLT: (A < B signed) ? 1 : 0
  - SLTU: (A < B unsigned) ? 1 : 0
  - COPY_A: A (for load/store address computation in parallel)
- Output: Result (32-bit), flags (zero, overflow, negative, etc.)

### Program Counter (PC)
- Tracks next instruction address
- Sources for next PC:
  1. PC + 4 (normal sequential)
  2. PC + immediate (branch/JAL)
  3. (rs1 + immediate) & ~1 (JALR)
- Updated at IF stage on every cycle

---

## Hazard Management

### Data Hazards
**Problem:** A later instruction depends on a result not yet written.

**Example:**
`
ADD x1, x2, x3    # x1 = x2 + x3 (result in WB at cycle 5)
ADD x4, x1, x5    # x4 = x1 + x5 (reads x1 at ID in cycle 2)
→ Data hazard: x4 reads stale value of x1
`

**Solution: Forwarding (Bypassing)**
- From EX/MEM to EX: If previous instr writes rd, forward ALU result to current EX input
- From MEM/WB to EX: If 2-stage-ago instr writes rd, forward WB data to current EX input
- Special case: Loads cannot be fully forwarded (memory latency unavoidable)

**Load-Use Hazard (not fully avoidable):**
`
LW x1, 0(x2)      # x1 = MEM[x2] (result in WB at cycle 5)
ADD x3, x1, x4    # Reads x1 at ID in cycle 2 → stall needed
`
- **Mitigation:** 1-cycle stall (insert bubble in pipeline), or software reorder

### Control Hazards
**Problem:** Fetching wrong instruction after branch/jump.

**Example:**
`
BEQ x1, x2, target  # Branch resolution in EX stage (cycle 3)
ADD x3, x4, x5      # Fetched speculatively, may be wrong
→ Control hazard: Fetch may have gone to wrong address
`

**Solution: Branch Prediction**
- **Simple strategy (assume not-taken):** Always fetch PC+4, only squash on actual branch taken
- **Squashing:** On branch resolved in EX, invalidate IF/ID and ID/EX stages, restart with correct PC

---

## Control Unit

The control unit decodes the instruction and generates control signals for each stage.

### Control Signal Generation (ID Stage)

From opcode, funct3, funct7:

`
ctrl_alu_op[3:0]    ← Selects ALU operation
ctrl_alu_sel_b[0]   ← Select B or immediate for ALU (0=register, 1=immediate)
ctrl_mem_read[0]    ← Load instruction?
ctrl_mem_write[0]   ← Store instruction?
ctrl_wb_sel[1:0]    ← Write-back source (0=ALU, 1=MEM, 2=PC+4, 3=unused)
ctrl_reg_write[0]   ← Write to register file?
ctrl_branch[0]      ← Is branch instruction?
ctrl_jump[0]        ← Is jump instruction (JAL)?
ctrl_jalr[0]        ← Is jump register instruction (JALR)?
`

### Instruction Decode Example (AND x3, x1, x2)
- Opcode: 0110011 → R-type
- funct3: 111, funct7: 0000000 → AND operation
- Control signals:
  - ctrl_alu_op = AND
  - ctrl_alu_sel_b = 0 (use register)
  - ctrl_mem_read = 0
  - ctrl_mem_write = 0
  - ctrl_wb_sel = 0 (ALU result)
  - ctrl_reg_write = 1
  - ctrl_branch = 0
  - ctrl_jump = 0

---

## Exception Handling (Minimal)

For this course project, only basic exceptions:

- **ECALL:** Halt simulation (used in testbenches)
- **Illegal instruction:** Treat as NOP or trap (optional)

Full interrupt support deferred to future extensions.

---

## Summary Table: Stage Responsibilities

| Stage | Input                      | Processing                               | Output                   |
| ----- | -------------------------- | ---------------------------------------- | ------------------------ |
| IF    | PC                         | Fetch instr from IMEM, increment PC      | instr, pc                |
| ID    | instr, rf[]                | Decode, read registers, gen ctrl signals | a, b, imm, rd, ctrl      |
| EX    | a, b, imm, pc, ctrl        | ALU compute, address calc                | alu_result, branch_taken |
| MEM   | alu_result, b, ctrl        | Mem read/write                           | mem_data or alu_result   |
| WB    | mem_data or alu_result, rd | Write register file                      | (none)                   |

---

## Integration Points with Modules (See MODULE_LIST.md)

- **cpu_top:** Top-level instantiation of all stages and memories
- **fetch_stage:** IF logic and PC management
- **decode_stage:** Instruction decoder and register file
- **execute_stage:** ALU and address computation
- **memory_stage:** DMEM interface and load/store logic
- **writeback_stage:** Register file write logic
- **register_file:** 2R1W synchronous register storage
- **alu:** Arithmetic and logic unit
- **imem:** 32 KB instruction BRAM
- **dmem:** 32 KB data BRAM
- **control_unit:** Instruction decoder (generates control signals)

