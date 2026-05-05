# Module List - CS202 SystemVerilog CPU Project

## Overview & Rationale for This Document

This document describes all SystemVerilog modules required for the 5-stage pipelined CPU.

---

## Core Pipeline Modules

### 1. cpu_top.sv (Top-Level Module)

**Purpose:** Instantiate and interconnect all pipeline stages, memories, control logic.

**Module Signature:**
\\\systemverilog
module cpu_top (
    input logic clk, reset,
    output logic [31:0] pc,           // For debugging
    output logic [31:0] debug_result  // For testbench
);
\\\

**Instantiates:** fetch_stage, decode_stage, execute_stage, memory_stage, writeback_stage, imem, dmem, forwarding_unit

**Key Responsibilities:**
- Connect pipeline registers between stages
- Route hazard signals (forwarding, stalls) to appropriate stages
- Manage global reset and clock distribution

---

### 2. fetch_stage.sv (IF Stage)

**Purpose:** Fetch instructions, manage program counter, handle jumps/branches.

**Module Signature:**
\\\systemverilog
module fetch_stage (
    input logic clk, reset,
    input logic [31:0] jump_target,   // From EX (JAL/JALR/branch)
    input logic jump_valid,            // High if jump taken
    input logic stall,                 // High to freeze PC
    output logic [31:0] pc,            // Current PC
    output logic [31:0] instr          // Fetched instruction (from IMEM)
);
\\\

**Key Logic:**
- PC increment: pc ← pc + 4 (or jump_target if jump_valid)
- IMEM synchronous read (1 cycle latency)
- Stall support (freeze PC when stall asserted)

**Testable Scenarios:**
- Sequential PC increment (PC += 4 each cycle)
- Jump taken (PC = jump_target)
- Stall (PC frozen for 1+ cycles)

---

### 3. if_id_reg.sv (IF/ID Pipeline Register)

**Purpose:** Hold IF stage outputs for ID stage consumption.

**Module Signature:**
\\\systemverilog
module if_id_reg (
    input logic clk, reset,
    input logic flush,                 // Clear register (on branch squash)
    input logic stall,                 // Hold register value
    input logic [31:0] instr_in, pc_in,
    output logic [31:0] instr_out, pc_out
);
\\\

---

### 4. decode_stage.sv (ID Stage)

**Purpose:** Decode instructions, read registers, generate control signals.

**Module Signature:**
\\\systemverilog
module decode_stage (
    input logic clk, reset,
    input logic [31:0] instr, pc,
    input logic [31:0] write_data,     // From WB stage
    input logic [4:0] write_addr,      // From WB stage
    input logic write_en,              // Write enable from WB
    
    // Control signals
    output logic [3:0] ctrl_alu_op,    // ALU operation select
    output logic ctrl_alu_sel_b,       // 0=register, 1=immediate
    output logic ctrl_mem_read, ctrl_mem_write,
    output logic [1:0] ctrl_wb_sel,    // 0=ALU, 1=MEM, 2=PC+4
    output logic ctrl_reg_write,
    output logic ctrl_branch, ctrl_jump, ctrl_jalr,
    
    // Data outputs
    output logic [31:0] rs1_data, rs2_data, imm,
    output logic [4:0] rd, rs1, rs2,
    output logic [2:0] funct3,
    output logic [6:0] funct7
);
\\\

**Key Tasks:**
- Extract fields: rd, rs1, rs2, opcode, funct3, funct7 from instr[31:0]
- Generate immediate: Sign-extend based on opcode (I/S/U/J/B types)
- Register file read: A = RF[rs1], B = RF[rs2]
- Control signal generation: Map opcode → control signals

