# Development Roadmap - CS202 SystemVerilog CPU Project

## Hybrid Incremental Approach (4-Week Intensive Sprint)

This roadmap follows a **hybrid incremental development strategy**: Define the ISA and architecture first, then implement the 5-stage pipeline in 4 intense weeks, with each week producing testable, incrementally functional results.

**Total Duration:** 4 weeks (intensive: 20-25 hours/week per student)  
**Language:** SystemVerilog (IEEE 1800-2017)  
**Simulator:** VCS/Questa or IVerilog  
**Synthesis Target:** Xilinx XC7A35T-1CSG324 (optional for final testing)

---

## Phase 1: Foundation & Building Blocks (Week 1)

### Goals
- Establish development environment and project structure
- Implement core functional units (ALU, register file, decoder, memories)
- Create testbench framework
- Unit-test each module independently

### Deliverables
- SystemVerilog modules: alu.sv, register_file.sv, control_unit.sv, imem.sv, dmem.sv
- Testbench framework (cpu_tb.sv)
- Build/simulation scripts (Makefile or shell scripts)

### Key Tasks
- [ ] Setup SystemVerilog environment (simulator choice: VCS, Questa, or IVerilog+GTKWave)
- [ ] Implement ALU.sv: ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, SLT, SLTU
- [ ] Implement register_file.sv: 32x32-bit, 2R1W, x0 hardwired to 0
- [ ] Implement control_unit.sv: Decode opcode, funct3, funct7 → control signals
- [ ] Implement imem.sv: 32 KB BRAM, synchronous read (word-addressed)
- [ ] Implement dmem.sv: 32 KB BRAM, 1R1W with byte-enable logic
- [ ] Unit tests for each module (directed tests, not exhaustive)
- [ ] Verify compilation and basic functionality

### Test Coverage
`systemverilog
// ALU tests
ALU: AND(0xFF00, 0x00FF) → 0x0000 ✓
ALU: SLL(0x0001, 5) → 0x0020 ✓
ALU: SRA(0x80000000, 1) → 0xC0000000 ✓

// Register file tests
RF: Write x5=0x12345678, Read x5 → 0x12345678 ✓
RF: Write x0=0xFFFFFFFF, Read x0 → 0x00000000 ✓ (x0 read-only)
`

### Cumulative Progress
**~800 LOC**: All building blocks functional and tested

---

## Phase 2: Single-Cycle CPU & Memory Operations (Week 2)

### Goals
- Assemble blocks into functional single-cycle CPU
- Implement all load/store instructions with proper memory interface
- Verify data path with Fibonacci and bit-counting programs
- Refactor modules for easy conversion to pipelined design

### Deliverables
- cpu_top.sv: Single-cycle CPU top-level
- Complete instruction set working (base + load/store)
- Test programs: Fibonacci F(20), bit-count tests
- Refactored fetch/decode/execute/memory/writeback logic

### Key Tasks
- [ ] Assemble cpu_top.sv: Instantiate ALU, RegFile, IMEM, DMEM, control unit
- [ ] Implement instruction fetch: PC increment every cycle
- [ ] Implement immediate sign-extension (I/S/U/J/B types)
- [ ] Implement ALU input multiplexer (register vs immediate)
- [ ] Implement writeback multiplexer (ALU / MEM data / PC+4)
- [ ] Implement DMEM write-enable logic (byte/halfword/word selects)
- [ ] Implement load sign-extension (LB sign→32bit, LH, LW, etc.)
- [ ] Test all instruction types: R/I/S/U/J/B formats
- [ ] Run Fibonacci test: Compute F(20) correctly
- [ ] Run bit-count test: POPCNT sequence on known values

### Test Programs
`ssembly
# Fibonacci: F(n) where input n in x5, output in x1
ADDI x5, x0, 20          # n = 20
ADDI x1, x0, 0           # a = 0
ADDI x2, x0, 1           # b = 1
ADDI x3, x0, 0           # counter = 0

fib_loop:
  BEQ x3, x5, fib_end    # if counter == n, exit
  ADD x4, x1, x2         # temp = a + b
  ADD x1, x0, x2         # a = b
  ADD x2, x0, x4         # b = temp
  ADDI x3, x3, 1         # counter++
  JAL x0, fib_loop       # Loop (using unconditional JAL)

fib_end:
  # Result in x1 (should be 6765 for F(20))
`

