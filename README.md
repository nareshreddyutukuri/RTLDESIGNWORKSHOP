# Module 1: Core Concepts & Tools

**Contents**
1. Design, Testbench, and Simulator
2. Icarus Verilog
3. The "Good Mux" Lab
4. RTL vs. Gate-Level Netlist
5. What is Synthesis?
6. What is Yosys?
7. What Can We Do With Yosys?

This section covers the foundational concepts, files, and tools used across this chip design program.

---

## 1. Design, Testbench, and Simulator

Before a digital circuit is manufactured, it has to be verified through simulation. This flow has three main pieces:

- **Design (DUT – Design Under Test):** The Verilog RTL code that describes the logic and behavior of the hardware we're building.
- **Testbench:** A separate Verilog file that drives the design with stimulus (input signals) and checks the outputs against expected behavior. It is never synthesized into hardware — it exists only to verify the design.
- **Simulator:** The tool that runs the design and testbench together over time, modeling how the hardware behaves. It produces a waveform file (`.vcd`) so signal transitions can be inspected visually.

---

## 2. Icarus Verilog (iverilog)

`iverilog` is a free, open-source Verilog simulator that functions as the compiler in this workflow. It compiles the design and testbench files together into an executable binary. Running that binary generates the Value Change Dump (`.vcd`) file, which is then viewed in **GTKWave**.

---

## 3. The "Good Mux" Lab

`good_mux.v` — a simple 2:1 multiplexer — is the introductory exercise used to walk through the full end-to-end flow:

1. Write the RTL behavior.
2. Write a testbench to toggle the select line and inputs.
3. Simulate with `iverilog` and verify the waveform.
4. Run synthesis to see how the behavioral mux maps to standard logic gates.

---

## 4. RTL vs. Gate-Level Netlist

As a design moves from concept to implementation, it goes through two very different representations:

| Feature | RTL (Register Transfer Level) | Gate-Level Netlist |
|---|---|---|
| **What it is** | Human-readable Verilog describing *what* the circuit should do (behavior and data flow) | A structural file describing *how* the circuit is built, using actual standard cells (AND, OR, flip-flops) |
| **Technology** | Technology-independent — reusable across manufacturing processes | Technology-dependent — mapped to a specific library (e.g., SkyWater 130nm PDK) |
| **Creation** | Written by the RTL designer | Generated automatically by the synthesis tool |
| **Readability** | High — uses `if-else`, `always` blocks, `assign` | Low — a long list of interconnected gate instances |

---

## 5. What Is Synthesis?

Synthesis (logic synthesis) is the automated process that converts abstract, human-readable RTL code into a structural gate-level netlist.

A useful analogy: just as a software compiler turns C++ into machine code a processor can execute, a synthesis tool turns Verilog into a network of physical logic gates that can actually be fabricated on silicon.

---

## 6. What Is Yosys?

**Yosys** is an open-source framework for RTL synthesis. Where `iverilog` *verifies* the logic, Yosys *builds* it — it's the bridge between behavioral code and physical gates.

---

## 7. What Can We Do With Yosys?

Yosys handles several key steps in the VLSI frontend flow:

- **RTL Parsing** — reads and interprets the Verilog RTL.
- **Logic Optimization** — simplifies Boolean expressions, removes redundant logic, and reduces area/power.
- **Technology Mapping** — maps the optimized logic onto physical standard cells from a Process Design Kit (e.g., `sky130_fd_sc_hd`).
- **Netlist Generation** — outputs the final gate-level netlist (`.v` file) for the physical design/layout team.
- **Visualizing Logic** — the `show` command generates a graphical schematic so you can inspect how the code mapped to hardware.
