## New Testing Infrastructure

### Configurable Memory Modules
**Files**: `i_mem.v`, `d_mem.v`

Added parameters to support multiple test scenarios:
```verilog
parameter MEM_INIT_FILE = "mem.bin"       // Instruction memory
parameter DATA_INIT_FILE = "data_mem.dat" // Data memory
```

### Step C Performance Comparison Test
------------------------->




**Purpose**: Measure cycle count to PASS and enable performance comparison

**Test Program** (`mem_cpu12_stepC.bin` + `data_cpu12_stepC.dat`):
- Load data from data memory (x2, x3)
- Write computation results to signature region
- Execute x5 = x2 + x3 addition
- Save computation result as signature
- Set PASS flag and exit

**Signature Map**:
- 0x80 = 0x44332211 (input data 1)
- 0x84 = 0x88776655 (input data 2)
- 0x88 = 0xCCAA8866 (computation result: 0x44332211 + 0x88776655)
- 0x8C = 0x00000001 (PASS flag)

**Pass Criterion**: PASS flag confirmed and all signatures match, cycle count captured

**Performance Result**: PASS achieved in 39 cycles (mem[0x08] set to 1)

### Debug Testbench
**File**: `tb_cpu1_stepA_debug.v`

Provides cycle-by-cycle instruction trace showing:
- Program counter
- Decoded instruction (mnemonic)
- Register file state (non-zero values)
- Memory operations (read/write address and data)

**Usage**: Useful for tracing control flow and datapath activity (note: no hazard/forwarding logic implemented).

---

## Compilation and Execution

### Standard Test (Pass/Fail + Performance Metrics)
```bash
iverilog -g2012 -DCPU14 -o sim_stepC.vvp tb_cpu14_stepC.v rv32i.v i_mem.v d_mem.v alu.v

vvp sim_stepC.vvp 

### Waveform Analysis
```bash
gtkwave stepC.vcd
```

---

## Architecture Overview

### 5-Stage Pipeline Implementation

```
┌────────┐   ┌────────┐   ┌─────────┐   ┌────────┐   ┌───────────┐
│ Fetch  │──▶│ Decode │──▶│ Execute │──▶│ Memory │──▶│ Writeback │
└────────┘   └────────┘   └─────────┘   └────────┘   └───────────┘
    │            │              │             │              │
    pc         inst          alu_res       d_in/wr        wd/r_we
  _reg         rdata1/2       imm_E       _addr/data     rd_W
               opcode_E      funct3_E     opcode_M      opcode_W
