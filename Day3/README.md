# Day 3: Combinational and Sequential Optimizations

Day 3 we focus on the intelligence of the synthesis tool. We'll explore how tools like Yosys automatically transform our initial Verilog code into a gate-level netlist that is significantly more efficient in terms of performance, power, and area (PPA).

## Table of Contents
1. [Introduction to Logic Optimization](#1-introduction-to-logic-optimization)
2. [Combinational Logic Optimization](#2-combinational-logic-optimization-)
3. [Sequential Logic Optimization](#3-sequential-logic-optimization-)

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

y = a'c'+ac

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
module counter_opt (input clk , input reset , output q);
reg [2:0] count;
assign q = count[0];

always @(posedge clk ,posedge reset)
begin
	if(reset)
		count <= 3'b000;
	else
		count <= count + 1;
end

endmodule
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
|--|----------|----------------------|---------|
| `dff_const1.v` | D-flop with D='1' and async reset | No optimization | <img width="610" height="309" alt="dff_const1_show_crop" src="https://github.com/user-attachments/assets/d815d617-d98b-4a0d-9e52-cbd4d350af84" />|
| `dff_const2.v` | D-flop with D='1' and async set | Constant value 1 | <img width="596" height="390" alt="dff_const2_show_crop" src="https://github.com/user-attachments/assets/b86d8147-8cfe-4350-8bb9-de4f2be27a28" />|
| `dff_const3.v` | 2 D-FF's with first FF having D = '1' (async reset) and second FF D pin connected to the first FF's Q pin (async set) | No optimization |<img width="672" height="102" alt="dff_const3_show_crop" src="https://github.com/user-attachments/assets/f6c70d15-98fa-4fd5-abb8-4c5c976686ec" />|
| `dff_const4.v` | 2 D-FF's (async set) with first FF having D = '1' and second FF D pin connected to the first FF's Q pin  | Constant value 1 | <img width="593" height="389" alt="dff_const4_show_crop" src="https://github.com/user-attachments/assets/a7ffbf6f-f152-4c49-8306-ba2f01a18c25" />|
| `dff_const5.v` | 2 D-FF's (async reset) with first FF having D = '1' and second FF D pin connected to the first FF's Q pin  | No optimization | <img width="668" height="301" alt="dff_const5_show_crop" src="https://github.com/user-attachments/assets/f60e2936-fa2e-4ca4-bc75-fe3c19f9d1eb" />|
| `counter_opt.v` | A counter with output as count[0] | Unused output optimization| <img width="1194" height="157" alt="counter_opt_show_crop" src="https://github.com/user-attachments/assets/d966d0a2-9531-43c7-b5ee-5150dd58111c" />|

### Examples (Waveform and analysis)

| File | Waveform | Explaination | 
|--|----------|----------------------|
| `dff_const1.v` | <img width="1285" height="868" alt="dff_const1_waveform" src="https://github.com/user-attachments/assets/3df168b7-b885-4878-a370-02c27e3abd65" />| Q changes to 1 only at the rising edge of the clock and so we cannot write Q = reset'|
| `dff_const2.v` | <img width="1285" height="859" alt="dff_const2_wf" src="https://github.com/user-attachments/assets/412d1967-ba4d-4740-89c0-ae47d3124c64" />| Q is always a constant value 1|
| `dff_const3.v` | <img width="1283" height="866" alt="dff_const3_wf" src="https://github.com/user-attachments/assets/bfd24d1a-ace8-4df5-927e-d75c5e42a820" />| When reset goes low, q1 is set to 1 at the next rising edge of the clock. <br> At that same edge, q captures the previous value of q1, which is 0.<br> By the following clock cycle, both q1 and q hold the value 1|
| `dff_const4.v` | <img width="1346" height="826" alt="dff_const4_wf" src="https://github.com/user-attachments/assets/d9067217-ddea-4f10-a852-a8679a1e432c" />| Q is always a constant value 1|
| `dff_const5.v` | <img width="1292" height="730" alt="dff_const5_wf" src="https://github.com/user-attachments/assets/b44af497-d003-44b1-837b-a180b073c043" />| When reset goes low, q1 is set to 1 at the next rising edge of the clock. <br> At that same edge, q captures the previous value of q1, which is 0.<br> By the following clock cycle, both q1 and q hold the value 1|
| `counter_opt.v` |  <img width="1287" height="736" alt="counter_opt_wf" src="https://github.com/user-attachments/assets/27978d7b-4dfc-403d-96de-e7fd88743e7f" />| For each number of the counter the clock toggles|
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
