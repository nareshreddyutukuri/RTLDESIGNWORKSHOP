# Module 4 (Lab) — GLS, Synthesis-Simulation Mismatch & Verilog Statements

Hands-on labs for running gate-level simulation, deliberately reproducing a synthesis-simulation mismatch, and seeing exactly how blocking-assignment misuse breaks sequential logic.

## Contents
1. [Lab: Gate-Level Simulation (GLS)](#1-lab-gate-level-simulation-gls)
2. [Lab: Reproducing a Synthesis-Simulation Mismatch](#2-lab-reproducing-a-synthesis-simulation-mismatch)
3. [Lab: Blocking vs. Non-Blocking Behavior](#3-lab-blocking-vs-non-blocking-behavior)
4. [Lab: Blocking Statement Caveat (Shift Register Bug)](#4-lab-blocking-statement-caveat-shift-register-bug)

---

## 1. Lab: Gate-Level Simulation (GLS)

**Goal:** Simulate the synthesized gate-level netlist (instead of RTL) using the same testbench, and confirm the outputs still match expected behavior.

### Steps

```bash
# Step 1: Synthesize the RTL to get the gate-level netlist
yosys
read_liberty -lib /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog design.v
synth -top design
abc -liberty /path/to/sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr design_netlist.v
exit

# Step 2: Run GLS using iverilog with the netlist + gate models + testbench
iverilog -o gls_sim design_netlist.v tb_design.v \
    /path/to/sky130_fd_sc_hd.v \
    /path/to/primitives.v

# Step 3: Run the simulation
vvp gls_sim

# Step 4: View the waveform
gtkwave gls_sim.vcd
```

### Result

Gate-level simulation of a ternary-operator mux:

<img src="./images/waveform_of_ternary_op_mux_with_GLS.png" width="750" alt="GLS waveform of a ternary-operator mux">

RTL-level simulation of the same design, for comparison:

<img src="./images/ternary_op_mux_waveform.png" width="750" alt="RTL-level waveform of the same ternary-operator mux">

**What to check:** Compare this waveform against the RTL-level simulation waveform from Module 1 — for a correctly coded design, they should match exactly.

---

## 2. Lab: Reproducing a Synthesis-Simulation Mismatch

**Goal:** Deliberately write RTL with a common mismatch-causing bug (incomplete sensitivity list) and observe RTL sim vs. GLS diverge.

**Buggy design (`mux_bad.v`):**

```verilog
module mux_bad(input a, input b, input sel, output reg y);
    always @(sel) begin   // BUG: missing a, b from sensitivity list
        if (sel)
            y = a;
        else
            y = b;
    end
endmodule
```

### Steps

1. Run RTL simulation with `iverilog` + a testbench that toggles `a`/`b` without toggling `sel`.
   - **Observed:** `y` does **not** update when `a` or `b` changes (matches the buggy sensitivity list).
2. Synthesize with Yosys and run GLS on the resulting netlist with the same testbench.
   - **Observed:** `y` **does** update correctly, since real hardware (a mux) is inherently sensitive to all its inputs.

### Result

RTL sim (`y` stuck) vs. GLS (`y` correct) for the same stimulus, side by side:

<img src="./images/waveform_comparison_of_badmux.png" width="750" alt="Waveform comparison showing a synthesis-simulation mismatch caused by an incomplete sensitivity list">

**Conclusion:** RTL and gate-level simulation disagree — a textbook synthesis-simulation mismatch caused by an incomplete sensitivity list.

**Fix:** use `always @(*)` instead of `always @(sel)`.

---

## 3. Lab: Blocking vs. Non-Blocking Behavior

**Goal:** Compare a 2-stage shift register written with blocking vs. non-blocking assignments.

**Correct version (`shift_nonblocking.v`):**

```verilog
module shift_nonblocking(input clk, input d, output reg q1, output reg q2);
    always @(posedge clk) begin
        q1 <= d;
        q2 <= q1;
    end
endmodule
```

### Steps

```bash
iverilog -o sim_nb shift_nonblocking.v tb_shift.v
vvp sim_nb
gtkwave sim_nb.vcd
```

**What to check:** On each clock edge, `q2` should reflect the value `q1` held *before* that edge — a proper 2-cycle-delayed version of `d`.

---

## 4. Lab: Blocking Statement Caveat (Shift Register Bug)

**Goal:** Reproduce the classic blocking-assignment bug and see how it collapses a 2-stage shift register into effectively one stage.

**Buggy version (`shift_blocking.v`):**

```verilog
module shift_blocking(input clk, input d, output reg q1, output reg q2);
    always @(posedge clk) begin
        q1 = d;
        q2 = q1;   // BUG: picks up the NEW q1, not the old one
    end
endmodule
```

### Steps

```bash
iverilog -o sim_b shift_blocking.v tb_shift.v
vvp sim_b
gtkwave sim_b.vcd
```

### Result

RTL simulation of the buggy version — `q2` tracks `d` with only **1 cycle of delay** instead of 2, even though the code visually looks like a 2-stage shift register:

<img src="./images/waveform_of_blocking_caveat_with_GLS.png" width="750" alt="Waveform exposing the blocking-assignment shift register bug">

**What to check:** Compare this waveform against the non-blocking version from Lab 3.

**Follow-up:** Synthesize `shift_blocking.v` with Yosys and run GLS on it. Since real flip-flops physically cannot behave the way the blocking-assignment simulation suggests, the gate-level simulation shows the **correct 2-stage delay** — directly demonstrating a synthesis-simulation mismatch caused by blocking-statement misuse in sequential logic.

---

### Repo structure

```
.
├── README.md
└── images/
    ├── waveform_of_ternary_op_mux_with_GLS.png
    ├── ternary_op_mux_waveform.png
    ├── waveform_comparison_of_badmux.png
    └── waveform_of_blocking_caveat_with_GLS.png
```
