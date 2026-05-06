# Architecture Design — Dual-Mode CPU (SystemVerilog)

This document extends the original 5-stage pipeline description to a dual-mode CPU design that supports both a functional single-cycle core and an optimized 5-stage pipeline core. The goal is to enable fast functional verification (single-cycle) and a performance-oriented implementation (pipeline) while minimizing duplicated logic via shared modules.

## Design Overview (Dual Mode)

- Modes: `SINGLE` (single-cycle) and `PIPELINE` (5-stage pipeline).
- The top-level `cpu_top` latches a `mode` input during reset and instantiates either `cpu_single_cycle` or `cpu_pipeline`. Shared modules such as `imem`, `dmem`, `register_file`, `alu`, and special-function units are reused by both cores where possible.
- Shared modules: `alu`, `register_file`, `control_unit`, `imem`, `dmem`, `popcnt`, `fclass`, `fquant`.

Design goals: reduce duplicated code through parameterization and shared instances, enable quick functional checks with the single-cycle core, and incrementally add pipeline features (forwarding, stalls) to the pipeline core.
---


## 5-Stage Pipeline (Recap)

- IF: instruction fetch and PC update (PC+4 / branch / JALR)
- ID: instruction decode, immediate generation, register reads, control signal generation
- EX: ALU operations, address computation, branch evaluation and target calculation
- MEM: data memory access (byte/half/word with byte enables), load sign/zero extension
- WB: write-back to register file (ALU result / memory / PC+4)

Key pipeline features: pipeline registers (IF/ID, ID/EX, EX/MEM, MEM/WB), forwarding from EX/MEM and MEM/WB into EX, a 1-cycle load-use stall, and a simple not-taken branch policy with flush on taken.

---


## Single-Cycle Core (Overview)

- `cpu_single_cycle` implements IF→ID→EX→MEM→WB in a single clock cycle and is intended for quick functional verification and unit testing.
- Note: the single-cycle core has long combinational paths and is not intended for high-frequency FPGA synthesis; use it primarily for simulation and correctness checks.
- The single-cycle core should reuse shared modules (`alu`, `register_file`, `imem`, `dmem`, `control_unit`, special modules).

Pros: straightforward to implement and debug. Cons: not suitable for timing-constrained FPGA builds.

---


## Top-Level `cpu_top` (Mode Selection)

Suggested top-level interface:

```systemverilog
module cpu_top (
  input logic clk, reset,
  input logic mode_select, // 0 = SINGLE, 1 = PIPELINE (latched at reset)
  output logic [31:0] pc,
  output logic [31:0] debug_result
);
```

Design notes:
- `mode_select` is latched during reset; after reset the selected core is active until the next reset (no runtime hot-swap).
- `imem`, `dmem`, and `register_file` are parameterized for address/depth and can be shared between the two cores to reduce duplication.
- Top-level selects outputs (e.g., `pc`, `debug_result`) from the active core via multiplexers.

Avoid runtime mode switching to prevent state synchronization complexities.

---


## Memory Parameterization (IMEM / DMEM)

- Use parameters such as `ADDR_WIDTH` or `DEPTH` in `imem` and `dmem` to allow compile/synthesis-time size configuration. Example:

```systemverilog
module imem #(
  parameter ADDR_WIDTH = 15
) (
  input logic [ADDR_WIDTH-1:0] addr,
  input logic clk,
  output logic [31:0] data
);
```

- The top-level can choose sizes for different builds. For simulation, a single maximum-size memory can be used and initialized with small programs.

Benefits: avoids duplicating memory modules per mode and eases resource planning for the FPGA target.

---


## Special Instruction & Floating-Point Strategy

Special instructions (`POPCNT`, `FCLASS`, `FQUANT`) should be implemented as separate modules and invoked by the EX stage under control signals generated in decode:

- Advantages: the integer datapath remains 32-bit simple; special/FP logic can be tested and optimized separately.
- Implementation: decode detects custom opcodes and asserts `ctrl_special`; the EX stage routes operands to a special-function unit (e.g., `popcnt`) and consumes the result as ALU output or write-back input.
- For full IEEE floating-point support, plan a multi-cycle FPU module later and add handshake/issue logic to integrate it with the pipeline.

Conclusion: keep the integer core clean and use separate modules for teaching-focused FP/quantization features.

---


## Impact and Required Changes

Suggested changes without altering the ISA semantics:

1. Ensure `docs` reflect the dual-mode design (this file).
2. Update `MODULE_LIST.md` to categorize modules (shared / single-cycle / pipeline) — done in parallel.
3. Make `imem`/`dmem` parameterized or use a single maximum-sized memory with program initialization files.
4. Add `cpu_single_cycle.sv` skeleton reusing shared units for quick verification.
5. Implement `cpu_top.sv` that latches `mode_select` at reset and multiplexes top-level outputs.

These changes are structural and prioritize parameterization and top-level selection logic to minimize duplicated code.

---


## Testing & Validation Recommendations

- Start with the single-cycle core to validate ISA behavior (testcases: Fibonacci, popcount, load/store patterns).
- For the pipeline core, incrementally add pipeline registers, then forwarding, and finally hazard/stall logic. Validate load-use stalls and branch flush semantics.
- Provide two simulation entry points or macros: `sim_single` and `sim_pipeline`.

Example commands (iverilog + vvp):

```powershell
# Single-cycle simulation
iverilog -g2012 -o sim_single rtl/*.sv tb/cpu_tb.sv -DCPU_MODE_SINGLE
vvp sim_single

# Pipeline simulation
iverilog -g2012 -o sim_pipeline rtl/*.sv tb/cpu_tb.sv -DCPU_MODE_PIPELINE
vvp sim_pipeline
```

(`-D` macros may be used to select modes at compile time or use `cpu_top`'s `mode_select` port.)

---


## Summary

- Do not rewrite the ISA; focus on top-level `cpu_top`, memory parameterization, and a `cpu_single_cycle` skeleton that reuses shared modules.
- Latch mode at reset (no hot-swap) to keep implementation and verification simple.
- Keep the integer datapath simple; add special-function FP/quant modules independently. A full multi-cycle FPU is a future enhancement.

Next steps I can take now:
1. Update `MODULE_LIST.md` to explicitly categorize modules (shared / single-cycle / pipeline).  
2. Generate `rtl/` templates: `cpu_top.sv`, `cpu_single_cycle.sv`, parameterized `imem.sv` and `dmem.sv`.
Please confirm which to prioritize.
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

