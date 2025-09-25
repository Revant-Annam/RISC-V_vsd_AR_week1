# Day 1: Introduction to Verilog RTL Design & Synthesis

This document covers the fundamental tools and concepts for designing, simulating, and synthesizing digital circuits using Verilog.

## Table of Contents

1. [Introduction to iverilog and gtkwave](#1-introduction-to-iverilog-and-gtkwave)
2. [Labs using iverilog and gtkwave (2:1 Mux)](#2-labs-using-iverilog-and-gtkwave-21-mux)
3. [Verilog files used](#3-verilog-files-used)
4. [Introduction to yosys and logic synthesis](#4-introduction-to-yosys-and-logic-synthesis)
5. [Labs using yosys and sky130 PDK](#5-labs-using-yosys-and-sky130-pdk)

---

## 1. Introduction to iverilog and gtkwave

To verify our digital designs, we use a combination of Verilog code and specialized tools. The two primary code components are the **Design** and the **Testbench**.

### Design
The Design is the actual Verilog code describing the hardware module you want to create. It contains the logic with specific inputs and outputs to perform a required function.

### Testbench (TB)
The Testbench is a separate Verilog module written specifically to verify the Design. It's a non-synthesizable piece of code that acts as a testing environment. Its main jobs are to:

- **Generate Stimulus**: Create and apply input signals to the Design's ports
- **Observe Outputs**: Capture the Design's outputs to check if they are correct

### The Simulator ⚙️

A simulator is a software tool used to execute the testbench and model the behavior of the design. It works on an **event-driven basis**. This means the simulator only performs calculations when it detects a change on an input signal. If there are no input changes, the outputs remain the same, which is a very efficient approach for simulating digital logic.

### Simulation Flow using iverilog

1. **Input**: The Design and Testbench Verilog files are fed into iverilog
2. **Simulation**: iverilog compiles the code and runs the simulation, generating a Value Change Dump (.vcd) file that logs all signal activity
3. **Viewing**: A waveform viewer like gtkwave opens the .vcd file to display the signals as graphical waveforms for analysis

---

## 2. Labs using iverilog and gtkwave (2:1 Mux)

### Step 1: Lab Setup 📂

First, clone the required workshop repository to get the necessary files. We will assume that iverilog and gtkwave were installed during the Week 0 setup.

```bash
git clone https://github.com/kunalg123/sky130RTLDesignAndSynthesisWorkshop.git
cd sky130RTLDesignAndSynthesisWorkshop/
```

### Step 2: Running Icarus Verilog Simulation 🚀

Use the iverilog command to compile your Verilog design and testbench files. The `-o` flag specifies the output file name. Then, use the `vvp` command to run the compiled simulation, which generates the .vcd waveform file.

```bash
iverilog -o mux_2_1_tb mux_2_1.v mux_2_1_tb.v
vvp mux_2_1_tb
```

### Step 3: Viewing the Waveform in GTKWave 📉

Finally, use gtkwave to open the generated .vcd file and visualize the waveforms to verify the design's behavior.

```bash
gtkwave mux2_1.vcd
```

---

## 3. Verilog Files Used

### Design: 2-to-1 Multiplexer (mux_2_1.v)

```verilog
module mux_2_1(a, b, sel, y);
    input a, b, sel;
    output y;
    assign y = sel ? b : a;
endmodule
```

### Testbench: (mux_2_1_tb.v)

```verilog
`timescale 1ns/1ps
`include "mux_2_1.v"

module mux_2_1_tb;
    reg a, b, sel;
    wire y;

    mux_2_1 M1(a, b, sel, y);

    initial begin
        $dumpfile("mux2_1.vcd");
        $dumpvars(0, mux_2_1_tb);

        a=0; b=0; sel=0; #10;
        a=0; b=1; sel=0; #10;
        a=1; b=0; sel=0; #10;
        a=1; b=1; sel=0; #10;
        a=0; b=0; sel=1; #10;
        a=0; b=1; sel=1; #10;
        a=1; b=0; sel=1; #10;
        a=1; b=1; sel=1; #10;
        $finish;
    end
endmodule
```

### How a 2:1 Mux Works

A 2-to-1 multiplexer (mux) is a simple digital circuit that selects one of two data inputs and forwards it to a single output.

- **Inputs**: It has two data inputs, `a` and `b`
- **Select Line**: A single control line, `sel`, determines which input is chosen
- **Output**: A single output, `y`

The logic is straightforward:
- If `sel` is 0, the output `y` is equal to input `a`
- If `sel` is 1, the output `y` is equal to input `b`

---

## 4. Introduction to yosys and logic synthesis

### From RTL to Gates: What is Synthesis? 🤖

**Synthesis** is the process of converting an abstract, high-level hardware description, like our Verilog RTL code, into a physical implementation in the form of a gate-level netlist. This netlist is a detailed description of how to build the circuit using basic logic gates (like AND, OR, NOT) and flip-flops from a specific technology library.

**Yosys** is a powerful, open-source synthesis tool that we use to perform this automated conversion. It takes our design and constraints and maps them to a library of standard cells.

### Verification of Synthesis

After synthesis, it's crucial to verify that the synthesized netlist behaves identically to the original RTL code. This is typically done by running the same testbench on both the RTL design and the synthesized netlist.

### Logic Synthesis and Standard Cell Libraries 📚

The synthesis tool needs a "menu" of available components to build the circuit. This is provided by a technology library file.

#### What is a .lib file?

The `.lib` file (a Liberty file) is a standard cell library provided by the foundry (like SkyWater for the SKY130 PDK). It contains detailed information about every available logic gate, such as:
- Logical function
- Timing characteristics (delay, rise/fall times)
- Power consumption
- Physical area

#### Why have cells with the same function but different characteristics?

The `.lib` file contains multiple versions of the same logic gate (e.g., a fast AND gate and a slow AND gate) to help the synthesis tool meet timing constraints. This is crucial for fixing timing violations found during setup and hold analysis:

- **Setup Analysis**: Checks if data is stable before the clock edge arrives at a flip-flop. A setup violation occurs if the data path is too slow
- **Hold Analysis**: Checks if data remains stable after the clock edge has passed. A hold violation occurs if the data path is too fast and the new data arrives before the old data has been properly captured

#### Which cells are used?

The synthesis tool strategically selects cells based on the design's constraints:
- **Cells with smaller delay** (faster, but often larger and more power-hungry) are used on critical paths to fix setup violations
- **Cells with larger delay** (slower, but smaller and power-efficient) can be used on non-critical paths or to intentionally slow down a path to fix a hold violation

---

## 5. Labs using yosys and sky130 PDK

This section outlines the basic command flow to synthesize a Verilog design into a gate-level netlist using Yosys. These commands are typically run inside the Yosys interactive shell.

### Step 1: Load the Standard Cell Library

This command reads the Liberty (.lib) file, which contains the definitions of all the standard cells the synthesizer can use to build the circuit.

```tcl
yosys> read_liberty -lib <path_to_lib_file>
```

### Step 2: Read the Verilog Design

This command reads your RTL Verilog design file into the synthesis tool.

```tcl
yosys> read_verilog mux_2_1.v
```

### Step 3: Run High-Level Synthesis

This is the main synthesis command. It converts the generic Verilog RTL into a circuit of generic logic components. You must specify the name of the top-level module in your design.

```tcl
yosys> synth -top mux_2_1
```

### Step 4: Map to the Technology Library

The `abc` command performs technology mapping. It takes the generic logic circuit from the previous step and maps it to the specific standard cells that were loaded from the .lib file.

```tcl
yosys> abc -liberty <path_to_lib_file>
```

### Step 5: View the Synthesized Netlist

The `show` command generates a graphical representation of the gate-level netlist, allowing you to visualize the synthesized circuit.

```tcl
yosys> show
```

### Step 6: Write the Gate-Level Verilog

This command saves the final synthesized circuit as a new Verilog file. This gate-level netlist describes the design as an interconnection of standard cells and is the input for the next stage of the physical design process (place and route).

```tcl
yosys> write_verilog -noattr mux_2_1_netlist.v
```

---

## Summary

This guide covered the essential workflow for Verilog RTL design and synthesis:

1. **Design Creation**: Writing Verilog RTL code and testbenches
2. **Simulation**: Using iverilog and gtkwave for functional verification
3. **Synthesis**: Converting RTL to gate-level netlists using yosys
4. **Technology Mapping**: Mapping to specific standard cell libraries

The next steps in the digital design flow would typically include physical design (place and route), timing analysis, and final verification before tape-out.

---

*This document is part of the SKY130 RTL Design and Synthesis Workshop series.*