**Immediate Types (SystemVerilog sign-extension):**
\\\systemverilog
logic [31:0] imm_i = {{20{instr[31]}}, instr[31:20]};           // I-type
logic [31:0] imm_s = {{20{instr[31]}}, instr[31:25], instr[11:7]}; // S-type
logic [31:0] imm_u = {instr[31:12], 12'b0};                      // U-type
logic [31:0] imm_j = {{12{instr[31]}}, instr[19:12], instr[20], instr[30:21], 1'b0}; // J-type
\\\

---

### 5. id_ex_reg.sv (ID/EX Pipeline Register)

**Purpose:** Hold ID stage outputs and control signals for EX stage.

**Module Signature:**
\\\systemverilog
module id_ex_reg (
    input logic clk, reset, flush, stall,
    input logic [31:0] rs1_data_in, rs2_data_in, imm_in, pc_in,
    input logic [4:0] rd_in, rs1_in, rs2_in,
    input logic [2:0] funct3_in,
    input logic [6:0] funct7_in,
    input logic [3:0] ctrl_alu_op_in,
    // ... all control signals (15+ signals)
    
    output logic [31:0] rs1_data_out, rs2_data_out, imm_out, pc_out,
    output logic [4:0] rd_out, rs1_out, rs2_out,
    output logic [2:0] funct3_out,
    output logic [6:0] funct7_out,
    output logic [3:0] ctrl_alu_op_out,
    // ... all control signals out
);
\\\

---

### 6. execute_stage.sv (EX Stage)

**Purpose:** Perform ALU operations, address computation, resolve branches/jumps.

**Module Signature:**
\\\systemverilog
module execute_stage (
    input logic clk, reset,
    input logic [31:0] rs1_data, rs2_data, imm, pc,
    input logic [4:0] rd, rs1, rs2,
    input logic [3:0] ctrl_alu_op,
    input logic ctrl_alu_sel_b, ctrl_jump, ctrl_jalr, ctrl_branch,
    // Forwarding inputs
    input logic [31:0] mem_stage_result, wb_stage_result,
    input logic [4:0] mem_stage_rd, wb_stage_rd,
    input logic mem_stage_valid, wb_stage_valid,
    
    output logic [31:0] alu_result, jump_target,
    output logic jump_taken,
    output logic [31:0] rs2_forwarded  // For store data
);
\\\

**Key Logic:**
- Data forwarding muxes: Select rs1/rs2 from register file or forwarded results
- ALU input selection: alu_b ← (ctrl_alu_sel_b) ? imm : rs2_forwarded
- Branch/jump resolution: Compute jump_target, assert jump_taken if branch taken

**Forwarding Rules (in priority order):**
1. Forward from EX/MEM if rd matches and write_en asserted
2. Forward from MEM/WB if rd matches and write_en asserted (and not already matched EX/MEM)
3. Use register file value otherwise

---

### 7. ex_mem_reg.sv (EX/MEM Pipeline Register)

**Purpose:** Hold EX stage results for MEM stage.

**Module Signature:**
\\\systemverilog
module ex_mem_reg (
    input logic clk, reset, flush, stall,
    input logic [31:0] alu_result_in, rs2_data_in,
    input logic [4:0] rd_in,
    input logic ctrl_mem_read_in, ctrl_mem_write_in,
    input logic [1:0] ctrl_wb_sel_in,
    
    output logic [31:0] alu_result_out, rs2_data_out,
    output logic [4:0] rd_out,
    output logic ctrl_mem_read_out, ctrl_mem_write_out,
    output logic [1:0] ctrl_wb_sel_out
);
\\\

---

### 8. memory_stage.sv (MEM Stage)

**Purpose:** Read/write data memory, handle byte/halfword/word loads and stores.

**Module Signature:**
\\\systemverilog
module memory_stage (
    input logic clk, reset,
    input logic [31:0] alu_result,   // Address
    input logic [31:0] rs2_data,     // Store data
    input logic ctrl_mem_read, ctrl_mem_write,
    input logic [2:0] funct3,        // For LB/LH/LW/SB/SH/SW distinction
    
    output logic [31:0] mem_data_out // Loaded data
);
\\\

**Key Tasks:**
- Byte enable generation: Map funct3 + address[1:0] → [3:0] byte_en
- DMEM interface: Read/write with byte enables
- Sign-extension for LB, LH (zero-extend for LBU, LHU handled in decode_stage)

---

### 9. mem_wb_reg.sv (MEM/WB Pipeline Register)

**Purpose:** Hold MEM stage results for WB stage.

**Module Signature:**
\\\systemverilog
module mem_wb_reg (
    input logic clk, reset,
    input logic [31:0] wb_data_in,
    input logic [4:0] rd_in,
    input logic ctrl_reg_write_in,
    
    output logic [31:0] wb_data_out,
    output logic [4:0] rd_out,
    output logic ctrl_reg_write_out
);
\\\

---

### 10. writeback_stage.sv (WB Stage)

**Purpose:** Select write-back data source, write registers.

**Module Signature:**
\\\systemverilog
module writeback_stage (
    input logic clk, reset,
    input logic [31:0] mem_data, alu_result,
    input logic [1:0] ctrl_wb_sel,   // 0=ALU, 1=MEM, 2=PC+4
    input logic [31:0] pc,
    input logic [4:0] rd,
    input logic ctrl_reg_write,
    
    output logic [31:0] write_data,
    output logic write_en,
    output logic [4:0] write_addr
);
\\\

**Write-Back Mux Logic:**
\\\systemverilog
always_comb
    case (ctrl_wb_sel)
        2'b00: write_data = alu_result;   // ALU result
        2'b01: write_data = mem_data;     // Memory data
        2'b10: write_data = pc + 32'd4;   // PC + 4 (for JAL/JALR)
        default: write_data = 32'b0;
    endcase
\\\

---

## Functional Units

### 11. register_file.sv (32 x 32-bit Registers)

**Purpose:** 2-read, 1-write port register storage (x0-x31).

**Module Signature:**
\\\systemverilog
module register_file (
    input logic clk, reset,
    input logic [4:0] rs1, rs2, rd,
    input logic [31:0] write_data,
    input logic write_en,
    
    output logic [31:0] data1, data2  // Asynchronous reads
);
\\\

**Implementation Notes:**
- Asynchronous read (combinational)
- Synchronous write (edge-triggered)
- x0 always reads 0; writes to x0 are ignored

---

### 12. alu.sv (32-bit ALU)

**Purpose:** Arithmetic and logic operations.

**Module Signature:**
\\\systemverilog
module alu (
    input logic [31:0] a, b,
    input logic [3:0] op,  // Operation select
    output logic [31:0] result
);
\\\

**Operation Encoding:**
- 0x0: ADD, 0x1: SUB, 0x2: AND, 0x3: OR, 0x4: XOR
- 0x5: SLL, 0x6: SRL, 0x7: SRA (arithmetic shift)
- 0x8: SLT (signed), 0x9: SLTU (unsigned)

---

### 13. control_unit.sv (Instruction Decoder)

**Purpose:** Map opcode/funct3/funct7 → control signals.

**Module Signature:**
\\\systemverilog
module control_unit (
    input logic [6:0] opcode,
    input logic [2:0] funct3,
    input logic [6:0] funct7,
    
    output logic [3:0] alu_op,
    output logic alu_sel_b, mem_read, mem_write,
    output logic [1:0] wb_sel,
    output logic reg_write, branch, jump, jalr
);
\\\

---

### 14. imem.sv (Instruction Memory - 32 KB)

**Purpose:** BRAM-based instruction ROM (0x00000000 - 0x00007FFF).

**Module Signature:**
\\\systemverilog
module imem (
    input logic [14:0] addr,   // Word address (0-8191)
    input logic clk,
    output logic [31:0] data   // Synchronous read output
);
\\\

**Implementation:** Initialize from hex file (\) during simulation.

---

### 15. dmem.sv (Data Memory - 32 KB)

**Purpose:** BRAM-based data memory (0x00008000 - 0x0000FFFF).

**Module Signature:**
\\\systemverilog
module dmem (
    input logic [14:0] addr,       // Word address
    input logic [31:0] write_data,
    input logic write_en,
    input logic [3:0] byte_en,     // Byte enables (for SB/SH/SW)
    input logic clk,
    output logic [31:0] read_data  // Synchronous read
);
\\\

**Byte Enable Logic:** 
\\\systemverilog
if (write_en) begin
    if (byte_en[0]) mem[addr][7:0]   <= write_data[7:0];
    if (byte_en[1]) mem[addr][15:8]  <= write_data[15:8];
    if (byte_en[2]) mem[addr][23:16] <= write_data[23:16];
    if (byte_en[3]) mem[addr][31:24] <= write_data[31:24];
end
\\\

---

## Special Function Modules

### 16. popcnt.sv (Population Count)

**Purpose:** Count 1-bits in lower 8 bits of input.

**Module Signature:**
\\\systemverilog
module popcnt (
    input logic [7:0] data_in,
    output logic [3:0] count_out
);
\\\

**Implementation:** \count_out = \(data_in);\

---

### 17. fclass.sv (IEEE-754 fp16 Classification)

**Purpose:** Classify 16-bit float: ±Zero (0), ±Infinity (1), NaN (2), Normalized (3), Denormalized (4).

**Module Signature:**
\\\systemverilog
module fclass (
    input logic [15:0] fp16_in,   // [sign][5-bit exp][10-bit frac]
    output logic [2:0] class_out  // 0-4
);
\\\

**Classification Logic:**
- exp=0, frac=0: ±Zero (0)
- exp=0, frac≠0: Denormalized (4)
- 0<exp<31: Normalized (3)
- exp=31, frac=0: ±Infinity (1)
- exp=31, frac≠0: NaN (2)

---

### 18. fquant.sv (Float → Q3.4 Quantization)

**Purpose:** Convert IEEE-754 fp16 to Q3.4 fixed-point (1 sign + 3 int + 4 frac bits).

**Module Signature:**
\\\systemverilog
module fquant (
    input logic [15:0] fp16_in,   // IEEE-754 fp16
    output logic [7:0] q34_out    // Q3.4 fixed-point (or two's complement for negative)
);
\\\

**Implementation Notes:**
- Extract mantissa, exponent, sign
- Convert to fixed-point with quantization (rounding)
- Output as 8-bit two's complement for negative values

---

### 19. forwarding_unit.sv (Data Forwarding & Hazard Detection)

**Purpose:** Generate forwarding control signals for EX stage data inputs.

**Module Signature:**
\\\systemverilog
module forwarding_unit (
    input logic [4:0] rs1, rs2,          // Current instruction source regs
    input logic [4:0] mem_rd, wb_rd,     // Previous instruction dest regs
    input logic mem_write_en, wb_write_en,
    
    output logic forward_rs1,            // 0=no forward, 1=from EX/MEM
    output logic forward_rs2,
    output logic forward_rs1_wb,         // 1=from MEM/WB
    output logic forward_rs2_wb
);
\\\

---

### 20. hazard_control.sv (Load-Use Stall Detection)

**Purpose:** Detect load-use data hazards, assert stall signal.

**Module Signature:**
\\\systemverilog
module hazard_control (
    input logic [4:0] id_rs1, id_rs2,    // Current ID stage source regs
    input logic [4:0] ex_rd,              // EX stage destination reg
    input logic ex_mem_read,              // EX stage is a load
    
    output logic stall_signal             // Asserts 1 if load-use hazard detected
);
\\\

**Stall Logic:**
- If (ex_mem_read) AND ((ex_rd == id_rs1) OR (ex_rd == id_rs2)) → stall_signal = 1

---

### 21. cpu_tb.sv (Testbench)

**Purpose:** Instantiate CPU, drive clock/reset, monitor outputs.

**Module Signature:**
\\\systemverilog
module cpu_tb;
    logic clk, reset;
    logic [31:0] pc, result;
    
    cpu_top cpu (.clk(clk), .reset(reset), .pc(pc), .debug_result(result));
    
    // Clock generation
    initial forever #5 clk = ~clk;
    
    // Test stimulus and monitoring...
endmodule
\\\

---

## Module Dependency & Integration Order

\\\
cpu_top (top-level)
  ├─ fetch_stage → if_id_reg → decode_stage
  │              ├─ register_file (2R1W)
  │              ├─ control_unit
  │              └─ imem (sync read)
  │
  ├─ id_ex_reg → execute_stage
  │              ├─ alu
  │              ├─ forwarding_unit (routes data)
  │              └─ hazard_control (generates stall)
  │
  ├─ ex_mem_reg → memory_stage
  │               └─ dmem (1R1W)
  │
  ├─ mem_wb_reg → writeback_stage
  │
  ├─ Special modules: popcnt.sv, fclass.sv, fquant.sv
  │   (integrated into execute or decode stages as needed)
  │
  └─ cpu_tb (testbench, not synthesized)
\\\

---

## Implementation Checklist

- [ ] ALU (all 10 operations)
- [ ] Register file (2R1W, x0 hardwired)
- [ ] Control unit (opcode → signals)
- [ ] Fetch stage (PC, IMEM read)
- [ ] Decode stage (fields, RF read, immediate generation)
- [ ] Execute stage (ALU, address calc, branch resolution)
- [ ] Memory stage (DMEM interface with byte enables)
- [ ] Write-back stage (MUX for ALU/MEM/PC+4)
- [ ] All 4 pipeline registers (IF/ID, ID/EX, EX/MEM, MEM/WB)
- [ ] Forwarding unit (EX/MEM and MEM/WB muxes)
- [ ] Hazard control (load-use detection)
- [ ] IMEM & DMEM (BRAM, sync read/write)
- [ ] Special instructions: POPCNT, FCLASS, FQUANT
- [ ] Testbench (stimulus, monitoring, result checking)
- [ ] Integration & system test

---

## Notes on Code Organization

**Why code sketches in this document?**

1. **Port definitions are reference material:** You'll flip back to these during implementation 50+ times.
2. **Signal naming conventions:** Established here ensure consistency (e.g., \ctrl_*\, \mem_*\, \ex_*\).
3. **Integration guide:** Knowing which module outputs connect to which inputs upfront reduces wiring errors.
4. **Not a replacement for implementation:** Full code goes in individual .sv files; this is the architecture blueprint.

Each module will have its own file with complete implementation, simulation setup, and unit tests.
