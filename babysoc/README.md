# BabySoC — Synthesis & Simulation Walkthrough

This log documents the RTL-to-gate-level synthesis flow for the `vsdbabysoc`
design using **Yosys** (mapped to the `sky130_fd_sc_hd` standard cell
library), along with pre-synthesis vs. post-synthesis simulation
verification in **GTKWave**.

---

## 1. Design Statistics (`stat` after `read_verilog`)

Top-level hierarchy showing `avsddac`, `avsdpll`, and `rvmyth` as
sub-modules of `vsdbabysoc`, with `rvmyth` instantiating 7 `clk_gate`
cells.

![Yosys design stats](01_yosys_stats_vsdbabysoc.jpeg)

---

## 2. Generated RTL Netlist — `clk_gate` / `rvmyth` modules

Yosys-generated Verilog showing the `clk_gate` module definition and the
start of the `rvmyth` core wire declarations.

![clk_gate and rvmyth netlist](02_clk_gate_rvmyth_netlist.jpeg)

---

## 3. GTKWave — Pre-Synthesis vs. Post-Synthesis Comparison

Side-by-side waveform comparison of `pre_synth_sim.vcd` and
`post_synth_sim.vcd`, confirming matching behavior on `CLK`, `OUT`,
`RV_TO_DAC[9:0]`, and other key signals.

![GTKWave pre vs post synth](03_gtkwave_pre_vs_post_synth.jpeg)

---

## 4. Netlist File in Mousepad — `baby_soc_netlist.v`

Viewing the raw generated netlist file directly in a text editor.

![baby_soc_netlist.v in Mousepad](04_baby_soc_netlist_mousepad.jpeg)

---

## 5. Yosys `stat` + `check` Pass

Full design statistics followed by the `CHECK` pass confirming
**0 problems found** across `clk_gate`, `rvmyth`, and `vsdbabysoc`.

![Yosys stat and check pass](05_yosys_check_pass_stats.jpeg)

---

## 6. DFF Legalization — Mapping to `sky130_fd_sc_hd` Flip-Flops

`dfflegalize` pass mapping generic `$DFF_*` / `$DFFSR_*` cells to
technology-specific flip-flops (`dfxtp`, `dfrtp`, `dfstp`, `edfxtp`,
`dfbbn`, `dfbbp`), including 1273 `$DFF_P_` cells mapped in `rvmyth`.

![DFF legalize mapping](06_dfflegalize_mapping.jpeg)

---

## 7. GTKWave — Pre vs Post Synthesis (Design Hierarchy Expanded)

Second waveform comparison with the `uut` hierarchy expanded to show
`core`, `dac`, and `pll` sub-blocks.

![GTKWave pre vs post synth v2](07_gtkwave_pre_vs_post_synth_v2.jpeg)

---

## 8. ABC Synthesis Results — Cell-Level Breakdown

`ABC RESULTS` summary listing the final `sky130_fd_sc_hd` cell counts
(nand2, a21oi, nor2, mux2, xor2, etc.) used to implement the design.

![ABC synthesis results](08_abc_results_synthesis.jpeg)

---

## 9. Yosys Optimization Passes (`opt`)

`OPT_REDUCE`, `OPT_MERGE`, `OPT_DFF`, `OPT_CLEAN`, and `OPT_EXPR` passes
completing with no further changes needed.

![Yosys opt passes](09_yosys_opt_passes.jpeg)

---

## 10. Comparing Two Netlist Versions Side-by-Side

`baby_soc_netlist.v` vs `baby_soc2_netlist.v` opened in separate Mousepad
tabs for comparison.

![Two netlist files in Mousepad](10_netlist_files_mousepad_tabs.jpeg)

---

## 11. Final Gate-Level Netlist — `sky130_fd_sc_hd` Standard Cells

Snippet of the final mapped netlist showing instantiated `nand2`, `nor2`,
`a311oi`, and `o211ai` standard cells.

![Final netlist with sky130 cells](11_netlist_sky130_cells.jpeg)

---

## 12. Standard Cell Usage Summary (Terminal)

Full reference count of every `sky130_fd_sc_hd` cell type used in the
final synthesized netlist.

![Terminal cell reference counts](12_terminal_cell_reference_counts.jpeg)

---

## 13. Testbench — `testbench.v`

Top-level testbench with `PRE_SYNTH_SIM` / `POST_SYNTH_SIM` conditional
includes, instantiating the `vsdbabysoc` DUT.

![testbench.v in Mousepad](13_testbench_mousepad.jpeg)

---

### Summary

| Stage | Tool | Result |
|---|---|---|
| RTL elaboration | Yosys `stat` | 5160 cells, 0 problems on `check` |
| Technology mapping | ABC + `dfflegalize` | Mapped to `sky130_fd_sc_hd` |
| Optimization | `opt_*` passes | 0 further changes (already clean) |
| Verification | GTKWave | Pre-synth vs post-synth waveforms match |