### Cumulative Progress
**~1200 LOC**: Fully functional single-cycle CPU with all base + memory instructions

---

## Phase 3: 5-Stage Pipeline & Hazard Control (Week 3)

### Goals
- Refactor single-cycle into 5-stage pipeline (IF-ID-EX-MEM-WB)
- Implement all pipeline registers and stage logic
- Add data forwarding to minimize hazards
- Optimize performance: target CPI ~1.1-1.2

### Deliverables
- fetch_stage.sv, decode_stage.sv, execute_stage.sv, memory_stage.sv, writeback_stage.sv
- Pipeline registers: if_id_reg.sv, id_ex_reg.sv, ex_mem_reg.sv, mem_wb_reg.sv
- forwarding_unit.sv: hazard detection & bypass logic
- hazard_control.sv: stall logic for load-use dependencies
- Performance measurement showing CPI ~1.1-1.2

### Key Tasks
- [ ] Create fetch_stage.sv: PC management, IMEM read, branch/jump handling
- [ ] Create decode_stage.sv: Instruction decode, register file reads, immediate generation
- [ ] Create execute_stage.sv: ALU execution, address computation, branch resolution
- [ ] Create memory_stage.sv: DMEM interface (read/write with byte enables)
- [ ] Create writeback_stage.sv: Write-back mux (ALU / MEM / PC+4)
- [ ] Create all 4 pipeline registers (if_id, id_ex, ex_mem, mem_wb)
- [ ] Implement forwarding_unit.sv: Forward EX/MEM and MEM/WB results to EX inputs
- [ ] Implement hazard_control.sv: Detect load-use, insert 1-cycle stall
- [ ] Add branch squashing: On taken branch, flush IF/ID and ID/EX stages
- [ ] Verify pipeline correctness:
  - [ ] Data forwarding eliminates most hazards
  - [ ] Load-use stalls work correctly
  - [ ] Branch squashing recovers correctly
  - [ ] Fibonacci still produces correct result
- [ ] Measure CPI: Count cycles / instructions → target ~1.1-1.2

### Hazard Scenarios to Test
`systemverilog
// Data forwarding from EX/MEM
ADD x1, x2, x3       # Result in EX → x1
ADD x4, x1, x5       # Read x1 in EX → Forward from EX/MEM

// Data forwarding from MEM/WB
LW x6, 0(x7)         # Result in MEM → x6
ADD x8, x6, x9       # Read x6 in EX → Forward from MEM/WB (or stall if still in MEM)

// Load-use stall
LW x10, 0(x11)       # Result available in WB only
ADD x12, x10, x13    # Reads x10 in EX → STALL for 1 cycle

// Branch squashing
BEQ x1, x2, target   # Predicted not-taken, resolve in EX as taken
// → Flush IF/ID and ID/EX, restart with correct target
`

### Cumulative Progress
**~1600 LOC**: Pipelined CPU with forwarding, ~1.1-1.2 CPI on test programs

---

## Phase 4: Special Instructions, Integration & Documentation (Week 4)

### Goals
- Implement special instructions (POPCNT, FCLASS, FQUANT)
- Comprehensive system integration testing
- Performance analysis and documentation
- Prepare for synthesis (optional hardware deployment)

### Deliverables
- popcnt.sv: Bit count module
- fclass.sv: IEEE-754 fp16 type classification
- fquant.sv: fp16 → Q3.4 fixed-point quantization
- Complete test suite covering all functionality
- Performance report: CPI, resource utilization, timing
- Final design documentation