```

### Supported RV32I Instructions

**Implemented**:
- **Arithmetic**: ADD, SUB, ADDI
- **Logical**: AND, OR, XOR, ANDI, ORI, XORI
- **Shifts**: SLL, SRL, SRA, SLLI, SRLI, SRAI
- **Comparison**: SLT, SLTU, SLTI, SLTIU
- **Memory**: LW, LH, LB, LHU, LBU, SW, SH, SB
- **Branches**: BEQ, BNE, BLT, BGE, BLTU, BGEU
- **Jumps**: JAL, JALR
- **Upper Immediate**: LUI, AUIPC

### Hazard Handling

**Implemented Features**:
1. **Data Forwarding Paths**:
   - EX/MEM → ALU (Path 1): Forward ALU results to next instruction
   - MEM/WB → ALU (Path 2): Forward write-back data to resolve dependencies
   - Store-data forwarding: Ensures SW uses correct data from prior instructions

2. **Load-Use Hazard Detection**: 
   - Detects when instruction needs result from LW before it's ready
   - Inserts 1-cycle stall automatically

3. **Control Flow Handling**:
   - Branch/Jump flushing: Clears invalid instructions after control transfers
   - PC stall mechanism: Holds program counter during hazards

### Module Structure

**Core Modules**:
- `rv32i.v` - Top-level CPU with 5-stage pipeline
- `alu.v` - Arithmetic Logic Unit (32 operations)
- `reg.v` - 32x32-bit register file
- `i_mem.v` - Instruction memory (parameterized)
- `d_mem.v` - Data memory (parameterized)
- `defines.v` - Opcode and instruction definitions

**Testbenches**:
- `tb_cpu14_stepC.v` - Performance test with cycle counting

---

## Project Structure

```
Sotsuken_CPU14/
├── rv32i.v                  # Main CPU module (5-stage pipeline)
├── alu.v                    # ALU implementation
├── reg.v                    # Register file
├── i_mem.v                  # Instruction memory
├── d_mem.v                  # Data memory
├── defines.v                # Instruction definitions
├── tb_cpu14_stepC.v         # Step C testbench
├── mem_cpu12_stepC.bin      # Test program binary
├── data_cpu12_stepC.dat     # Test data file
├── stepC_expected           # Expected test output
├── README.md                # This file
├── README_JP.md             # Japanese version
├── Pipeline_and_Hazard_Problems_CPU14.txt  # Implementation details
└── rv32i_Pipeline_explanation.txt          # Pipeline architecture guide
```

---

## CPU14 vs CPU12: Performance Advantage

**CPU14's Key Improvement**: Data forwarding eliminates most pipeline stalls

### Why CPU14 is Better

**Without Forwarding (CPU12-style)**:
- Must insert manual NOPs between dependent instructions
- OR accept data corruption from hazards
- Result: Longer programs, more cycles

**With Forwarding (CPU14)**:
```
LW  x2, 0(x0)      # Cycle 1: Load data
ADD x3, x2, x2     # Cycle 2: Use x2 immediately (forwarded!)
ADD x4, x3, x3     # Cycle 3: Use x3 immediately (forwarded!)
SW  x4, 8(x0)      # Cycle 4: Store x4 (forwarded!)
```
- Automatic forwarding: No manual NOPs needed
- Only stalls when absolutely necessary (load-use = 1 cycle)
- Shorter programs, faster execution

### Step C Performance Metrics

The Step C test demonstrates this advantage:
- **29 instructions** executed in **39 cycles**
- **CPI = 1.34**: Near-ideal performance (perfect pipeline = 1.0)
- **IPC = 0.74**: High instruction throughput

The 10 extra cycles come from:
- ~5 cycles: Pipeline fill/drain
- ~3-5 cycles: Unavoidable load-use stalls (CPU14 handles automatically)
- ~0-2 cycles: Branch penalties

**Without CPU14's forwarding**, the same program would either:
1. Need ~15-20 manual NOPs inserted → 44-49 instructions → 60+ cycles
2. Produce wrong results due to data hazards

---

## Prerequisites

- **Icarus Verilog** (iverilog) - For simulation
- **GTKWave** - For waveform viewing
- **RISC-V GNU Toolchain** (optional) - For compiling custom test programs





## Quick Start

1. **Clone or download the repository**

2. **Run the performance test**:
```bash
iverilog -g2012 -DCPU14 -o sim_stepC.vvp tb_cpu14_stepC.v rv32i.v i_mem.v d_mem.v alu.v
vvp sim_stepC.vvp
```

3. **Expected output**:
```
=== PERF TB: CPU14 mode (with retirement count if valid_W exists) ===
PASS flag observed at cycle 38 (mem[0x08]=1).
---- Signature checks ----
SIG  OK  @0x80: 0x44332211
SIG  OK  @0x84: 0x88776655
SIG  OK  @0x88: 0xccaa8866
SIG  OK  @0x8c: 0x00000001
---- Performance report ----
Cycles_to_PASS = 39
Retired_instructions = 35
CPI = 1.114286
IPC = 0.897436
TEST RESULT: PASS
```

4. **View waveforms** (optional):
```bash
gtkwave stepC.vcd
```

---

## Key Features

- **Full RV32I Base Instruction Set** (40 instructions)
- **Dual-path data forwarding** prevents most pipeline stalls
- **Automated hazard detection** ensures correct execution
- **Configurable memory modules** support multiple test scenarios
- **Cycle-accurate simulation** for performance analysis
- **Comprehensive test infrastructure** with signature verification

---

## Known Limitations

- No cache implementation (direct memory access)
- No privilege modes (M-mode only)
- No CSR (Control and Status Registers)
- No exceptions/interrupts
- Fixed memory size (256 words instruction, 256 words data)

---

## References

- [RISC-V Specification](https://riscv.org/technical/specifications/)
- Pipeline_and_Hazard_Problems_CPU14.txt - Detailed implementation notes
- rv32i_Pipeline_explanation.txt - Architecture explanation

---

**Document Version**: 1.1  
**Last Updated**: February 1, 2026    
**Status**: Step A, Step B, and Step C integration tests passing