# Day 1: Introduction to Verilog RTL Design & Synthesis

This document covers the fundamental tools and concepts for designing, simulating, and synthesizing digital circuits using Verilog.

## Table of Contents

1. [Introduction to iverilog and gtkwave](#1-introduction-to-iverilog-and-gtkwave)
2. [Labs using iverilog and gtkwave (2:1 Mux)](#2-labs-using-iverilog-and-gtkwave-21-mux)
3. [Verilog files used](#3-verilog-files-used)
4. [Introduction to yosys and logic synthesis](#4-introduction-to-yosys-and-logic-synthesis)
5. [Labs using yosys and sky130 PDK](#5-labs-using-yosys-and-sky130-pdk)
6. [Results of both labs](#6-results-of-both-labs)

---

## 1. Introduction to iverilog and gtkwave

To verify our digital designs, we use a combination of Verilog code and specialized tools. The two primary code components are the **Design** and the **Testbench**.

### Design
The Design is the actual Verilog code describing the hardware module you want to create. It contains the logic with specific inputs and outputs to perform a required function.

### Testbench (TB)
The Testbench is a separate Verilog module written specifically to verify the Design. It's a non-synthesizable piece of code that acts as a testing environment. Its main jobs are to:

- **Generate Stimulus**: Create and apply input signals to the Design's ports
- **Observe Outputs**: Capture the Design's outputs to check if they are correct

<img width="800" height="450" alt="tb_block_dig" src="https://github.com/user-attachments/assets/a15076c4-bd96-4571-a243-d309509eba80" />

### The Simulator ⚙️

A simulator is a software tool used to execute the testbench and model the behavior of the design. It works on an **event-driven basis**. This means the simulator only performs calculations when it detects a change on an input signal. If there are no input changes, the outputs remain the same, which is a very efficient approach for simulating digital logic.

### Simulation Flow using iverilog

1. **Input**: The Design and Testbench Verilog files are fed into iverilog
2. **Simulation**: iverilog compiles the code and runs the simulation, generating a Value Change Dump (.vcd) file that logs all signal activity
3. **Viewing**: A waveform viewer like gtkwave opens the .vcd file to display the signals as graphical waveforms for analysis

<img width="800" height="450" alt="iverilog_flow" src="https://github.com/user-attachments/assets/762e378d-dc47-48da-afe6-100860426be2" />

---

## 2. Labs using iverilog and gtkwave (2:1 Mux)

### Step 1: Lab Setup 📂

First, clone the required workshop repository to get the necessary files. The tools iverilog and gtkwave were installed during the Week 0 setup.

```bash
git clone https://github.com/kunalg123/sky130RTLDesignAndSynthesisWorkshop.git
cd sky130RTLDesignAndSynthesisWorkshop/
```

### Step 2: Running Icarus Verilog Simulation 🚀

Use the iverilog command to compile your Verilog design and testbench files. If design contains of multiple files then we have to mention all the files. The a.out file is generated. The './a.out' command is used to run the compiled simulation, which generates the .vcd waveform file.

```bash
iverilog good_mux.v tb_good_mux.v
./a.out
```

### Step 3: Viewing the Waveform in GTKWave 📉

Finally, use gtkwave to open the generated .vcd file and visualize the waveforms to verify the design's behavior.

```bash
gtkwave tb_good_mux.vcd
```

---

## 3. Verilog Files Used

### Design: 2-to-1 Multiplexer (good_mux.v)

```verilog
module good_mux (input i0, input i1, input sel, output reg y);
always @ (*)
begin
    if(sel)
        y <= i1;
    else 
        y <= i0;
end
endmodule
```

### Testbench: (mux_2_1_tb.v)

```verilog
`timescale 1ns / 1ps
module tb_good_mux;
	// Inputs
	reg i0,i1,sel;
	// Outputs
	wire y;

        // Instantiate the Unit Under Test (UUT)
	good_mux uut (
		.sel(sel),
		.i0(i0),
		.i1(i1),
		.y(y)
	);

	initial begin
	$dumpfile("tb_good_mux.vcd");
	$dumpvars(0,tb_good_mux);
	// Initialize Inputs
	sel = 0;
	i0 = 0;
	i1 = 0;
	#300 $finish;
	end

always #75 sel = ~sel;
always #10 i0 = ~i0;
always #55 i1 = ~i1;
endmodule
```

### How a 2:1 Mux Works

A 2-to-1 multiplexer (mux) is a simple digital circuit that selects one of two data inputs and forwards it to a single output.

- **Inputs**: It has two data inputs, `i0` and `i1`
- **Select Line**: A single control line, `sel`, determines which input is chosen
- **Output**: A single output, `y`

The logic is straightforward:
- If `sel` is 0, the output `y` is equal to input `i0`
- If `sel` is 1, the output `y` is equal to input `i1`

---

## 4. Introduction to yosys and logic synthesis

### From RTL to Gates: What is Synthesis? 🤖

**Synthesis** is the process of converting an abstract, high-level hardware description, like our Verilog RTL code, into a physical implementation in the form of a gate-level netlist. This netlist is a detailed description of how to build the circuit using basic logic gates (like AND, OR, NOT) and flip-flops from a specific technology library.

**Yosys** is a powerful, open-source synthesis tool that we use to perform this automated conversion. It takes our design and constraints and maps them to a library of standard cells.

### Yosys setup
<img width="800" height="450" alt="Screenshot (299)" src="https://github.com/user-attachments/assets/bab33456-9d32-4c0e-96d6-b53e8e1e13b5" />


### Verification of Synthesis

After synthesis, it's crucial to verify that the synthesized netlist behaves identically to the original RTL code. This is typically done by running the same testbench on both the RTL design and the synthesized netlist.

<img width="800" height="450" alt="Screenshot (298)" src="https://github.com/user-attachments/assets/9ea9beb2-fe0d-4726-ac56-af6183947aa3" />


### Logic Synthesis and Standard Cell Libraries 📚

The synthesis tool needs a menu of available components to build the circuit. This is provided by a technology library file.

#### What is a .lib file?

The `.lib` file (a Liberty file) is a standard cell library provided by the foundry (like SkyWater for the SKY130 PDK). It contains detailed information about every available logic gate, such as:
- Logical function
- Timing characteristics (delay, rise/fall times)
- Power consumption
- Physical area

#### Why have cells with the same function but different characteristics?

The `.lib` file contains multiple versions of the same logic gate (e.g., a fast AND gate and a slow AND gate) to help the synthesis tool meet timing constraints. This is crucial for fixing timing violations found during setup and hold analysis:

- **Setup Analysis**: Checks if data is stable before the clock edge arrives at a flip-flop. A setup violation occurs if the data path has higher delay
- **Hold Analysis**: Checks if data remains stable after the clock edge has passed. A hold violation occurs if the data path is too fast and the new data arrives before the old data has been properly captured

#### Which cells are used?

The synthesis tool strategically selects cells based on the design's constraints:
- **Cells with smaller delay** (faster, but often larger and more power-hungry) are used on critical paths to fix setup violations
- **Cells with larger delay** (slower, but smaller and power-efficient) can be used on non-critical paths or to intentionally slow down a path to fix a hold violation

---

## 5. Labs using yosys and sky130 PDK

This section outlines the basic command flow to synthesize a Verilog design into a gate-level netlist using Yosys. These commands are typically run inside the Yosys interactive shell.

### Step 1: Start yosys

This command starts the yosys synthesizer.

```tcl
yosys
```

### Step 2: Load the Standard Cell Library

This command reads the Liberty (.lib) file, which contains the definitions of all the standard cells the synthesizer can use to build the circuit.

```tcl
yosys> read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

### Step 3: Read the Verilog Design

This command reads your RTL Verilog design file into the synthesis tool.

```tcl
yosys> read_verilog good_mux.v
```

### Step 4: Run High-Level Synthesis

This is the main synthesis command. It converts the generic Verilog RTL into a circuit of generic logic components. You must specify the name of the top-level module in your design.

```tcl
yosys> synth -top good_mux
```

### Step 5: Map to the Technology Library

The `abc` command performs technology mapping. It takes the generic logic circuit from the previous step and maps it to the specific standard cells that were loaded from the .lib file.

```tcl
yosys> abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

### Step 6: View the Synthesized Netlist

The `show` command generates a graphical representation of the gate-level netlist, allowing you to visualize the synthesized circuit. If multiple modules are present we need to mention the module which we want to look into.

```tcl
yosys> show
```

### Step 7: Write the Gate-Level Verilog

This command saves the final synthesized circuit as a new Verilog file. This gate-level netlist describes the design as an interconnection of standard cells and is the input for the next stage of the physical design process (place and route).

```tcl
yosys> write_verilog -noattr good_mux_netlist.v
```
---
## 6. Results of both labs

`good_mux.v` gtkwave waveform:

<img width="800" height="450" alt="good_mux_waveform" src="https://github.com/user-attachments/assets/49679926-ef4a-4230-8901-e1921bb0bcb1" />


`good_mux.v` synthesized output:

<img width="800" height="450" alt="show_good_mux" src="https://github.com/user-attachments/assets/0d74e63e-1bc7-4441-92aa-8b2176558759" />


`good_mux_netlist` Netlist:

<img width="800" height="450" alt="good_mux_netlist" src="https://github.com/user-attachments/assets/e1c12e44-e322-4f14-a01d-6261848f52c0" />

### Note: The synthesized output from the workshop video and the result obtained is different due to the updation of the Yosys and the standard cell library.

---

## Summary

This guide covered the essential workflow for Verilog RTL design and synthesis:

1. **Design Creation**: Writing Verilog RTL code and testbenches
2. **Simulation**: Using iverilog and gtkwave for functional verification
3. **Synthesis**: Converting RTL to gate-level netlists using yosys
4. **Technology Mapping**: Mapping to specific standard cell libraries
