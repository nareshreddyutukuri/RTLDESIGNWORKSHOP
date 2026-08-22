# Module 3 (Lab) — Logic Optimization Techniques

Hands-on Yosys flow for observing combinational and sequential logic optimization on the SKY130 PDK.

## Contents
1. [Lab: Combinational Logic Optimization](#1-lab-combinational-logic-optimization)
2. [Lab: Sequential Logic Optimization](#2-lab-sequential-logic-optimization)
3. [Lab: Sequential Optimization — Unused Outputs](#3-lab-sequential-optimization--unused-outputs)

---

## 1. Lab: Combinational Logic Optimization

**Goal:** See how Yosys collapses redundant combinational logic (nested ternary/mux chains, chained inverters, etc.) down to a minimal gate count.

### Background — what `opt_check` and `opt_check2` reduce to

A ternary like `y = a ? b : 0` reduces to a single AND gate, and a double-inverted mux like `y = !a ? !b : a` reduces to an OR gate. Working through the Boolean algebra by hand before running synthesis makes it much easier to sanity-check the tool's output:

<img src="./images/opt_check_theory.png" width="600" alt="Hand-derived Boolean reduction for opt_check and opt_check2">

### Steps

```bash
# Launch Yosys
yosys

# Read the liberty file (standard cell library)
read_liberty -lib /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# Read your RTL design
read_verilog comb_opt.v

# Set the top module
synth -top comb_opt

# Run the optimization pass
opt_clean

# Map to standard cells
abc -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# View the synthesized schematic
show
```

### Result: `opt_check`

`y = a ? b : 0` synthesizes straight down to a single 2-input AND gate — exactly matching the hand-derived `y = ab`:

<img src="./images/opt_check1_schematic.png" width="500" alt="opt_check synthesized schematic — single AND gate">

### Result: `opt_check2`

The mux-based description collapses into inverters feeding a NAND2, realizing `y = a + b` with minimal cells:

<img src="./images/opt_check2_schematic.png" width="650" alt="opt_check2 synthesized schematic — inverters into NAND2">

### Result: `opt_check3` (3-input case)

Extending the pattern to a third input (`a`, `b`, `c`) still reduces cleanly to a single AND3 cell:

<img src="./images/opt_check3_schematic.png" width="450" alt="opt_check3 synthesized schematic — single AND3 gate">

**`stat` output for `opt_check3`** — confirms the final netlist has just 2 cells (`$ANDNOT_`, `$NAND_`) mapped down from the original RTL:

<img src="./images/opt_check3_statistics.png" width="500" alt="Yosys stat output for opt_check3 showing final cell count">

**What to check:** Compare the `stat` output (cell count) before and after `opt_clean` — you should see fewer mux/logic cells in the final netlist than in the naive RTL.

---

## 2. Lab: Sequential Logic Optimization

**Goal:** Observe retiming/state optimization on a sequential design (e.g. an FSM or pipelined register chain).

### Steps

```bash
yosys

read_liberty -lib /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog seq_opt.v
synth -top seq_opt

# Sequential optimization pass
opt -full

abc -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

**What to check:** Use `stat` to compare flip-flop count before/after — equivalent states or redundant flops should collapse into fewer sequential elements.

---

## 3. Lab: Sequential Optimization — Unused Outputs

**Goal:** Confirm Yosys removes flip-flops whose outputs never reach a primary output.

**Design file (`counter.v`):**

```verilog
module counter(input clk, input rst, output [1:0] count);
    reg [3:0] full_count;
    always @(posedge clk or posedge rst)
        if (rst) full_count <= 0;
        else full_count <= full_count + 1;

    assign count = full_count[1:0]; // only lower 2 bits used
endmodule
```

### Steps

```bash
yosys

read_liberty -lib /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog counter.v
synth -top counter

opt_clean -purge

abc -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib

# Check final flip-flop count
stat

show
```

**What to check:** Run `stat` and confirm only **2 flip-flops** remain in the final netlist (not 4) — proving `full_count[3:2]` and its logic were pruned since they never reach the `count` output.

---

### Repo structure

```
.
├── README.md
└── images/
    ├── opt_check_theory.png
    ├── opt_check1_schematic.png
    ├── opt_check2_schematic.png
    ├── opt_check3_schematic.png
    └── opt_check3_statistics.png
```
