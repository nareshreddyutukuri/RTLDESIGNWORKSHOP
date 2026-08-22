# Module 3 — Logic Optimization Techniques

How synthesis tools shrink combinational and sequential logic without changing circuit behavior.

## Contents
1. [Combinational Logic Optimization](#1-combinational-logic-optimization)
2. [Sequential Logic Optimization](#2-sequential-logic-optimization)
3. [Sequential Optimization — Unused Outputs](#3-sequential-optimization--unused-outputs)

---

## 1. Combinational Logic Optimization

**What it is:** Simplifying purely combinational logic (no clock, no memory) down to the fewest gates possible, while preserving the exact same truth table.

Yosys and similar synthesis tools lean on two main techniques:

### Constant Propagation
If a gate's input is tied to a fixed `0` or `1`, the tool eliminates that gate entirely and propagates the constant forward through the rest of the logic.

<img width="1858" height="918" alt="Constant propagation example" src="https://github.com/user-attachments/assets/631dd07e-8260-47df-a4ca-290ae835d08a">

### Boolean Logic Minimization
Redundant logic gets reduced using techniques similar to K-map simplification — merging overlapping product terms, dropping don't-cares, and collapsing equivalent branches.

<img width="1790" height="949" alt="Boolean logic minimization example" src="https://github.com/user-attachments/assets/f2687873-8c78-465b-9173-8b5adc88c14e">

### Worked Example

```verilog
assign y = a ? b : (a ? c : d);
```

Both branches of the outer condition depend on `a` being true, so the tool recognizes `c` is unreachable and simplifies this to:

```verilog
assign y = a ? b : d;
```

Fewer muxes reach the final netlist — smaller area, less power, identical output for every input combination.

---

## 2. Sequential Logic Optimization

**What it is:** Optimization applied specifically to circuits containing flip-flops, where the circuit's *state over time* matters — not just instantaneous logic values.

| Technique | What it does |
|---|---|
| **State Optimization** | Removes unreachable FSM states, merges states that produce identical outputs for identical future input sequences |
| **Retiming** | Moves flip-flops across combinational boundaries (without changing behavior) to balance delay between pipeline stages and raise max clock frequency |
| **Sequential Cloning** | Duplicates a flip-flop that drives many high-fanout loads, splitting the loading/timing burden across copies instead of one overloaded flop |

### Worked Example — Retiming

Moving a flip-flop from *before* a long combinational path to *after* it is functionally equivalent (same overall latency) but can rebalance delay between two pipeline stages — enabling a higher clock frequency without altering circuit behavior.

<img width="1914" height="1079" alt="Retiming before" src="https://github.com/user-attachments/assets/5fc7c450-c16f-4f9f-95ab-b93c3a0d9e81">

<img width="1879" height="1024" alt="Retiming after" src="https://github.com/user-attachments/assets/05d35419-4658-4782-9f62-dd9477759159">

---

## 3. Sequential Optimization — Unused Outputs

**What it is:** A specific case of sequential optimization where the tool detects that certain flip-flop outputs (or register bits) never reach anywhere downstream in the design, and removes the associated hardware entirely.

**How it works:**
1. Trace every output bit forward through the design.
2. Identify flip-flops with no path to any primary output or other used logic.
3. Prune those unused flip-flops — and the logic feeding them — completely from the final netlist.

### Worked Example

```verilog
module counter(input clk, input rst, output [1:0] count);
    reg [3:0] full_count;
    always @(posedge clk or posedge rst)
        if (rst) full_count <= 0;
        else full_count <= full_count + 1;

    assign count = full_count[1:0]; // only lower 2 bits used
endmodule
```

`full_count[3:2]` never reaches the output port `count`. Yosys recognizes this during synthesis and **removes those two unused flip-flops and their incrementing logic entirely** — the synthesized circuit behaves as if only a 2-bit counter had been declared, even though the RTL describes a 4-bit one. Net result: less area, less power, zero change in observable behavior.
