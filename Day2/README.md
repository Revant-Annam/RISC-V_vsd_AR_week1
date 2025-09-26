# Day 2: Timing Libraries, Synthesis Approaches, and Efficient Flip-Flop Coding Styles

On Day 2, we dive deeper into the synthesis process by understanding the critical role of timing libraries and PVT corners. We also explore different synthesis strategies and learn about best practices for coding sequential logic in Verilog.

## Table of Contents
1. [Timing Libraries](#1-timing-libraries)
2. [Hierarchical vs. Flatten Synthesis](#2-hierarchical-vs-flatten-synthesis)
3. [Different Coding Styles for Flip-Flops](#3-different-coding-styles-for-flip-flops)
4. [Results](#4-results)

---

## 1. Timing Libraries

### What is a PDK?

A **Process Design Kit (PDK)** is a collection of files provided by a semiconductor foundry (like SkyWater) that enables designers to create an Integrated Circuit (IC) for that specific manufacturing process. It includes:

- **Technology files** - Process parameters and design rules
- **Simulation models** - Device models for circuit simulation  
- **Timing Libraries (.lib files)** - Performance characteristics of pre-designed standard cells

### Understanding the Library Name

The name of a timing library file is very descriptive. Let's break down the example for the SKY130 PDK:

**Example:** `sky130_fd_sc_hd__tt_025C_1v80.lib`

| Component | Description |
|-----------|-------------|
| `sky130` | SkyWater 130nm manufacturing technology |
| `fd_sc_hd` | Foundry Standard Cell library with High-Density cells |
| `tt` | Process corner: Typical-Typical |
| `025C` | Operating Temperature: 25° Celsius |
| `1v80` | Operating Voltage: 1.80 Volts |

### PVT Corners Explained

A chip's performance varies based on **Process, Voltage, and Temperature (PVT)**. To ensure a design will function reliably, it must be tested across a range of these conditions, known as PVT corners.

**Common PVT Corners:**
- **Slow corner** - Worst-case performance (high temperature, low voltage, slow process)
- **Typical corner** - Nominal performance conditions  
- **Fast corner** - Best-case performance (low temperature, high voltage, fast process)

Testing across these corners ensures the design works under all expected operating conditions.

---

## 2. Hierarchical vs. Flatten Synthesis

When synthesizing a design that contains multiple sub-modules, the synthesis tool can take one of two main approaches: **Hierarchical** or **Flatten**.

### Hierarchical Synthesis

Hierarchical synthesis preserves the modular structure of the design. The tool synthesizes each module in the hierarchy as a distinct entity and then connects them. This is the default behavior in Yosys.

#### Advantages ✅
- **Faster synthesis** - Each module synthesized independently
- **Lower memory usage** - Smaller problem size per module
- **Easier debugging** - Module boundaries preserved for traceability

#### Disadvantages ❌
- **Less optimal design** - Cannot perform optimizations across module boundaries
- **Potential inefficiencies** - May miss global optimization opportunities

#### Yosys Command Flow (Hierarchical)

```tcl
yosys

read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

read_verilog multiple_modules.v

synth -top multiple_modules

abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

write_verilog -noattr multiple_modules_hier_netlist.v
```

Example:

<img width="800" height="450" alt="show_multi_mod" src="https://github.com/user-attachments/assets/112b1b9c-4369-4a74-b5cf-0d03f866c172" />


### Flatten Synthesis

Flatten synthesis is an approach where the tool first collapses the entire design hierarchy, merging all sub-modules into a single, large module.

#### Advantages ✅
- **Maximum optimization** - Allows optimization across module boundaries
- **Best performance** - Potentially smallest area and fastest timing
- **Global view** - Tool can see entire design for optimization

#### Disadvantages ❌
- **Slower synthesis** - Can be very slow and memory-intensive for large designs
- **Harder debugging** - Original module structure lost
- **Complex ECOs** - Engineering change orders more difficult

#### Yosys Command Flow (Flatten)

The command `flatten` removes hierarchy from your design and converts it into a single module netlist.

```tcl
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

read_verilog multiple_modules.v

flatten

synth -top multiple_modules

abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

write_verilog -noattr multiple_modules_flat_netlist.v
```
Example:

<img width="800" height="450" alt="show_multi_mod_flat" src="https://github.com/user-attachments/assets/ab264664-1a2b-45c0-a8a7-0d038f42fff2" />


---

## 3. Different Coding Styles for Flip-Flops

Flip-flops are essential for synchronizing digital logic, and set/reset signals are used to force them into a known initial state.

### Asynchronous Reset

An asynchronous reset changes the flop's state **immediately**, independent of the clock.

```verilog
always @ (posedge clk , posedge async_reset)
begin
	if(async_reset)
		q <= 1'b0;
	else	
		q <= d;
end
```

**Key Characteristics:**
- Reset in sensitivity list alongside clock
- Immediate response to reset assertion
- Used when instant reset response is required

### Synchronous Reset

A synchronous reset only affects the flop on the **next active clock edge**.

```verilog
always @ (posedge clk )
begin
	if (sync_reset)
		q <= 1'b0;
	else	
		q <= d;
end
```

**Key Characteristics:**
- Only clock in sensitivity list
- Reset synchronized to clock edge
- Eliminates reset-related metastability issues

### Asynchronous Set

An asynchronous set forces the flop's output to '1' **immediately**, regardless of the clock.

```verilog
always @(posedge clk or posedge async_set)
begin
    if (async_set)
        q <= 1'b1;  // Set takes immediate effect
    else
        q <= d;     // Normal operation
end
```

### Synchronous Set

A synchronous set forces the flop's output to '1' only on the **next active clock edge**.

```verilog
always @ (posedge clk)
begin
	if(sync_set)
		q <= 1'b1;
	else	
		q <= d;
end
```

### Common Yosys Synthesis Commands for Flip-Flops

The following sequence of commands is used to synthesize any of the flip-flop designs, map them to the library cells, and view the result. The `dfflibmap` command maps flip-flops (like DFFs, latches) in your RTL design to the equivalent sequential cells from a given standard cell library (.lib file).
Note: If the D-FF is not present in the existing library then we need to read the library in which D-FF is present.

```tcl
yosys> read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

yosys> read_verilog dff_asyncres.v

yosys> synth -top dff_asyncres

yosys> dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

yosys> abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib

yosys> show
```

---

## 4. Results

### Synthesis Netlists 📜

#### Hierarchical Netlist
The hierarchical netlist preserves the design's modularity. You can clearly see the instances of sub-modules being connected:
<img width="800" height="450" alt="multi_mod_hier" src="https://github.com/user-attachments/assets/06a76c66-1881-447c-a580-84f14a7f185b" />


#### Flatten Netlist  
The flatten netlist merges all logic into a single module. The original module boundaries are gone, replaced by a "flat" sea of logic gates:
<img width="800" height="450" alt="multi_mod_flat" src="https://github.com/user-attachments/assets/bd97dd14-907f-4220-85fb-e6f39232897b" />


### Waveform Analysis 📉

#### Asynchronous Reset Behavior
- **Output `q` goes to 0 instantly** when reset becomes active
- Reset effect is independent of clock edge

<img width="800" height="450" alt="waveform_dff_async_res" src="https://github.com/user-attachments/assets/0b2b2665-da5a-4c86-a1fb-c1e9e26742c1" />


#### Synchronous Reset Behavior  
- **Output `q` waits for the next rising clock edge** after reset becomes active before it goes to 0
- Reset effect synchronized with clock

<img width="800" height="450" alt="waveform_dff_sync_res" src="https://github.com/user-attachments/assets/080a9bf8-6da7-4f7d-8e89-e81d12e8c2ee" />


### Synthesized Flip-Flop Circuits 🔬

#### Asynchronous Reset Synthesis
- The reset signal is connected **directly to the dedicated asynchronous clear (CLR) pin** of the library's D-flip-flop cell
- Hardware reset capability utilized

<img width="800" height="450" alt="show_dff_asyncres" src="https://github.com/user-attachments/assets/3072e033-8c19-467d-b2eb-1006528460bd" />


#### Synchronous Reset Synthesis
- The reset signal is fed into **combinational logic (e.g., a multiplexer)** that precedes the D input of the flip-flop
- Reset functionality implemented in combinational logic rather than using dedicated hardware pins

<img width="1281" height="868" alt="show_dff_syncres" src="https://github.com/user-attachments/assets/d9713fac-9b65-4d04-b5ec-f244d11fe98e" />


---

## Key Takeaways

1. **Library Selection**: Choose appropriate timing libraries based on your target PVT corners
2. **Synthesis Strategy**: Use hierarchical for faster compilation and debugging, flatten for maximum optimization
3. **Reset Design**: Choose async reset for immediate response, sync reset for metastability-free operation
4. **Trade-offs**: Every design decision involves trade-offs between area, timing, power, and debuggability

Understanding these concepts is crucial for successful digital design synthesis and achieving optimal results in your ASIC or FPGA implementations.
