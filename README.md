# Week 1: RTL Design, Synthesis and Simulation

This week provided a comprehensive foundation in digital design, covering the complete flow from RTL description to gate-level implementation. We explored simulation methodologies, synthesis techniques, optimization strategies, and best practices for reliable hardware design.

## Table of Contents
1. [Day 1: Verilog RTL Design & Simulation](#day-1-verilog-rtl-design--simulation-)
2. [Day 2: Synthesis, Timing Libraries & Flop Coding](#day-2-synthesis-timing-libraries--flop-coding-)
3. [Day 3: Logic Optimization](#day-3-logic-optimization-)
4. [Day 4: GLS & Verilog Best Practices](#day-4-gls--verilog-best-practices-)
5. [Day 5: Synthesis of Verilog Constructs](#day-5-synthesis-of-verilog-constructs-)
6. [Week 1 Summary](#week-1-summary)

---

## Day 1: Verilog RTL Design & Simulation 📝

The first day established the fundamental workflow for verifying any digital design, introducing the essential tools and methodologies used throughout the digital design process.

### Core Components

We learned the critical distinction between two fundamental elements of digital verification:

- **Design**: The actual hardware description written in Verilog
- **Testbench**: A separate Verilog file dedicated to applying stimulus and checking outputs

This separation allows for independent development and reusable verification components.

### Toolchain Introduction

We introduced the open-source toolchain that forms the backbone of our design flow:

| Tool | Purpose | Function |
|------|---------|----------|
| **iverilog** | Compiler and Simulator | Compiles Verilog code and runs simulations |
| **gtkwave** | Waveform Viewer | Graphically analyzes simulation results |

### Basic Simulation Workflow

The fundamental simulation process follows these steps:

1. **Write Verilog Code** - Create design and testbench files
2. **Compile with iverilog** - Process Verilog files into executable
3. **Run Simulation** - Execute simulation to generate `.vcd` file
4. **View Results in gtkwave** - Analyze waveforms and verify functionality

```bash
# Complete simulation flow
iverilog design.v testbench.v          # Compile
./a.out                                # Run simulation  
gtkwave testbench.vcd                  # View waveforms
```

---

## Day 2: Synthesis, Timing Libraries & Flop Coding ⚙️

Day 2 transitioned from simulation to synthesis, focusing on how RTL code is translated into physical gates and the critical role of timing libraries.

### Timing Libraries (.lib Files)

We introduced the concept of the **Process Design Kit (PDK)** and timing libraries:

- **Purpose**: Describe performance characteristics (timing, power, area) of standard cells
- **PVT Corners**: Libraries cover different Process, Voltage, and Temperature conditions
- **Critical for**: Accurate synthesis and timing analysis

**Example Library Name**: `sky130_fd_sc_hd__tt_025C_1v80.lib`
- `sky130`: SkyWater 130nm technology
- `tt`: Typical-Typical process corner  
- `025C`: 25°C temperature
- `1v80`: 1.80V supply voltage

### Synthesis Strategies

We compared two fundamental approaches to synthesis:

#### Hierarchical Synthesis
- **Approach**: Preserves the design's modular structure
- **Advantages**: 
  - ✅ Faster synthesis time
  - ✅ Easier debugging and maintenance
  - ✅ Lower memory usage
- **Disadvantages**: 
  - ❌ Less global optimization
  - ❌ Potential inefficiencies at module boundaries

#### Flatten Synthesis
- **Approach**: Collapses hierarchy for maximum optimization
- **Advantages**:
  - ✅ Maximum optimization across module boundaries
  - ✅ Best possible performance and area
- **Disadvantages**:
  - ❌ Slower synthesis process
  - ❌ Harder to debug
  - ❌ Higher memory requirements

### Coding Best Practices for Flip-Flops

**Golden Rule**: Use **non-blocking assignments (`<=`)** for sequential logic

```verilog
// ✅ CORRECT: Non-blocking for sequential logic
always @(posedge clk) begin
    q1 <= d;
    q2 <= q1;  // Creates proper shift register
end
```

#### Reset Types Comparison

| Reset Type | Behavior | Use Case |
|------------|----------|----------|
| **Synchronous Reset** | Resets only on clock edge | Eliminates metastability |
| **Asynchronous Reset** | Resets immediately | Immediate response required |

### Synthesis Optimizations

We discovered how synthesis tools intelligently optimize arithmetic operations:

**Example**: `a * 9` becomes `(a << 3) + a`
- Replaces expensive multiplier with shift and add
- Significant area and power savings
- Demonstrates tool intelligence in optimization

---

## Day 3: Logic Optimization ✨

This day focused on the automated optimization techniques used by synthesis tools to improve a design's **PPA (Performance, Power, and Area)**.

### Combinational Logic Optimization

#### Constant Propagation
- **Technique**: Simplifies logic where inputs are tied to constants
- **Examples**: 
  - `A & 0 = 0` (eliminates AND gate)
  - `A | 1 = 1` (eliminates OR gate)
- **Impact**: Significant area and power reduction

#### Boolean Logic Optimization
- **Technique**: Applies Boolean algebra rules to reduce complex expressions
- **Examples**:
  - `(A & B) | (A & ~B) = A` (simplifies to buffer)
  - De Morgan's laws application
- **Result**: Simpler, more efficient hardware

### Sequential Logic Optimization

#### Basic Optimizations

| Optimization | Description | Result |
|--------------|-------------|--------|
| **Sequential Constant Propagation** | Removes flip-flops with constant inputs | Replace with tie-hi/tie-lo cells |
| **Unused Output Removal** | Deletes unconnected flip-flops | Significant area and power savings |

#### Advanced Techniques

- **Retiming**: Moves registers to balance logic paths for better timing
- **Cloning**: Duplicates cells to fix high fanout timing issues  
- **State Optimization**: Removes unreachable states in FSMs

### Optimization Impact

The synthesis results clearly demonstrated the power of automatic optimization:
- Redundant logic eliminated
- Complex Boolean expressions simplified
- Unused sequential elements removed
- Overall PPA improvements achieved

---

## Day 4: GLS & Verilog Best Practices ✅

Day 4 covered critical verification steps and coding rules to ensure designs work correctly after synthesis.

### Gate Level Simulation (GLS)

**Purpose**: Verify the synthesized netlist using the same testbench as RTL simulation

**GLS Flow**:
```bash
# Compile with gate-level netlist and library models
iverilog primitives.v sky130_cells.v netlist.v testbench.v
./a.out
gtkwave testbench.vcd
```

**Benefits**:
- ✅ Verifies functional correctness post-synthesis
- ✅ Can include timing verification (delay annotation)
- ✅ Ultimate confidence in design functionality

### Simulation-Synthesis Mismatch

A **dangerous bug** where RTL simulates correctly but synthesizes to incorrect hardware.

#### Primary Causes

1. **Missing Sensitivity List**
   ```verilog
   // ❌ WRONG: Incomplete sensitivity list
   always @(sel) begin
       if (sel) y = a; else y = b;  // Missing 'a' and 'b'
   end
   
   // ✅ CORRECT: Complete sensitivity list  
   always @(*) begin
       if (sel) y = a; else y = b;
   end
   ```

2. **Blocking/Non-Blocking Assignments**

#### Critical Examples

**Sequential Logic**:
```verilog
// ✅ CORRECT: Non-blocking for sequential
always @(posedge clk) begin
    q1 <= d;
    q2 <= q1;  // Uses OLD value of q1 (shift register)
end
```

**Combinational Logic**:
```verilog
// ✅ CORRECT Ordering blocking assignment for combinational
always @(*) begin
    temp = a & b;      // temp updated immediately
    y = temp | c;      // y uses NEW value of temp
end
```

---

## Day 5: Synthesis of Verilog Constructs 🏗️

The final day explored how common Verilog constructs are interpreted by synthesis tools and the hardware they produce.

### `if` vs. `case` Statements

Understanding the hardware implications of different coding constructs:

#### Priority Logic (`if-else if`)
```verilog
if (sel == 2'b00)      y = a;
else if (sel == 2'b01) y = b;
else if (sel == 2'b10) y = c;
else                   y = d;
```
**Result**: Cascading chain of 2:1 MUXes (unbalanced delays)

#### Parallel Logic (`case`)  
```verilog
case (sel)
    2'b00: y = a;
    2'b01: y = b;
    2'b10: y = c;
    2'b11: y = d;
endcase
```
**Result**: Single 4:1 MUX (balanced delays, better performance)

### Avoiding Inferred Latches

**Critical Rule**: Any incomplete `if` or `case` in combinational logic infers unwanted latches.

**Solutions**:
- Always include final `else` clause
- Always include `default` case in `case` statements
- Assign values to ALL outputs in ALL branches

### `for` Loop vs. `generate` Block

Understanding the fundamental difference between behavioral and structural constructs:

| Construct | Location | Purpose | Synthesis Result |
|-----------|----------|---------|------------------|
| **`for` loop** | Inside `always` block | Describes behavior | Single large logic cone |
| **`generate` block** | Outside `always` block | Hardware replication | Multiple distinct instances |

#### `for` Loop Example (Behavioral)
```verilog
always @(*) begin
    for (i = 0; i < 8; i = i + 1) begin
        if (sel == i) y = data[i];  // Unrolled to parallel logic
    end
end
```

#### `generate` Block Example (Structural)
```verilog
genvar i;
generate
    for (i = 0; i < 32; i = i + 1) begin
        full_adder fa (.a(a[i]), .b(b[i]), .cin(c[i]), .sum(s[i]), .cout(c[i+1]));
    end
endgenerate  // Creates 32 separate full_adder instances
```

---

## Week 1 Summary

### Key Achievements 🎯

By the end of Week 1, we have established a solid foundation in:

1. **Complete Design Flow**: From RTL to gates
2. **Verification Methodology**: Simulation and GLS practices  
3. **Synthesis Understanding**: How code becomes hardware
4. **Optimization Awareness**: Automatic tool improvements
5. **Best Practices**: Coding rules for reliable designs

### Critical Concepts Mastered

#### Coding Rules
- **Sequential**: Use non-blocking (`<=`) assignments
- **Sensitivity Lists**: Always use `always @(*)` for combinational logic
- **Completeness**: Avoid incomplete `if`/`case` statements

#### Synthesis Awareness
- Understand hardware implications of coding choices
- Leverage automatic optimizations
- Choose constructs based on desired hardware structure

### Tools and Techniques

| Category | Tools/Techniques | Purpose |
|----------|------------------|---------|
| **Simulation** | iverilog, gtkwave | RTL verification |
| **Synthesis** | Yosys, ABC | RTL to gates transformation |
| **Libraries** | PDK timing libraries | Technology mapping |
| **Verification** | GLS, waveform comparison | Post-synthesis validation |

### Key Takeaways for Professional Practice

1. **Verification First**: Always verify before synthesis
2. **Code for Synthesis**: Understand hardware implications  
3. **Follow Golden Rules**: Prevent simulation-synthesis mismatches
4. **Leverage Tools**: Use automatic optimizations effectively
5. **Complete Verification**: Always run GLS on synthesized designs

Week 1 has provided the essential knowledge and practical skills needed for successful digital design implementation.
