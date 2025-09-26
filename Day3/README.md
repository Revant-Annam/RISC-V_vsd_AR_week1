# Day 3: Combinational and Sequential Optimizations

Day 3 we focus on the intelligence of the synthesis tool. We'll explore how tools like Yosys automatically transform our initial Verilog code into a gate-level netlist that is significantly more efficient in terms of performance, power, and area (PPA).

## Table of Contents
1. [Introduction to Logic Optimization](#1-introduction-to-logic-optimization)
2. [Combinational Logic Optimization](#2-combinational-logic-optimization-)
3. [Sequential Logic Optimization](#3-sequential-logic-optimization-)
4. [Optimization Results Summary](#4-optimization-results-summary)

---

## 1. Introduction to Logic Optimization

Logic optimization is the process of automatically modifying a logic circuit to create an equivalent circuit that is "better." A circuit is considered **equivalent** if it produces the exact same outputs for the same sequence of inputs.

The definition of "better" depends on the design goals, which are typically a trade-off between three key metrics, often called **PPA**:

### Performance (Speed) ⚡
- **Primary goal**: Increase the maximum clock speed
- **Method**: Reduce delay of the longest logical path between flip-flops (critical path)
- **Impact**: Higher operating frequencies, better system performance

### Power 🔋
- **Goals**: Minimize energy consumption
- **Methods**: 
  - Reduce switching activity
  - Eliminate redundant logic
  - Use lower-power standard cells
- **Impact**: Longer battery life, reduced heat generation

### Area 📐
- **Goal**: Minimize physical size on silicon die
- **Method**: Reduce total number and size of logic gates
- **Impact**: Lower manufacturing cost, higher chip density

> **Note:** Logic optimization is a core function of the synthesis process and is performed by `abc` command in the Yosys.

---

## 2. Combinational Logic Optimization 🔄

Combinational logic optimization is the process of optimizing the logic between registers to create the most efficient design, resulting in significant area and power savings. The two primary techniques used by synthesis tools are:

### Constant Propagation

This technique involves simplifying logic if one or more inputs to a circuit are constant (i.e., always '1' or '0'). The synthesis tool propagates these constant values through the logic, eliminating redundant gates.

For example, 
- Consider the circuit below with the expression Y=((A⋅B)+C')
- If the tool knows that input A is always tied to 0, the logic simplifies as follows:
  1. The output of the AND gate, (A⋅B), becomes (0⋅B), which is always 0.
  2. The entire expression becomes Y=(0+C')
  3. This simplifies to Y=C' 

As a result, the tool replaces the original AND and NOR gates with a single, much more efficient NOT gate (inverter).

**Boolean Algebra:**
- `A & 0 = 0` (AND with 0 always produces 0)
- `A | 1 = 1` (OR with 1 always produces 1)
- `A & 1 = A` (AND with 1 passes the signal through)
- `A | 0 = A` (OR with 0 passes the signal through)

### Boolean Logic Optimization

This is a more powerful technique where the synthesis tool applies the rules of Boolean algebra to transform a complex logic expression into a simpler, functionally equivalent one.

Consider a complex Verilog expression that synthesizes to a chain of multiplexers. Through analysis, the tool can derive the full Boolean expression for the output.
For the expression shown, the tool derives:
```verilog
assign y= a?(b?c:(c?a:0)):c';
```

By applying Boolean algebra, the synthesis tool can simplify this complex equation down to its most fundamental form:

Y= a'c'+ac

This is the expression for an XNOR gate. The tool then replaces the entire complex chain of multiplexers with a single, highly optimized XNOR gate, leading to massive savings in area and a significant improvement in performance.

### Combinational Optimization Examples

| File | Original Logic | Expected Optimization | Result |
|------|---------------|----------------------|--------|
| `opt_check1.v` | y = a?b:0; | Reduced to `and` gate with 2 inputs| <img width="595" height="139" alt="opt_check_show_cropped" src="https://github.com/user-attachments/assets/ff2ca467-1aba-4b99-9054-2224739e22d6" />|
| `opt_check2.v` | y = a?1:b; | Reduced to `or` gate with 2 inputs | <img width="595" height="139" alt="opt_check2_show_crop" src="https://github.com/user-attachments/assets/23f0dc8f-b50a-459c-881d-37aa19eaf0b3" />|
| `opt_check3.v` | y = a?(c?b:0):0; | Reduced to `and` gate with 3 inputs | <img width="595" height="139" alt="opt_check3_show_crop" src="https://github.com/user-attachments/assets/aeae07c1-8cd7-487d-9e79-9328c563d7fe" />|
| `opt_check4.v` | y = a?(b?(a&c):c):c'; | Reduced to `xnor` gate | <img width="595" height="139" alt="opt_check4_show" src="https://github.com/user-attachments/assets/513b8f0c-62b3-43a9-a110-3a4bf037f821" />|
| `multiple_module_opt.v` | y = c or (b&a); (multiple modules) | `and` gate & `or` gate  |<img width="590" height="296" alt="multi_mod_flat_opt_show_crop" src="https://github.com/user-attachments/assets/fd8731ab-cefc-4e16-aef3-e536c682a05e" />|
| `multiple_module_opt.v` | y = 0&a&b&c&d (multiple modules) | Constant value of 0 | <img width="405" height="548" alt="multi_mod_opt2_show_crop" src="https://github.com/user-attachments/assets/8be43991-9798-4698-b146-6b3bfb90375b" />|

### Yosys Commands for Combinational Optimization

```tcl
# Read design and library
yosys> read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
yosys> read_verilog opt_check1.v

# Synthesize with optimization
yosys> synth -top opt_check1

# Apply combinational optimizations
yosys> opt_clean -purge

# Technology mapping
yosys> abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# View results
yosys> show
```

---

## 3. Sequential Logic Optimization ⏰

Sequential optimizations focus on the flip-flops (registers) and the overall state of the design. They can be divided into **basic techniques** (commonly applied) and **advanced techniques** (more complex).

### Basic Sequential Optimizations

These fundamental optimizations target and remove redundant flip-flops from the design.

#### Sequential Constant Propagation

This optimization simplifies logic when a flip-flop's input is tied to a constant value ('1' or '0'). The tool can often eliminate the flip-flop and its downstream logic entirely.

In the example shown, a D-flip-flop's D input is tied to 0 and active high reset.
  1. After the first clock cycle, when reset is low, the flop's output Q will be 0.
  2. After few clock cycles when the reset is high then Q will be 0.
  3. The value of Q doesnot change so it can be replaced by Q=0.

The synthesis tool recognizes that the output Y will always be 0. It optimizes the circuit by removing the flip-flop, replacing them with a single constant 0 cell.

#### Unused Output Optimization

If the output of a flip-flop is not connected to any other logic (dangling/unloaded output), the flip-flop serves no purpose. The synthesis tool identifies and removes these unused registers.

**Example:**
```verilog
// Original design
reg unused_reg;
always @(posedge clk) begin
    unused_reg <= data;  // Output never used
end

// Optimized result: Entire flip-flop removed
// (Nothing replaces it since output was unused)
```

### Advanced Optimizations

These are more complex techniques, often performed during physical synthesis when the tool is aware of the chip's layout.

#### State Optimization

This technique analyzes **Finite State Machines (FSMs)** and removes any "unreachable" states to simplify the state transition logic.

**Benefits:**
- Reduces number of flip-flops needed for state encoding
- Simplifies combinational logic for state transitions
- Improves overall FSM performance

#### Retiming

This powerful technique improves clock speed by **moving registers across combinational logic** to better balance delays between pipeline stages.

**Process:**
1. **Analyze critical paths** in the design
2. **Move flip-flops** from non-critical to critical paths  
3. **Balance pipeline stages** for optimal timing
4. **Maintain functional equivalence**

#### Sequential Logic Cloning

This is a **physical-aware optimization** used to fix timing issues caused by high fanout, where a single cell drives many physically distant cells.

**Solution:** "Clone" the source cell, creating identical copies placed closer to their destinations.

**Benefits:**
- Reduces wire delays
- Improves timing closure
- Balances electrical loads

### Sequential Optimization Examples

| File | Original Logic | Expected Optimization | Result |
|------|---------------|----------------------|--------|
| `dff_const1.v` | D-flop with D='0' | Removed, output = tie-low | ✅ Constant propagation |
| `dff_const2.v` | D-flop with D='1' | Removed, output = tie-high | ✅ Constant propagation |
| `dff_const3.v` | D-flop with unused output | Completely removed | ✅ Dead code elimination |
| `dff_const4.v` | Two identical D-flops | Merged into single flop | ✅ Resource sharing |

### Yosys Commands for Sequential Optimization

```tcl
# Read design and library
yosys> read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
yosys> read_verilog dff_const1.v

# Synthesize with sequential optimization
yosys> synth -top dff_const1

# Apply sequential optimizations
yosys> opt_clean -purge

# Map flip-flops to library
yosys> dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# Technology mapping
yosys> abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

# View results
yosys> show
yosys> stat
```

---

## 4. Optimization Results Summary

### Impact of Optimizations

| Optimization Type | Area Savings | Power Savings | Performance Impact |
|------------------|--------------|---------------|--------------------|
| **Constant Propagation** | High | High | Positive |
| **Boolean Simplification** | Medium | Medium | Positive |
| **Sequential Constant** | Very High | Very High | Positive |
| **Unused Logic Removal** | High | High | Neutral |
| **State Optimization** | Medium | Medium | Positive |
| **Retiming** | Neutral | Neutral | Very Positive |

### Key Optimization Principles

1. **Equivalence Preservation**: All optimizations maintain functional equivalence
2. **PPA Trade-offs**: Optimizations typically improve all three metrics
3. **Hierarchical Awareness**: Some optimizations work better with flattened designs
4. **Physical Awareness**: Advanced optimizations consider chip layout
5. **Tool Intelligence**: Modern synthesis tools automatically apply most optimizations

### Best Practices for Optimization-Friendly Code

```verilog
// ✅ Good: Clear, simple logic
assign result = enable ? data : 1'b0;

// ❌ Avoid: Unnecessarily complex expressions  
assign result = (enable & data) | (~enable & 1'b0);

// ✅ Good: Explicit constants
parameter RESET_VALUE = 8'h00;

// ✅ Good: Proper reset handling
always @(posedge clk or posedge reset) begin
    if (reset)
        q <= RESET_VALUE;
    else
        q <= d;
end
```

---

## Key Takeaways

1. **Automatic Intelligence**: Modern synthesis tools automatically optimize designs for PPA
2. **Constant Propagation**: One of the most effective optimization techniques
3. **Boolean Algebra**: Synthesis tools are expert at simplifying complex logic expressions
4. **Sequential Optimization**: Removing redundant flip-flops provides significant savings
5. **Design Awareness**: Write RTL code that enables optimization rather than hindering it
6. **Tool Verification**: Always verify that optimized designs maintain functional equivalence

Understanding these optimization techniques helps you write better RTL code and interpret synthesis results more effectively, leading to higher-quality digital designs.