### Key Tasks
- [ ] Implement popcnt.sv: (data_in[7:0]) → count (0-8)
- [ ] Implement fclass.sv: Parse fp16 [sign, 5-bit exp, 10-bit frac] → classification (0-4)
- [ ] Implement fquant.sv: Convert fp16 to Q3.4 fixed-point with quantization
- [ ] Integrate special instructions into decode_stage and execute_stage
- [ ] System integration testing: All instructions working together
- [ ] Run comprehensive test suite:
  - [ ] Fibonacci F(20) produces correct output
  - [ ] Bit count on test vectors (0x00, 0xFF, 0x55, 0xAA, etc.)
  - [ ] FCLASS on various fp16 values (zero, denorm, norm, inf, nan)
  - [ ] FQUANT on positive/negative fp16 values
  - [ ] Mixed workload: Control flow + memory + arithmetic + special instructions
- [ ] Performance measurement:
  - [ ] Measure CPI on final workload → confirm ~1.1-1.2
  - [ ] Estimate area (LUTs) for XC7A35T → target ~30-40%
  - [ ] Check timing closure on critical paths
- [ ] Documentation:
  - [ ] Design specification: Final ISA, pipeline, hazard control
  - [ ] Test results: Pass/fail for each functional category
  - [ ] Code walkthrough: Key modules and design decisions
  - [ ] Performance analysis: CPI breakdown, area utilization

### Special Instruction Test Cases
`
POPCNT:
Input: 0x00 → Output: 0
Input: 0xFF → Output: 8
Input: 0x55 → Output: 4
Input: 0xAA → Output: 4

FCLASS:
+0.0 (0x0000)    → 0 (±Zero)
-0.0 (0x8000)    → 0 (±Zero)
+∞ (0x7C00)      → 1 (±Infinity)
-∞ (0xFC00)      → 1 (±Infinity)
NaN (0x7C01)     → 2 (NaN)
Norm (0x3C00)    → 3 (Normalized, =1.0)
Denorm (0x0001)  → 4 (Denormalized)

FQUANT (fp16 → Q3.4):
+2.5 (0x4480)    → 0x28 (0010_1000: +2.5 in Q3.4)
-1.25 (0xBD00)   → 0xEE (1110_1110: -1.25 as two's complement)
`

### Cumulative Progress
**~2000 LOC**: Fully functional pipelined CPU with all features, comprehensive testing and documentation

---

## Weekly Deliverables Summary

| Week | Focus | Key Modules | LoC Target | Verification |
|------|-------|-------------|-----------|--------------|
| 1 | Building blocks | ALU, RegFile, Decoder, IMEM, DMEM | ~800 | Unit tests pass |
| 2 | Single-cycle | cpu_top, control path, memory ops | ~1200 | Fibonacci, bit-count |
| 3 | Pipeline + Hazards | 5 stages, forwarding, stall logic | ~1600 | CPI ~1.1-1.2 |
| 4 | Special + Integration | POPCNT, FCLASS, FQUANT, system test | ~2000 | Full test suite |

---

## Risk Mitigation & Contingency

- **Week 1 risk:** Simulator setup delays → Start with simple testbenches in parallel
- **Week 2 risk:** Single-cycle CPU harder than expected → Pre-made testbenches to verify blocks incrementally
- **Week 3 risk:** Pipeline hazards complex → Study forwarding logic early, test incrementally (add forwarding gradually)
- **Week 4 risk:** Time running out → Prioritize: POPCNT >> FCLASS >> FQUANT if pressed
- **Hardware deployment:** If time permits after week 4, use Vivado to synthesize and test on XC7A35T

---

## Success Criteria

- ✅ All required instructions implemented and verified correct
- ✅ 5-stage pipeline working with CPI ~1.1-1.2
- ✅ Data forwarding eliminates ALU-to-ALU hazards
- ✅ Load-use stalls handled correctly
- ✅ Branch squashing works on taken branches
- ✅ Special instructions (POPCNT, FCLASS, FQUANT) functional
- ✅ Comprehensive test suite passing (Fibonacci, bit-count, floating-point)
- ✅ Code well-documented and ready for extension
- ✅ (Bonus) Synthesis shows reasonable resource usage (~30-40% LUT on XC7A35T)
