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
*** Begin Module List (categorized) ***

# Module List - CS202 SystemVerilog CPU Project

This document lists the planned SystemVerilog modules, grouped by whether they are shared between modes, specific to the single-cycle implementation, or specific to the 5-stage pipeline. Each entry includes the module purpose and a concise signature to keep integration consistent.

---

## Shared Modules (used by both SINGLE and PIPELINE)

- `register_file.sv` — 32 x 32-bit registers, 2R/1W
  - Signature:
    ```systemverilog
    module register_file(
        input logic clk, reset,
        input logic [4:0] rs1, rs2, rd,
        input logic [31:0] write_data,
        input logic write_en,
        output logic [31:0] data1, data2
    );
    ```

- `alu.sv` — 32-bit ALU (ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, SLT, SLTU)
  - Signature:
    ```systemverilog
    module alu(
        input logic [31:0] a, b,
        input logic [3:0] op,
        output logic [31:0] result
    );
    ```

- `imem.sv` — Parameterized instruction memory (sync read, init from hex)
  - Signature:
    ```systemverilog
    module imem #(
        parameter ADDR_WIDTH = 15
    )(
        input logic [ADDR_WIDTH-1:0] addr,
        input logic clk,
        output logic [31:0] data
    );
    ```

- `dmem.sv` — Parameterized data memory with byte-enable
  - Signature:
    ```systemverilog
    module dmem #(
        parameter ADDR_WIDTH = 15
    )(
        input logic [ADDR_WIDTH-1:0] addr,
        input logic [31:0] write_data,
        input logic write_en,
        input logic [3:0] byte_en,
        input logic clk,
        output logic [31:0] read_data
    );
    ```

- `control_unit.sv` — Instruction decoder → control signals
  - Signature:
    ```systemverilog
    module control_unit(
        input logic [6:0] opcode,
        input logic [2:0] funct3,
        input logic [6:0] funct7,
        output logic [3:0] alu_op,
        output logic alu_sel_b, mem_read, mem_write,
        output logic [1:0] wb_sel, reg_write, branch, jump, jalr
    );
    ```

- Special function modules (kept shared so either core can call them):
  - `popcnt.sv` — population count (`$countones` usage)
  - `fclass.sv` — fp16 classification
  - `fquant.sv` — fp16 → Q3.4 quantization

---

## Single-Cycle Specific Modules

- `cpu_single_cycle.sv` — Single-cycle top for functional validation (reuses shared modules)
  - Signature:
    ```systemverilog
    module cpu_single_cycle(
        input logic clk, reset,
        output logic [31:0] pc_out,
        output logic [31:0] debug_out
    );
    ```

Notes: reuses `imem`, `dmem`, `register_file`, `alu`, and special modules. Designed for fast functional simulation rather than FPGA timing closure.

---

## Pipeline-Specific Modules (5-stage)

- `cpu_pipeline.sv` — Top-level wrapper that instantiates pipeline stages
  - Signature:
    ```systemverilog
    module cpu_pipeline(
        input logic clk, reset,
        output logic [31:0] pc_out,
        output logic [31:0] debug_out
    );
    ```

- Pipeline stages and registers (file-per-module):
  - `fetch_stage.sv` — IF logic + PC management
  - `if_id_reg.sv` — IF/ID pipeline register
  - `decode_stage.sv` — ID logic, RF reads, control generation
  - `id_ex_reg.sv` — ID/EX pipeline register
  - `execute_stage.sv` — EX logic, ALU, branch resolution, forwarding inputs
  - `ex_mem_reg.sv` — EX/MEM pipeline register
  - `memory_stage.sv` — MEM logic, DMEM interface, byte enables
  - `mem_wb_reg.sv` — MEM/WB pipeline register
  - `writeback_stage.sv` — WB mux and RF write interface

- Forwarding & hazard units (support modules):
  - `forwarding_unit.sv` — generate forwarding control signals
  - `hazard_control.sv` — detect load-use hazards and assert stall

---

## Testbench and Aux

- `cpu_tb.sv` — top-level testbench that can instantiate `cpu_single_cycle` or `cpu_pipeline` via `cpu_top` or compile-time macro
- Helper scripts: `sim_single.sh/.ps1`, `sim_pipeline.sh/.ps1` (invoke iverilog/vvp)

---

## Integration Notes

- `cpu_top.sv` will be the project top: it latches mode at reset (`mode_select`) and instantiates either `cpu_single_cycle` or `cpu_pipeline` while sharing `imem`, `dmem`, and `register_file` instances to minimize duplication.
- Keep interface names consistent across modules: `ctrl_*`, `mem_*`, `pc`, `instr`, `rs1_data`, `rs2_data`, `alu_result`, `wb_data`.

---

## Implementation Checklist (categorized)

- Shared: `alu`, `register_file`, `imem`, `dmem`, `control_unit`, `popcnt`, `fclass`, `fquant`
- Single-cycle: `cpu_single_cycle`, simple test programs
- Pipeline: all pipeline stages, forwarding_unit, hazard_control, pipeline registers
- Test: `cpu_tb.sv`, sim scripts, sample `program.hex`

*** End Module List ***
