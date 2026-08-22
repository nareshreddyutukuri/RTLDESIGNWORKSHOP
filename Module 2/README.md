# Module 2 — Timing Libraries & Synthesis Strategies

A practical rundown of `.lib` files, synthesis approaches (hierarchical vs. flat), why sequential elements matter, reset styles, and a couple of hardware-efficient arithmetic tricks.

## Table of Contents
1. [What's Inside a `.lib` File?](#1-whats-inside-a-lib-file)
2. [Standard Cell Flavours — Why They Exist](#2-standard-cell-flavours--why-they-exist)
3. [`timing.lib` Explained](#3-timinglib-explained)
4. [Hierarchical Synthesis](#4-hierarchical-synthesis)
5. [Flat Synthesis](#5-flat-synthesis)
6. [Hierarchical vs. Flat — Side by Side](#6-hierarchical-vs-flat--side-by-side)
7. [Choosing the Right Strategy](#7-choosing-the-right-strategy)
8. [Why Flip-Flops Matter](#8-why-flip-flops-matter)
9. [Synchronous vs. Asynchronous Reset](#9-synchronous-vs-asynchronous-reset)
10. [Fast Math: Multiplying by Powers of Two](#10-fast-math-multiplying-by-powers-of-two)

---

## 1. What's Inside a `.lib` File?

A `.lib` (Liberty format) file is a plain-text database, supplied by the foundry, that describes every standard cell in a technology — AND, OR, MUX, flip-flops, and so on. Key contents include:

| Category | Description |
|---|---|
| **Functionality** | The Boolean behavior of each cell |
| **Area** | Physical silicon footprint |
| **Power** | Leakage and dynamic power draw |
| **Timing** | Delay, setup time, hold time for sequential cells |
| **PVT Corners** | Characterization at Process/Voltage/Temperature extremes (e.g., typical, fast-fast, slow-slow) |

<img width="868" height="405" alt="lib file contents" src="https://github.com/user-attachments/assets/305c24ad-7abf-4d46-a97a-28f909f22909" />

---

## 2. Standard Cell Flavours — Why They Exist

The same logic gate typically ships in several drive-strength variants (e.g., a 1x, 2x, 4x, 8x AND gate):

- **Larger/faster cells** — wider transistors, drive heavier capacitive loads with less delay. Best for timing-critical paths.
- **Smaller/slower cells** — less area and leakage, but higher delay. Best for paths with slack, to save power and area.

Synthesis tools pick the right flavour per path based on the timing budget.

---

## 3. `timing.lib` Explained

`timing.lib` is often just shorthand for `.lib`, but the name draws attention to its timing-model content specifically. It supplies delay lookup tables (typically Non-Linear Delay Models, NLDM) so the synthesis tool can compute gate delay from two inputs:

1. **Input slew** — how fast the input transitions (0→1 or 1→0)
2. **Output load** — total capacitance the output is driving (wires + fan-out gates)

Delay values shift depending on operating conditions/corner, which is why multiple `.lib` files exist per cell library.

<img width="1645" height="625" alt="timing lookup" src="https://github.com/user-attachments/assets/8f4aba5e-e20d-4aec-b6a5-a68a2ca633bf" />

---

## 4. Hierarchical Synthesis

The tool keeps your RTL's module boundaries intact in the final netlist.

- A top module instantiating several sub-modules stays structured that way after synthesis.
- Schematics show distinct "boxes" per sub-module, wired together.
- **Best for:** large SoCs, multi-team projects, and keeping netlists readable for debug.

<img width="781" height="452" alt="hierarchical synthesis" src="https://github.com/user-attachments/assets/aca507ce-243b-4bd5-9de8-d179d8a35cd8" />

---

## 5. Flat Synthesis

The tool erases module boundaries, merging everything into one continuous block of logic.

- In Yosys, the `flatten` command pulls all gates out of sub-modules into the top module and discards the empty shells.
- **Analogy:** it's like emptying every subfolder into the parent folder, then deleting the now-empty subfolders.
- **Best for:** smaller designs, where cross-boundary optimization yields the best area/timing.

---

## 6. Hierarchical vs. Flat — Side by Side

| | Hierarchical | Flat |
|---|---|---|
| Module boundaries | Preserved | Dissolved |
| Netlist structure | Mirrors RTL | Single merged block |
| Optimization scope | Per-module | Whole-design |
| Debuggability | Easier | Harder |

<img width="1157" height="752" alt="hierarchical vs flat" src="https://github.com/user-attachments/assets/96bf889f-88b2-4b0f-8fe6-58f68fb979c3" />

---

## 7. Choosing the Right Strategy

**Go hierarchical when:**
- The design is a large, complex SoC
- Multiple teams own different blocks
- Debugging ease matters more than squeezing out every last bit of optimization

**Go flat when:**
- The design (or block) is small
- You want maximum cross-boundary optimization — logic sharing and timing gains across former module lines

---

## 8. Why Flip-Flops Matter

Digital logic splits into combinational and sequential categories. Flip-flops (sequential elements) earn their place for two reasons:

- **Memory:** Combinational logic (AND/OR/etc.) has no state — output depends purely on current input. Counters, state machines, and processors all need something that *remembers*, which is what flip-flops provide.
- **Glitch isolation:** Combinational paths have varying propagation delays, so outputs can glitch briefly before settling. Flip-flops sample only at the clock edge, capturing the settled value and blocking glitches from rippling further downstream.

<img width="922" height="752" alt="why flip-flops" src="https://github.com/user-attachments/assets/9962570c-6d74-420e-927b-901163fb85da" />

---

## 9. Synchronous vs. Asynchronous Reset

Resets force a flip-flop to a known state (0 or 1), typically at power-up.

**Synchronous Reset**
- Only takes effect on the active clock edge.
- ✅ Simple timing analysis, predictable within the clock domain.
- ❌ Won't fire if the clock is dead.

**Asynchronous Reset**
- Takes effect immediately, independent of the clock.
- ✅ Works even without a clock — useful at power-up.
- ❌ Trickier timing (recovery/removal constraints), risk of metastability if released too close to a clock edge.

<img width="877" height="792" alt="sync vs async reset" src="https://github.com/user-attachments/assets/a9118a2f-a2f8-4c2e-a5ea-1f01590fd15a" />

---

## 10. Fast Math: Multiplying by Powers of Two

Dedicated multiplier hardware is expensive in area and power. But multiplying by a power of 2 is just a **left shift** — no logic gates required, only rewired data lines.

**× 2 (2¹):** shift left by 1
```verilog
y = x << 1;
// 0011 (3) -> 0110 (6)
```

**× 8 (2³):** shift left by 3
```verilog
y = x << 3;
// 0001 (1) -> 1000 (8)
```

Using `<<` instead of `*` lets the synthesis tool skip multiplier logic entirely, saving both area and power.
