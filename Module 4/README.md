# Module 4 — GLS, Synthesis-Simulation Mismatch & Verilog Statements

Why we simulate the gate-level netlist (not just the RTL), why RTL and gate-level behavior can disagree, and the Verilog assignment semantics that are usually the root cause.

## Contents
1. [What is GLS? Why GLS?](#1-what-is-gls-why-gls)
2. [Synthesis-Simulation Mismatch](#2-synthesis-simulation-mismatch)
3. [Blocking and Non-Blocking Statements in Verilog](#3-blocking-and-non-blocking-statements-in-verilog)
4. [Caveats with Blocking Statements](#4-caveats-with-blocking-statements)

---

## 1. What is GLS? Why GLS?

**What it is:** Gate-Level Simulation (GLS) runs the same testbench used for RTL simulation, but against the **synthesized gate-level netlist** instead of the original RTL code.

**How it works:**
- The gate-level netlist (Yosys/synthesis output) is simulated using the same stimulus as the RTL testbench.
- Often a **timing-annotated** netlist (via an SDF file) is used, so real gate delays are modeled — not just functional behavior.

**Why we need it:**
- **Verifies synthesis correctness** — confirms the synthesis tool didn't introduce functional bugs while converting RTL into gates.
- **Catches synthesis-simulation mismatches** — some RTL constructs simulate one way at the RTL level but synthesize into hardware that behaves differently; GLS catches these before tape-out.
- **Timing verification** — with delay annotation, GLS can reveal race conditions and hold-time violations that a zero-delay RTL simulation would never show.

### Example: ternary-operator mux under GLS

A simple `y = sel ? i1 : i0` mux, simulated post-synthesis:

<img src="./images/waveform_of_ternary_op_mux_with_GLS.png" width="750" alt="GLS waveform of a ternary-operator mux">

Same design, RTL-level simulation for comparison — note the timing/behavior lines up with the gate-level run above:

<img src="./images/ternary_op_mux_waveform.png" width="750" alt="RTL-level waveform of the same ternary-operator mux">

---

## 2. Synthesis-Simulation Mismatch

**What it is:** RTL simulation and gate-level (post-synthesis) simulation produce **different results** for the same testbench, even though both were derived from the "same" design.

**Why it happens:**
- **Incomplete sensitivity lists** — missing signals in an `always @(...)` list can make an RTL simulator behave differently than synthesized hardware, which is always "fully sensitive" to its real inputs.
- **Blocking vs. non-blocking misuse** — the wrong assignment type in sequential or combinational blocks can make the simulator model a different operation order than what hardware actually does.
- **Non-synthesizable constructs** — `initial` blocks, `#10`-style delays, or certain loop constructs simulate fine at RTL but are ignored or reinterpreted by the synthesis tool, since real hardware has no concept of "simulation time."
- **Ambiguous/incomplete coding** — incomplete `if-else` or `case` statements can infer unintended latches during synthesis, a mismatch RTL simulation won't reveal but GLS will.

**Why it matters:** left uncaught, this means your chip's *actual physical behavior* diverges from what you verified in simulation — an expensive thing to discover after fabrication.

**Important caveat:** GLS is a diagnostic checkpoint, not a fix. If GLS and RTL sim disagree, the RTL is buggy and must be corrected — then both simulations are re-run to confirm agreement. GLS should never be relied on to "fix" or compensate for incorrect RTL: it's slow, comes late in the flow, and doesn't guarantee every RTL bug surfaces as a mismatch (some bugs synthesize into equally wrong hardware).

### Example: a "bad mux" mismatch

Comparing the flagged waveform against the annotated signal values at the mismatch point (`i0=0, i1=1, sel=0, y=1` — wrong for this input combination):

<img src="./images/waveform_comparison_of_badmux.png" width="750" alt="Waveform comparison showing a synthesis-simulation mismatch in a bad mux">

---

## 3. Blocking and Non-Blocking Statements in Verilog

Verilog provides two kinds of procedural assignment, and picking the right one is critical for both correct simulation and correct synthesis.

### Blocking Assignment (`=`)
- Executes **sequentially**, in the exact order written — the next statement waits ("blocks") until the current one finishes.
- Intended for modeling **combinational logic**.

```verilog
always @(*) begin
    y = a & b;
    z = y | c;   // uses the just-updated value of y
end
```

### Non-Blocking Assignment (`<=`)
- All right-hand sides are evaluated first, and **all assignments happen simultaneously** at the end of the time step — mimicking how flip-flops physically update on a clock edge.
- Intended for modeling **sequential (clocked) logic**.

```verilog
always @(posedge clk) begin
    q1 <= d;
    q2 <= q1;   // uses q1's value BEFORE this clock edge, not after
end
```

This models a correct 2-stage shift register, because `q2` receives the *old* value of `q1` — exactly like real flip-flops updating in parallel.

---

## 4. Caveats with Blocking Statements

Using blocking assignments (`=`) in the wrong context is one of the most common causes of synthesis-simulation mismatch.

### Using blocking in sequential logic

```verilog
always @(posedge clk) begin
    q1 = d;
    q2 = q1;   // BUG: uses the just-updated q1, not the old value
end
```

Because blocking assignments execute immediately and sequentially, `q2` incorrectly gets the **new** value of `q1` on the same clock edge — collapsing what should be a 2-stage shift register into effectively a 1-stage delay in simulation. Real flip-flop hardware can never behave this way, so RTL simulation diverges from the actual synthesized gate-level behavior:

<img src="./images/waveform_of_blocking_caveat_with_GLS.png" width="750" alt="GLS waveform exposing the blocking-assignment caveat">

### Key points

- **Hardware doesn't "know" about blocking/non-blocking.** These are simulation-language semantics only. Real flip-flops always capture input at the clock edge regardless of which operator was used in the RTL — so a blocking-assignment bug produces a *wrong RTL simulation*, while GLS reveals the *true* hardware behavior. The mismatch is the signal to go fix the RTL, not something to depend on.
- **Order-dependent bugs.** Since blocking statements execute top-to-bottom, simply reordering two blocking statements inside an `always` block can silently change simulation results — a fragile, error-prone pattern for sequential logic.

### Rule of thumb

| Logic Type | Assignment to Use |
|---|---|
| Combinational (`always @(*)`) | Blocking (`=`) |
| Sequential (`always @(posedge clk)`) | Non-Blocking (`<=`) |

Mixing these up is one of the top sources of synthesis-simulation mismatches in real RTL designs.

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
