# Day 4: GLS, Blocking vs Non-Blocking, and Synthesis-Simulation Mismatch

On Day 4, we focus on advanced verification and crucial Verilog coding practices that ensure our design behaves as expected after synthesis. We'll cover Gate Level Simulation (GLS), the dangers of simulation-synthesis mismatch, and one of the most important concept for preventing it: the correct use of blocking and non-blocking assignments.

## Table of Contents
1. [Gate Level Simulation (GLS)](#1-gate-level-simulation-gls-)
2. [Simulation and Synthesis Mismatch](#2-simulation-and-synthesis-mismatch-)
3. [Blocking vs. Non-Blocking Statements](#3-blocking-vs-non-blocking-statements)
4. [Lab Results Analysis](#4-lab-results-analysis)

---

## 1. Gate Level Simulation (GLS) 🧪

### What is GLS?

**Gate Level Simulation (GLS)** is the process of running a simulation on the gate-level netlist of a design, which is the output of the synthesis tool. Instead of simulating the high-level RTL code, we simulate the circuit as it is described by the interconnection of standard library cells (like AND, OR, and DFF gates).

Because the netlist is logically equivalent to the original RTL, the same testbench can be used to verify its functionality.

### Why is GLS Important?

GLS serves two primary purposes:

#### 1. Verify Logical Correctness ✅
- **Main goal**: Confirms that the synthesis tool has correctly translated RTL into gate-level circuit
- **Catches**: Functional bugs introduced during synthesis
- **Ensures**: What you simulated matches what you synthesized

#### 2. Timing Verification ⏱️
- **Advanced use**: Verify timing behavior of the design
- **Requires**: Delay annotation 
- **Purpose**: Ensure setup/hold times are met in real hardware

### The GLS Flow and Required Inputs

The netlist and the RTL code use the same testbench as the input and the output ports of both are same. The GLS flow is nearly identical to RTL simulation, but uses different input files:

| Component | RTL Simulation | GLS Simulation |
|-----------|----------------|----------------|
| **Design Under Test** | Original `.v` RTL files | Synthesized `.v` netlist |
| **Testbench** | Same testbench `.v` file | Same testbench `.v` file |
| **Library Models** | Not required | Gate-level Verilog models |

### GLS Flow Diagram

<img width="1920" height="1080" alt="Screenshot (309)" src="https://github.com/user-attachments/assets/dbcf11a1-ebc3-483d-80e0-cffb4f4da06d" />


### Invoking GLS

Running a Gate Level Simulation involves three steps:

#### Step 1: Compile the Files
```bash
# Compile using iverilog
# Include: library models + synthesized netlist + testbench
iverilog ../my_lib/verilog_models/primitives.v \
         ../my_lib/verilog_models/sky130_fd_sc_hd.v \
         ternary_operator_mux_netlist.v \
         tb_ternary_operator_mux.v
```

#### Step 2: Execute the Simulation
```bash
# Run the compiled simulation
./a.out
```

#### Step 3: View the Waveform
```bash
# Open waveform viewer
gtkwave tb_ternary_operator_mux.vcd
```

---

## 2. Simulation and Synthesis Mismatch 

A **simulation-synthesis mismatch** occurs when your Verilog code behaves one way in RTL simulation but produces different, unintended hardware after synthesis. This is a critical bug because your initial verification may pass, but the final chip will not work as designed.

### Primary Causes of Mismatch

#### 1. Missing Sensitivity List

**Problem**: Incomplete sensitivity list in combinational logic

```verilog
// ❌ WRONG: Missing inputs in sensitivity list
always @(sel) begin  // Only sensitive to 'sel', not 'a' or 'b'
    if (sel)
        y = a;
    else  
        y = b;
end

// ✅ CORRECT: Complete sensitivity list
always @(*) begin    // Sensitive to all inputs
    if (sel)
        y = a;
    else
        y = b;
end
```

**Impact**: Simulator won't update output when `a` or `b` changes, but synthesized hardware will.

#### 2. Blocking Assignments

**Problem**: Using blocking assignment type for sequential/combinational logic

```verilog
// ❌ WRONG: Blocking in sequential logic
always @(posedge clk) begin
    q1 = d;      // Blocking assignment
    q2 = q1;     // Uses new value of q1 immediately
end

// ✅ CORRECT: Non-blocking in sequential logic  
always @(posedge clk) begin
    q1 <= d;     // Non-blocking assignment
    q2 <= q1;    // Uses old value of q1
end
```

#### 3. Non-Standard Verilog Coding

**Problem**: Using language features that synthesis tools interpret differently

Examples:
- Initial blocks in synthesizable code
- Delays in synthesizable code  
- Multiple clock edges in sensitivity list
- X and Z assignments in synthesis

### Detecting Mismatches

The most reliable way to detect mismatches is through **comprehensive GLS**:

1. **Run RTL simulation** with your testbench
2. **Synthesize the design** to get gate-level netlist
3. **Run GLS** with the same testbench
4. **Compare waveforms** - they should be identical

---

## 3. Blocking and Non-Blocking Statements

### Definitions

#### Blocking Assignment (`=`)
- **Behavior**: Executes statements **sequentially**, in the order they're written
- **Timing**: Each assignment completes before the next one starts
- **Usage**: Combinational logic modeling

#### Non-Blocking Assignment (`<=`)
- **Behavior**: Executes statements in **parallel**
- **Timing**: All updates scheduled to happen simultaneously at end of time step
- **Usage**: Sequential and combinational logic modeling

### Caveats of blocking statements

#### Example 1: Sequential Logic (Shift Register)

**❌ WRONG: Using Blocking in Sequential Logic**
```verilog
// This creates a BUFFER, not a shift register!
always @(posedge clk) begin
    q1 = d;      // q1 gets d immediately
    q2 = q1;     // q2 gets the NEW value of q1 (which is d)
    q3 = q2;     // q3 gets the NEW value of q2 (which is d)
end
// Result: q1 = q2 = q3 = d (all same value)
```

**✅ CORRECT: Using Non-Blocking in Sequential Logic**
```verilog
// This creates a proper shift register
always @(posedge clk) begin
    q1 <= d;     // q1 scheduled to get d
    q2 <= q1;    // q2 scheduled to get OLD value of q1
    q3 <= q2;    // q3 scheduled to get OLD value of q2  
end
// Result: Proper shift register operation
```

#### Example 2: Combinational Logic

**❌ WRONG Order of Blocking in Combinational Logic**
```verilog
// This creates unintended latches!
always @(*) begin
    y = temp | c;     // y uses OLD value of temp
    temp = a & b;     
end
// Result: Unintended delay/latch behavior
```

**✅ CORRECT Order of Blocking in Combinational Logic**  
```verilog
// This creates proper combinational logic
always @(*) begin
    temp = a & b;      // temp gets updated immediately
    y = temp | c;      // y uses NEW value of temp
end
// Result: Pure combinational logic
```

---

## 4. Lab Results Analysis

This section compares RTL simulation waveforms with GLS waveforms, highlighting the importance of correct coding practices and post-synthesis verification.

### Test Cases Summary

| File | Description | RTL Behavior | GLS Behavior | Mismatch? |
|------|-------------|--------------|--------------|-----------|
| `ternary_operator_mux.v` | Properly coded 2:1 MUX | <img width="1278" height="740" alt="ternary_mux_wf" src="https://github.com/user-attachments/assets/44e6e4e9-5aa2-4ff8-8fc7-7502db9393eb" />|<img width="1278" height="761" alt="ternary_mux_net_wf" src="https://github.com/user-attachments/assets/a6ad4cd9-0f31-4910-b273-9a468951cae6" />| ❌ No |
| `bad_mux.v` | MUX with incomplete sensitivity | <img width="1280" height="903" alt="bad_mux_wf" src="https://github.com/user-attachments/assets/3e920ff2-8599-48a2-a64c-16ae937cf273" />|<img width="1278" height="849" alt="bad_mux_net_wf" src="https://github.com/user-attachments/assets/5d48870f-1230-4fc1-a723-bafaab6bd555" />| ✅ Yes |
| `blocking_caveat.v` | Wrong ordering of blocking statements | <img width="1282" height="726" alt="blocking_caveat_wf" src="https://github.com/user-attachments/assets/bc2662c1-7043-4da2-b33d-50c29c79c294" />| <img width="1273" height="732" alt="blocking_caveat_net_wf" src="https://github.com/user-attachments/assets/2730f7d1-a859-4589-a178-75065949ae10" />| ✅ Yes |

### Detailed Analysis

#### Test Case 1: `correct_mux.v` ✅
```verilog
// Properly coded 2:1 MUX
always @(*) begin           // Complete sensitivity list
    if (sel)
        y = a;              // Blocking for combinational
    else
        y = b;
end
```

**Result**: 
- **RTL Simulation**: Correct MUX behavior
- **GLS Simulation**: Identical waveform  
- **Conclusion**: Synthesis successful, no mismatch

#### Test Case 2: `sens_list_mux.v` ⚠️
```verilog
// MUX with incomplete sensitivity list
always @(sel) begin         // Missing 'a' and 'b' in sensitivity
    if (sel)
        y = a;
    else
        y = b;
end
```

**Result**:
- **RTL Simulation**: Output doesn't change when `a` or `b` change (wrong!)
- **GLS Simulation**: Proper MUX behavior (synthesis inferred correct logic)
- **Conclusion**: Dangerous mismatch - RTL sim passes but behaves differently than hardware

#### Test Case 3: `blocking_ff.v` ⚠️  
```verilog
// Shift register using blocking assignments
always @(posedge clk) begin
    q1 = d;                 // Blocking in sequential logic
    q2 = q1;                // Gets new value immediately
end
```

**Result**:
- **RTL Simulation**: Behaves like buffer (`q1 = q2 = d`)
- **GLS Simulation**: Proper shift register behavior
- **Conclusion**: Mismatch due to incorrect assignment type

### Waveform Comparison Example

```
Time:     0ns   5ns   10ns  15ns  20ns
        ┌─────┬─────┬─────┬─────┬─────┐
d       │  0  │  1  │  0  │  1  │  0  │
        └─────┴─────┴─────┴─────┴─────┘

RTL Simulation (blocking_ff.v - WRONG):
        ┌─────┬─────┬─────┬─────┬─────┐
q1      │  0  │  1  │  0  │  1  │  0  │  ← Same as d
        └─────┴─────┴─────┴─────┴─────┘
        ┌─────┬─────┬─────┬─────┬─────┐
q2      │  0  │  1  │  0  │  1  │  0  │  ← Same as d (buffer)
        └─────┴─────┴─────┴─────┴─────┘

GLS Simulation (synthesized netlist - CORRECT):
        ┌─────┬─────┬─────┬─────┬─────┐
q1      │  0  │  1  │  0  │  1  │  0  │  ← Same as d
        └─────┴─────┴─────┴─────┴─────┘
        ┌─────┬─────┬─────┬─────┬─────┐
q2      │  0  │  0  │  1  │  0  │  1  │  ← Delayed by 1 clock (shift register)
        └─────┴─────┴─────┴─────┴─────┘
```

---

## Key Takeaways

### Critical Rules to Remember 🎯

1. **Always use GLS** to verify synthesized netlists
2. **Sequential logic**: Use non-blocking assignments (`<=`)
3. **Combinational logic**: Use blocking assignments (`=`)
4. **Sensitivity lists**: Always use `always @(*)` for combinational logic
5. **Verify early and often**: Don't wait until tape-out to run GLS

### Best Practices for Avoiding Mismatches

#### ✅ DO:
- Use `always @(*)` for all combinational logic
- Use non-blocking assignments in clocked always blocks
- Use blocking assignments in combinational always blocks
- Run GLS on all synthesized designs
- Compare RTL and GLS waveforms

#### ❌ DON'T:
- Mix blocking and non-blocking in same always block
- Use incomplete sensitivity lists
- Ignore GLS failures
- Use delays in synthesizable code
- Use initial blocks in synthesizable modules

### Debugging Mismatch Workflow

```
1. Identify mismatch in GLS
        ↓
2. Check sensitivity lists
        ↓
3. Verify blocking/non-blocking usage
        ↓
4. Review synthesis warnings
        ↓
5. Fix RTL code
        ↓
6. Re-run RTL simulation
        ↓
7. Re-synthesize
        ↓
8. Re-run GLS
        ↓
9. Verify waveforms match
```

Understanding these concepts is crucial for creating reliable digital designs that work correctly in silicon. Proper use of blocking/non-blocking assignments and thorough GLS verification are essential skills for any digital design engineer.
