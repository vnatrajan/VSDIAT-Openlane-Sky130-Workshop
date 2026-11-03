# VSDIAT-Openlane-Sky130-Workshop
This repository captures the outcome achieved while attending the Openlane Sky130 Workshop from VSD

# Digital VLSI SoC Design and Planning — RTL to GDSII with OpenLANE & Sky130

> A hands-on, 5-day workshop by **VSD (VLSI System Design)** covering the complete RTL-to-GDSII flow
> for the `picorv32a` RISC-V core, using only open-source EDA tools and the SkyWater **Sky130** PDK.
> This repository documents the lab outputs, screenshots, measurements and key takeaways from each day.

## Contents

- [Repository Layout](#repository-layout)
- [Run Summary — `picorv32a`](#run-summary--picorv32a)
- [Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK](#day-1--inception-of-open-source-eda-openlane--sky130-pdk)
- [Day 2 — Floorplanning and Introduction to Library Cells](#day-2--floorplanning-and-introduction-to-library-cells)
- [Day 3 — Design and Characterisation of Library Cells](#day-3--design-and-characterisation-of-library-cells)
- [Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis](#day-4--pre-layout-timing-analysis-and-clock-tree-synthesis)
- [Day 5 — Final Steps for RTL2GDS using TritonRoute and OpenSTA](#day-5--final-steps-for-rtl2gds-using-tritonroute-and-opensta)
- [Tools & Environment](#tools--environment)
- [Key Learnings](#key-learnings)
- [Acknowledgements](#acknowledgements)
- [References](#references)

---

## Repository Layout

| Path | Contents |
|---|---|
| [`src/openlane/`](src/openlane) | OpenLANE v0.21 setup, flow scripts and configuration used in the labs |
| [`src/openlane/designs/picorv32a/`](src/openlane/designs/picorv32a) | `picorv32a` design: RTL, SDC, timing libraries, custom inverter LEF and `config.tcl` |
| [`src/openlane/designs/picorv32a/runs/23-10_13-58/`](src/openlane/designs/picorv32a/runs/23-10_13-58) | Complete flow run — logs, reports and results (DEF, SPEF, netlists) |
| [`src/openlane/vsdstdcelldesign/`](src/openlane/vsdstdcelldesign) | Custom inverter standard cell (`sky130_inv.mag`) and Sky130 tech file used on Days 3–4 |
| [`src/pdks/sky130A/`](src/pdks/sky130A) | Sky130A PDK files (tech LEF, Liberty, Magic tech) |
| [`workshop_results/`](workshop_results) | Lab screenshots organised by day (`Day_1` … `Day_5`) and sub-module (`SKY130_Dx_SKy`) |

---

## Run Summary — `picorv32a`

All values below come from the run `runs/23-10_13-58` (OpenLANE v0.21) in this repository. Some lab screenshots further down were taken during earlier exploratory runs (`15-10_14-39`, `22-10_14-49`), so their numbers can differ slightly.

| Metric | Value | Source |
|---|---|---|
| Total cells after synthesis | 14,876 | `reports/synthesis/1-yosys_4.stat.rpt` |
| D flip-flops (`sky130_fd_sc_hd__dfxtp_2`) | 1,613 | `reports/synthesis/1-yosys_4.stat.rpt` |
| **Flop ratio** | 1613 / 14876 ≈ **0.1084 (10.84 %)** | calculated |
| Synthesised chip area | 147,712.92 µm² | `reports/synthesis/1-yosys_4.stat.rpt` |
| Core utilisation (`FP_CORE_UTIL`) | 10 % | `config.tcl` |
| Placement target density (`PL_TARGET_DENSITY`) | 0.4 | `config.tcl` |
| Die area | 1226.41 µm × 1237.13 µm | `reports/floorplan/3-verilog2def.die_area.rpt` |
| Core area | (5.52, 10.88) → (1220.84, 1224.00) µm | `reports/floorplan/3-verilog2def.core_area.rpt` |
| Total routed wire length | 1,072,693 µm | TritonRoute log |
| Total vias | 136,407 | TritonRoute log |
| DRC violations after detailed routing | **0** | `reports/routing/28-tritonRoute.drc` |

Worst negative slack (WNS) and total negative slack (TNS) reported by OpenSTA at each stage of this run:

| Stage | WNS (ns) | TNS (ns) |
|---|---|---|
| Post-synthesis STA | -24.89 | -759.46 |
| Post-resizer (after placement) | -16.12 | -352.27 |
| Post-route STA with SPEF | -16.90 | -368.12 |

The resizer recovered about a third of the initial WNS. Further slack-fixing experiments (synthesis strategy, fanout limits, cell replacement, CTS buffer sizes) are documented in [Day 4](#day-4--pre-layout-timing-analysis-and-clock-tree-synthesis).

---

## Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK

### SK1 — How to Talk to Computers

#### Package, Chip, Pads, Core and Die

The "chip" seen on a board (for example in a QFN-48 package) is really the **package**. The silicon **die** sits inside it and is wire-bonded to the package pins. On the die, **pads** around the edge carry signals in and out, and the **core** in the middle holds the digital logic. The core also hosts **foundry IPs** (PLLs, SRAMs, ADCs — blocks that need process know-how) and **macros** (reusable, purely digital blocks).

![QFN-48 package and IPs](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L1 - Introduction to QFN-48 Package, chip, pads, core, die and IPs - IP Foundries.png>)

#### RISC-V to Implementation to Layout

The workshop follows one design end to end: the RISC-V ISA is described in RTL (`picorv32a`), synthesised to gates, and implemented as a physical layout.

![RISC-V to layout](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L2 - Introduction to RISC-V-to-Implementation-to-layout.png>)

#### From Software Applications to Hardware

An application runs on hardware through the **system software** stack: the OS hands the program to the **compiler**, which produces ISA-specific instructions; the **assembler** turns them into binary; and the binary is executed by hardware that implements the ISA in RTL.

![Architecture of computer](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L3 - From Software Applications to Hardware - Architecture of computer.png>)

![Compiler input and output](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L3 - From Software Applications to Hardware - Compiler input and output.png>)

![Mapping software to binary](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L3 - From Software Applications to Hardware - Mapping application software to target architecture binary instruction set.png>)

![Mapping software to hardware layout](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L3 - From Software Applications to Hardware - Mapping application software to target architecture hardware layout.png>)

![RTL to netlist to physical](<workshop_results/Day_1/SKY130_D1_SK1/SKY_L3 - From Software Applications to Hardware - Synthesizing RTL to netlist to physical realization.png>)

### SK2 — SoC Design and OpenLANE

#### Open-Source Digital ASIC Design

An open-source ASIC flow needs three ingredients: **RTL designs**, **EDA tools** and a **PDK** (the interface between the designer and the fab: process design rules, device models, standard-cell libraries). SkyWater and Google released the Sky130 PDK as open source in 2020, which made the full flow possible without NDAs.

![RTL to GDSII flow](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - RTL to GDSII.png>)

#### Steps of the RTL-to-GDSII Flow

| Step | What happens |
|---|---|
| **Synthesis** | RTL is translated to a gate-level netlist of standard cells from the library |
| **Floor & power planning** | Die/core size, macro locations, I/O pins and the power grid are decided |
| **Placement** | Standard cells are placed on the floorplan rows — global placement, then legalisation |
| **CTS** | A clock tree is built to deliver the clock to all flops with minimal skew |
| **Routing** | Global routing creates guides; detailed routing draws metal and vias |
| **Sign-off** | Physical verification (DRC, LVS) and timing verification (STA) |

![Synthesis step](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - Synthesis step.png>)

![Synthesis step details](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - Synthesis step details.png>)

![Floor planning](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - Floor planning.png>)

![Macro floor planning](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - Macro Floor planning.png>)

![Power planning](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - power planning.png>)

![Routing](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - routing.png>)

![Sign off](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L1 - Introduction to all components of open-source digital asic design - sign off.png>)

#### OpenLANE Detailed ASIC Design Flow

OpenLANE is an automated flow built from open-source tools. Beyond the basic steps it adds **logic equivalence checking (LEC)** after netlist changes, optional **DFT** insertion, **antenna rule** fixing with diodes, and sign-off checks.

![OpenLANE ASIC flow](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L4 - Introduction to OpenLANE detailed ASIC design flow - OpenLANE ASIC Flow.png>)

| Stage | Tool(s) |
|---|---|
| Synthesis | Yosys, ABC |
| Static timing analysis | OpenSTA |
| Floorplan & PDN | OpenROAD (init_fp, ioPlacer, tapcell, pdngen) |
| Placement | RePlAce (global), OpenDP (detailed) |
| Clock tree synthesis | TritonCTS |
| Routing | FastRoute (global), TritonRoute (detailed) |
| Logic equivalence | Yosys |
| Parasitic extraction | SPEF extractor |
| GDSII streaming | Magic, KLayout |
| DRC & LVS | Magic, Netgen |

![Physical implementation](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L4 - Introduction to OpenLANE detailed ASIC design flow - Physical implementation.png>)

![LEC](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L4 - Introduction to OpenLANE detailed ASIC design flow - LEC.png>)

![DFT](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L4 - Introduction to OpenLANE detailed ASIC design flow - DFT.png>)

Long metal wires collect charge during fabrication and can damage transistor gates (the **antenna effect**). OpenLANE fixes this by inserting **antenna diodes** or by bridging to a higher metal layer.

![Antenna rules violation resolution](<workshop_results/Day_1/SKY130_D1-SK2/SKY_L4 - Introduction to OpenLANE detailed ASIC design flow - Antenna Rules Violation resolution.png>)

### SK3 — Lab: Getting Familiar with Open-Source EDA Tools

#### Invoking OpenLANE and Preparing the Design

```bash
cd ~/Desktop/work/tools/openlane_working_dir/openlane
docker
./flow.tcl -interactive
```

```tcl
package require openlane 0.9
prep -design picorv32a
```

`prep` merges the technology LEF and the cell LEF into `tmp/merged.lef` and creates a new run directory under `designs/picorv32a/runs/`.

![Design preparation](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane.png>)

![OpenLANE interactive mode](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - OpenLane commands interactive mode.png>)

![OpenLANE flow](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - OpenLane flow.png>)

![picorv32a runs](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - picorv32a runs.png>)

The run's `config.tcl` records every parameter the flow used, merging defaults with the design's own settings:

![Run config.tcl](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane runs config.tcl.png>)

#### Running Synthesis

```tcl
run_synthesis
```

![run_synthesis](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - run_synthesis.png>)

Synthesis completed with a chip area of **147,712.92 µm²**, **14,876 cells** and **1,613** `dfxtp_2` flip-flops.

**Flop ratio** = number of D flip-flops / total number of cells = 1613 / 14876 ≈ **0.1084 → 10.84 %**

![Post-synthesis output files](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - post synthesis runs output files.png>)

OpenSTA runs straight after synthesis and writes the critical path to `reports/synthesis/2-opensta.timing.rpt`:

![Post-synthesis OpenSTA timing report](<workshop_results/Day_1/SKY130_D1-SK3/SKY_L2 - Design Preparation Step using OpenLane - post synthesis runs output file opensta timing report.png>)

---

## Day 2 — Floorplanning and Introduction to Library Cells

### SK1 — Chip Floorplanning Considerations

#### Utilisation Factor and Aspect Ratio

- **Utilisation factor** = area occupied by the netlist / total core area (typically 0.5–0.7 in practice; 1.0 leaves no room for buffers or routing)
- **Aspect ratio** = core height / core width (1.0 means a square core)

#### Pre-Placed Cells and Decoupling Capacitors

Large blocks (memories, IPs, clock gating, muxes) are **pre-placed** before automated placement and their locations stay fixed. **Decoupling capacitors** placed next to them act as local charge reservoirs during switching, keeping supply droop and ground bounce within the noise margins.

#### Power Planning

A single VDD/VSS pair cannot supply a whole chip — when many cells switch at once, the rails sag (droop) or rise (ground bounce). A **power mesh** of VDD/VSS straps on several metal layers gives every cell a nearby power tap.

![Power planning](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L4 - Power planning-1.png>)

#### Pin Placement and Logical Cell Placement Blockage

I/O pins are placed around the core according to connectivity, and clock pins are usually larger because they drive the whole chip. The area between the core and the die edge is blocked so the placer does not put standard cells there.

![Pin placement](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L5 - Pin placement and logical cell placement blockage.png>)

### Lab — Floorplan

Floorplan settings come from `configuration/floorplan.tcl` (defaults), then `designs/picorv32a/config.tcl`, then `sky130A_sky130_fd_sc_hd_config.tcl` — each later file overrides the earlier one.

```tcl
run_floorplan
```

![OpenLANE command steps](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/OpenLane command steps.png>)

![run_floorplan](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE - run_floorplan.png>)

![Floorplan run output](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE_1.png>)

The ioPlacer log shows the design that the floorplan works on: **409 pins**, **14,876 components**, **14,978 nets**, 440 library cells and 13 technology layers. Pins are placed in random, equidistant mode.

![ioPlacer logs](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE - ioplacer logs.png>)

The floorplan DEF lists the die area and the standard-cell rows. With `UNITS DISTANCE MICRONS 1000`, the die in this exploratory run measured `(0 0) (660685 671405)` → **660.685 µm × 671.405 µm**.

![Floorplan DEF](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE - picorv32a.floorplan.def output.png>)

#### Viewing the Floorplan in Magic

```bash
cd designs/picorv32a/runs/<run-tag>/results/floorplan
magic -T ~/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef def read picorv32a.floorplan.def &
```

The I/O pins sit equally spaced around the edge, with tap cells and decap cells placed in the core rows.

![Floorplan in Magic](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE - running magic on the floorplan def.png>)

![Floorplan expanded](<workshop_results/Day_2/SKY130_D2_SK1 - Chip Floor planning considerations/SKY_L6 - Steps to run floorplan using OpenLANE - running magic on the floorplan expanded layout.png>)

### SK2 — Library Binding and Placement

The netlist gates are bound to real cells from the library (with actual shapes and sizes) and placed on the floorplan, close to the I/O pins they connect to. Long nets get **repeaters** (buffers) to restore signal integrity, and a quick **STA with ideal clocks** checks whether the placement can meet timing.

![Netlist binding and placement](<workshop_results/Day_2/SKY130_D2_SK2 - Library Binding and Placement/SKY_L1 - Netlist binding and initial place design - placement.png>)

![Static timing analysis](<workshop_results/Day_2/SKY130_D2_SK2 - Library Binding and Placement/SKY_L4 - Static timing analysis.png>)

### Lab — Placement

Placement runs in two stages: **global placement** (RePlAce) minimises wire length, then **detailed placement** (OpenDP) legalises cells onto rows without overlaps.

```tcl
run_placement
```

![run_placement](<workshop_results/Day_2/SKY130_D2_SK2 - Library Binding and Placement/SKY_L5 - Congestion aware placement using RePlAce - run_placement.png>)

![Placement in Magic](<workshop_results/Day_2/SKY130_D2_SK2 - Library Binding and Placement/SKY_L5 - Congestion aware placement using RePlAce - placement view using magic.png>)

![Placement expanded](<workshop_results/Day_2/SKY130_D2_SK2 - Library Binding and Placement/SKY_L5 - Congestion aware placement using RePlAce - placement expanded view using magic.png>)

### SK3 — Cell Design and Characterisation Flows

A standard cell goes through three stages:

1. **Inputs** — PDK (DRC/LVS rules, SPICE models), library and user-defined specs (cell height, supply voltage, metal layers, pin locations, drawn gate length)
2. **Design** — circuit design (W/L sizing), layout design (stick diagram, Euler path, DRC-clean layout), extraction of parasitics
3. **Outputs** — CDL, GDSII, LEF, extracted SPICE netlist, and timing/noise/power `.lib` files

![Cell design flow and SPICE parameters](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L1 - Inputs for cell design flow - Cell Design Flow & Spice mode parameters.png>)

![Placing and routing view](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L1 - Inputs for cell design flow - placing & routing view.png>)

![Circuit design step](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Circuit design step - Metal layers, pin location, drawn gate-length.png>)

![Overall cell design flow](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - overall cell design flow.png>)

The layout is planned with an **Euler path** (one continuous path through the transistor network so the diffusion can be shared) and a **stick diagram**:

![Euler path](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Eulers path.png>)

![Stick diagram](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Stick diagram.png>)

#### Characterisation Flow

The characterisation software (GUNA) takes eight inputs and produces timing, noise and power models:

1. Read in the SPICE models
2. Read in the extracted SPICE netlist
3. Recognise the cell behaviour
4. Read the sub-circuits
5. Attach power sources
6. Apply stimulus
7. Provide the output load capacitances
8. Provide the simulation commands

![Characterisation steps 1, 2, 5, 8](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Characetrization flow - Step-1 (read-in model), Step 2 (extract spice netlist), Step-5 (sub circuits), Step-8 (apply stimulus).png>)

![Characterisation steps 3, 5, 6, 7](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Characetrization flow - Step-3(behaviour), Step-5(attach power), Step-6(apply stimulus), Step-7(provide load capacitances).png>)

![GUNA after 8 steps](<workshop_results/Day_2/SKY130_D2_SK3 - Cell design and characterization flows/SKY_L2 - Characetrization flow - Apply GUNA after 8-STEPS.png>)

### SK4 — General Timing Characterisation Parameters

| Parameter | Definition |
|---|---|
| `slew_low_rise_thr` / `slew_high_rise_thr` | 20 % / 80 % of the rising waveform |
| `slew_low_fall_thr` / `slew_high_fall_thr` | 20 % / 80 % of the falling waveform |
| `in_rise_thr` / `in_fall_thr` | 50 % point of the input waveform |
| `out_rise_thr` / `out_fall_thr` | 50 % point of the output waveform |

- **Propagation delay** = time(out_thr) − time(in_thr). A poorly chosen threshold, or a large wire delay, can give a **negative delay**, which is not physically meaningful.
- **Transition time** = time(slew_high_thr) − time(slew_low_thr) for a rising edge, and the reverse for a falling edge.

![SLEW thresholds](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - Timing threshold definitions (SLEW).png>)

![Rise thresholds](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - Timing threshold definitions (rise threholds).png>)

![Fall thresholds](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - Timing threshold definitions (fall threholds).png>)

![Propagation delay](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - Propagation delay.png>)

![Transition time](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - transition time.png>)

![Transition time calculations](<workshop_results/Day_2/SKY130_D2_SK4 - General timing characterization parameters/SKY_L1 - transition time calculations.png>)

---

## Day 3 — Design and Characterisation of Library Cells

### SK1 — CMOS Inverter ngspice Simulations

#### I/O Placer Revision

Floorplan switches can be changed on the fly in interactive mode. The `FP_IO_MODE` variable (documented in `configuration/README.md` and set in `floorplan.tcl`) controls pin placement; the default `1` places pins randomly at equal distances.

```tcl
set ::env(FP_IO_MODE) <mode>
run_floorplan
```

![Configuration in README.md](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L0 - IO placer revision - Configuration in README_md.png>)

![Configuration in floorplan.tcl](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L0 - IO placer revision - Configuration in floorplan_tcl.png>)

![FP_IO_MODE change](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L0 - IO placer revision - pin config changes FP_IO_MODE env.png>)

#### SPICE Deck

The SPICE deck describes the connectivity of the PMOS and NMOS, their W/L ratios, the supply and input sources, the output load capacitance, the model files and the simulation commands.

![SPICE deck](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L2 - SPICE simulation lab for CMOS inverter - SPICE Deck.png>)

#### Switching Threshold (Vm) — Static Behaviour

**Vm** is the input voltage at which Vin = Vout; both transistors are in saturation and the inverter switches. Increasing the PMOS width relative to the NMOS shifts Vm higher and balances the rise and fall delays. With Wn = Wp = 0.375 µm (Wn/Ln = Wp/Lp = 1.5) the curve gives **Vm ≈ 0.99 V**, with a rise delay of 148 ps and a fall delay of 71 ps.

![Switching threshold 1](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L3 - Switching Threshold Vm - static behaviour evaluation 1.png>)

![Switching threshold 2](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L3 - Switching Threshold Vm - static behaviour evaluation 2.png>)

![Rise and fall delay tabulation](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - rise & fall delay tabulation.png>)

#### Static and Dynamic Simulation

A **DC sweep** of the input gives the voltage transfer characteristic (static behaviour), while a **transient** analysis with a pulse input gives the rise and fall delays (dynamic behaviour).

![Static and dynamic simulation](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - 1.png>)

![Types of analysis](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - 2 - types of analysis.png>)

![Transient analysis](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - 2 - transient analysis.png>)

![Output vs input and time](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - plot out vs inpute and time.png>)

![Rise delay calculation](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - rise delay calculation.png>)

![Fall delay calculation](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - fall delay calculation.png>)

![Inverter in Magic](<workshop_results/Day_3/SKY130_D3_SK1 - Labs for CMOS inverter ngspice simulations/SKY_L4 - Static and dynamic simulation of CMOS inverter - magic view.png>)

### SK2 — Inception of Layout: the 16-Mask CMOS Fabrication Process

1. **Substrate** — P-type silicon, high resistivity, (100) orientation
2. **Active regions** — SiO₂ and Si₃N₄ layers, photoresist and Mask 1, then LOCOS field oxide to isolate the transistor pockets
3. **N-well and P-well** — Mask 2 (boron for P-well) and Mask 3 (phosphorus for N-well), then a drive-in furnace to diffuse the wells
4. **Gate terminal** — Masks 4 and 5 for threshold-voltage implants, gate oxide regrowth, polysilicon deposition and Mask 6 for the gate
5. **Lightly doped drain (LDD)** — Masks 7 (N⁻) and 8 (P⁻) to reduce hot-electron and short-channel effects, then spacers by anisotropic plasma etching
6. **Source and drain** — screen oxide, Mask 9 (arsenic, N⁺) and Mask 10 (boron, P⁺), then annealing in a high-temperature furnace
7. **Contacts and local interconnects** — titanium sputtering, heating in nitrogen to form TiSi₂ and TiN, Mask 11 to etch the TiN into local interconnects
8. **Higher-level metals** — BPSG/CMP planarisation, Mask 12 contact holes, TiN liner and tungsten plugs, aluminium with Mask 13, repeat with Masks 14–15, then a Si₃N₄ passivation layer and Mask 16 for the bond-pad openings

![Active and isolated regions](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L1 - Create Active & isolated regions in CMOS prep process.png>)

![Create P-well](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L2 - Formation of N-well and P-well - Create P-Well in CMOS.png>)

![Create N-well](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L2 - Formation of N-well and P-well - Create N-Well in CMOS.png>)

![Drive-in furnace](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L2 - Formation of N-well and P-well - Create P-Well & N-Well heating in a drive-in firnace.png>)

![Threshold voltage](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L3 - Formation of gate terminal - Threshold voltage circuit design.png>)

![Gate using Mask 6](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L3 - Formation of gate terminal - Create Gate using Mask 6.png>)

![LDD formation](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L4 - Lightly doped drain (LDD) formation.png>)

![Anisotropic plasma etching](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L4 - Anisotropic Plasma Etching.png>)

![Arsenic N+ implant](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L5 - Source – drain formation - Arsenic doping with Mask 9 step for N+ transplant.png>)

![Annealing](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L5 - Source – drain formation - Annealing in high temp furnacet.png>)

![TiN local interconnect](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - TiN for local comms.png>)

![Tungsten layer](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post Mask-12 step add blanket tungsten layer.png>)

![Aluminium interconnect](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post Mask-13 step Aluminium layer interconnect.png>)

![Si3N4 passivation](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post Mask-15 step add Silicon Nitride (Si₃N₄) dielectric that protects entire chip.png>)

![Finished chip after fabrication steps](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Finished chip fabricate.png>)

<details>
<summary><b>More fabrication-step screenshots</b></summary>

![Remove and regrow oxide](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L3 - Formation of gate terminal - Remove & re-growing oxide layer in 16-mask process.png>)

![Step 4](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L3 - Formation of gate terminal - Step 4 of the 16-mask process.png>)

![Step 5](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L3 - Formation of gate terminal - Step 5 of the 16-mask process.png>)

![LDD N- implant Mask 7](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L4 - Lightly Doped Drain LDD creating N- implant using Mask 7 step.png>)

![LDD P- implant Mask 8](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L4 - Lightly Doped Drain LDD creating P- implant using Mask 8 step.png>)

![Screen oxide](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L5 - Source – drain formation - screen oxide coating.png>)

![Boron P+ implant](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L5 - Source – drain formation - Boron doping with Mask 9 step for P+ transplant.png>)

![Titanium sputtering](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Titanium deposit using sputtering.png>)

![Sputtering](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Sputtering.png>)

![Heating in nitrogen](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Heating in a Nitrogen ambient to create local interconnects.png>)

![RCA cleaning](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - RCA cleaning.png>)

![Post RCA cleaning](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post RCA cleaning.png>)

![Mask 11](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Mask11 step.png>)

![CMP](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - CMP step.png>)

![Mask 12](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Mask-12 step.png>)

![TiN liner](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post Mask-12 step add thin layer of tritranium nitride.png>)

![Aluminium deposition](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Aluminium layer deposition.png>)

![Mask 13](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Mask-13 step.png>)

![Mask 14 contact holes](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Mask-14 to create contact holes.png>)

![TiN after Mask 14](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Post Mask-14 create TiN deposit.png>)

![Mask 15](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L6 - Local interconnect formation - Mask-15 step.png>)

![Higher metal CMP](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L7 - Higher level metal formation - CMP step.png>)

![Glass layer](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L7 - Higher level metal formation - Glass layer deposit.png>)

</details>

### Lab — Custom Inverter Layout and SPICE Extraction

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
cd vsdstdcelldesign
cp ~/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech .
magic -T sky130A.tech sky130_inv.mag &
```

The layout shows the PMOS (top) and NMOS (bottom) sharing a polysilicon gate, with input `A`, output `Y`, and the `VPWR`/`VGND` rails. Hovering over a layer and typing `what` in the tkcon window identifies it; `drc why` explains any DRC marker.

Extracting the SPICE netlist from the layout in the Magic `tkcon` window:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

![Extract design](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L9 - Lab steps to create std cell layout and extract spice netlist - Extract design using ngspice.png>)

![ext2spice](<workshop_results/Day_3/SKY130_D3_SK2 - Inception of Layout – CMOS fabrication process/SKY_L9 - Lab steps to create std cell layout and extract spice netlist - ext2spice to create spice file.png>)

### SK3 — Lab: Characterising the Inverter with ngspice

The extracted netlist is edited before simulation: the grid size is checked in Magic (0.01 µm) and used for the `.option scale`, model names are changed to `pshort_model.0` / `nshort_model.0`, the Sky130 model library is included, a 3.3 V supply and a pulse input are added, and a transient analysis is set up.

![Creating the inverter spice file](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter.png>)

![sky130_inv.spice](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter - sky130_inverter.spice file.png>)

```bash
ngspice sky130_inv.spice
```

```
plot y vs time a
```

![plot y vs time a](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L2 - Lab steps to characterize inverter using sky130 model files - plot y vs time a.png>)

Increasing the load capacitance slows the output edges visibly:

![plot with larger C](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L2 - Lab steps to characterize inverter using sky130 model files - plot y vs time a for larger C value.png>)

#### Measured Values

With a 3.3 V supply, 20 % = 0.66 V, 50 % = 1.65 V and 80 % = 2.64 V. Points were picked by clicking on the waveform in ngspice:

| Parameter | Measured between | Calculation | Result |
|---|---|---|---|
| Rise transition | output 20 % → 80 % | 6.24602 ns − 6.18133 ns | **≈ 64.7 ps** |
| Fall transition | output 80 % → 20 % | 4.09324 ns − 4.05301 ns | **≈ 40.2 ps** |
| Cell rise delay | input 50 % → output 50 % | 4.07785 ns − 4.04940 ns | **≈ 28.5 ps** |
| Propagation delay | input 50 % → output 50 % | 2.21029 ns − 2.15049 ns | **≈ 59.8 ps** |

![20% to 80% rise transition](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter - 20% to 80% rise transition.png>)

![80% to 20% fall transition](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter - 80% to 20% fall transition.png>)

![50% to 50% cell rise delay](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter - 50% input to 50% output cell rise delay.png>)

![50% to 50% propagation delay](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L1 - Lab steps to create spice file for inverter - 50% input to 50% output propogation delay.png>)

### Lab — Sky130 Tech File and DRC Fixes in Magic

The `drc_tests` labs (from `opencircuitdesign.com/open_pdks/archive/drc_tests.tgz`) load small layouts, check DRC in Magic and fix incorrect rules in `sky130A.tech`. The tech file is edited and reloaded without restarting Magic:

```tcl
tech load sky130A.tech
drc check
drc why
```

First, `met3.mag` is loaded to see how Magic reports metal-3 rules:

![Load met3.mag](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L5 - Lab introduction to Magic and steps to load Sky130 tech-rules - Load met3.mag in Magic.png>)

Then `poly.mag` shows the **poly.9** problem: the spacing between a poly resistor and poly/diffusion is not flagged even though it violates the rule. The rule is corrected in `sky130A.tech` and the check is repeated.

![Partial fix in sky130A.tech](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L5 - Lab introduction to Magic and steps to load Sky130 tech-rules - partial fix in sky130A.tech for issue in poly.mag and checking in Magic.png>)

![Fixing poly.9 — 1](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L6 - Lab exercise to fix poly.9 error in Sky130 tech-file- Load poly.mag in Magic and check.png>)

![Fixing poly.9 — 2](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L6 - Lab exercise to fix poly.9 error in Sky130 tech-file - Load poly.mag in Magic and check 2.png>)

![Fixing poly.9 — 3](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L6 - Lab exercise to fix poly.9 error in Sky130 tech-file - Load poly.mag in Magic and check 3.png>)

![DRC error as geometrical construct](<workshop_results/Day_3/SKY130_D3_SK3 - Sky130 Tech File Labs/SKY_L8 - Lab challenge exercise to describe DRC error as geometrical construct.png>)

---

## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

### SK1 — Timing Modelling Using Delay Tables

#### Converting Grid Info to Track Info

For the placer and router, a standard cell must satisfy two rules:

1. Input and output ports lie on the intersection of horizontal and vertical routing tracks.
2. The cell width is an odd multiple of the horizontal track pitch, and the height is a multiple of the vertical track pitch.

The routing track pitches are read from `pdks/sky130A/libs.tech/openlane/sky130_fd_sc_hd/tracks.info` (li1: X 0.23 µm offset / 0.46 µm pitch, Y 0.17 µm offset / 0.34 µm pitch) and the Magic grid is set to match:

```tcl
grid 0.46um 0.34um 0.23um 0.17um
```

![Grid to track info](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L1 - Lab steps to convert grid info to track info.png>)

#### Converting the Magic Layout to a Standard-Cell LEF

Ports are defined on the layout (`Edit → Text`) with their class and use — `A` as input, `Y` as output, `VPWR` as inout power and `VGND` as inout ground. The cell is saved as `sky130_vsdinv.mag` and the LEF is written:

```tcl
save sky130_vsdinv.mag
lef write
```

![Selecting port A](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L2 - Lab steps to convert magic layout to std cell LEF - Selecting Port A.png>)

![Selecting port Y](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L2 - Lab steps to convert magic layout to std cell LEF - Selecting Port Y.png>)

![Selecting VPWR](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L2 - Lab steps to convert magic layout to std cell LEF - Selecting VPWR.png>)

![Creating LEF file](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L2 - Lab steps to convert magic layout to std cell LEF - crreating lef file.png>)

#### Including the Custom Cell in Synthesis

The cell LEF and the timing libraries are copied to `designs/picorv32a/src/` and referenced in `config.tcl` (see [`config_updated.tcl`](src/openlane/designs/picorv32a/config_updated.tcl)):

```tcl
set ::env(LIB_SYNTH)   "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_MIN)     "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_MAX)     "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL) "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)  [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

```tcl
prep -design picorv32a -tag <run-tag> -overwrite
set lefs [glob $::env(DESIGN_DIR)/src/*.lef]
add_lefs -src $lefs
run_synthesis
```

Synthesis ran successfully with the updated configuration: chip area 147,712.92 µm², **WNS −24.89 ns** and **TNS −759.46 ns** at the start of timing optimisation.

![Synthesis with the custom cell](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L3 - Introduction to timing libs and steps to include new cell in synthesis - run synthesis after changes in magic, config.tcl etc..png>)

#### Delay Tables

Cell delay depends on **input slew** and **output load**. Liberty files store it as 2-D lookup tables indexed by these two values; the timing tools interpolate between table entries. On a clock tree, buffers at the same level should drive the same load so that their delays — and therefore the skew — match.

![Delay table concept](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L5 - Delay table usage Part 1 - delay table concept.png>)

![Delay table concept continued](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L5 - Delay table usage Part 2 - delay table concept continued.png>)

#### Fixing Slack with Synthesis Settings

The synthesis variables in `configuration/README.md` control the trade-off between area and delay:

![Synthesis environment parameters](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L7 - Lab steps to configure synthesis settings to fix slack - Environment patameters.png>)

```tcl
echo $::env(SYNTH_STRATEGY)      ;# AREA 0
echo $::env(SYNTH_BUFFERING)     ;# 1
echo $::env(SYNTH_SIZING)        ;# 0
echo $::env(SYNTH_DRIVING_CELL)  ;# sky130_fd_sc_hd__inv_8
set ::env(SYNTH_STRATEGY) "AREA 2"
set ::env(SYNTH_SIZING) 1
run_synthesis
```

| Synthesis settings | WNS (ns) | TNS (ns) |
|---|---|---|
| `SYNTH_STRATEGY "AREA 0"`, `SYNTH_SIZING 0` (default) | -24.89 | -759.46 |
| `SYNTH_STRATEGY "AREA 2"`, `SYNTH_SIZING 1` | -24.01 | -702.12 |

![SYNTH_STRATEGY and SYNTH_SIZING](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L7 - Lab steps to configure synthesis settings to fix slack - Change environment SYNTH_STRATEGY & SYNTH_SIZING.png>)

The flow then continues with floorplan and placement, and the placed custom cell can be checked in Magic:

```tcl
init_floorplan
place_io
tap_decap_or
run_placement
```

![Run floorplan and placement](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L7 - Lab steps to configure synthesis settings to fix slack - run floorplan and placement.png>)

![Post-placement check in Magic](<workshop_results/Day_4/SKY130_D4_SK1 - Timing modelling using delay tables/SKY_L7 - Lab steps to configure synthesis settings to fix slack -post placement check in magic.png>)

### SK2 — Timing Analysis with Ideal Clocks Using OpenSTA

#### Setup Time, Jitter and Uncertainty

For a launch flop and capture flop with combinational logic in between, the setup condition is:

> combinational delay < clock period − setup time − setup uncertainty (jitter)

**Jitter** is the variation in clock-edge arrival caused by the clock source; it is modelled as **clock uncertainty** and reduces the time available for the data path.

![Clock jitter and uncertainty](<workshop_results/Day_4/SKY130_D4_SK2 - Timing analysis with ideal clocks using openSTA/SKY_L2 - Introduction to clock jitter and uncertainty.png>)

#### Post-Synthesis STA with OpenSTA

A `pre_sta.conf` file loads the libraries, the synthesised netlist and the SDC (`my_base.sdc`) into OpenSTA, outside the OpenLANE flow, so that individual cells can be examined and swapped:

```bash
sta pre_sta.conf
```

![OpenSTA post-synthesis](<workshop_results/Day_4/SKY130_D4_SK2 - Timing analysis with ideal clocks using openSTA/SKY_L3 - Lab steps to configure OpenSTA for post-synth timing analysis - update pre_sta.conf and run sta.png>)

#### Reducing Setup Violations

Two techniques were tried:

**1. Allowing a higher fanout during synthesis** — `SYNTH_MAX_FANOUT` was raised from 6 to 8 and synthesis was re-run. In this lab, the worst path with a 12 ns clock had slack **−36.74 ns** (TNS −3955.43 ns). After re-running synthesis, the report showed **WNS −16.31 ns, TNS −362.51 ns**.

```tcl
set ::env(SYNTH_MAX_FANOUT) 8
run_synthesis
```

![Increasing SYNTH_MAX_FANOUT](<workshop_results/Day_4/SKY130_D4_SK2 - Timing analysis with ideal clocks using openSTA/SKY_L4 - Lab steps to optimize synthesis to reduce setup violations - increasing SYNTH_MAX_FANOUT.png>)

**2. Upsizing cells on the critical path** — nets with high fanout and slow transitions were found with `report_net`, and their driving cells were replaced with stronger ones:

```tcl
report_net -connections _10391_
replace_cell _13013_ sky130_fd_sc_hd__inv_8
replace_cell _13016_ sky130_fd_sc_hd__or4_4
report_checks -fields {net cap slew input_pins} -digits 4
report_worst_slack -max
```

This improved the worst slack from **−36.74 ns to −35.90 ns** — a small but measurable gain from two cell swaps.

![Cell replacement for marginal improvement](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS - Minor tweaking for marginal improvement.png>)

The modified netlist is written back with `write_verilog` so the flow continues from the improved design:

![Verifying the written verilog](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS - verifying verilog file written with changed values.png>)

### SK3 — Clock Tree Synthesis with TritonCTS and Signal Integrity

- **H-tree** routing drives the clock from the mid-point of the flops, so every flop sees a similar wire length — this minimises **skew**.
- **Clock buffers** act as repeaters along the tree; unlike regular buffers they have equal rise and fall times.
- **Clock net shielding** surrounds the clock net with VDD/VSS wires, preventing **crosstalk** that could cause glitches or **delta delay**.

![H-tree mid-point routing](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L1 - Clock tree routing and buffering using H-Tree algorithm - clock mid point routing strategy.png>)

![Buffers as repeaters](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L1 - Clock tree routing and buffering using H-Tree algorithm - using buffers as repeaters strategy.png>)

![Glitch due to crosstalk](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L1 - Clock tree routing and buffering using H-Tree algorithm - glitch impact due to inadequate clock net shielding.png>)

![Clock net shielding](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L1 - Clock tree routing and buffering using H-Tree algorithm - using clock net shielding.png>)

With a real clock tree, the setup check also includes the clock network delay to each flop:

![Setup timing analysis with real clocks](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L1 - Setup timing analysis using real clocks - timimg analysis.png>)

### Lab — Running CTS

The CTS variables (target skew 200 ps, root buffer, sink clustering size and diameter, timing report) are documented in `configuration/README.md`:

![CTS configuration](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS -  CTS configurations.png>)

```tcl
run_floorplan
run_placement
run_cts
```

![run_floorplan with changed values](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS -  run_floorplan with changed values.png>)

![run_placement with changed values](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS -  run_placement with changed values.png>)

CTS finished successfully with **WNS −16.29 ns** and **TNS −362.51 ns**. The design after CTS has **36,936 components**, **15,590 nets** and **411 pins**, and a new netlist `picorv32a.synthesis_cts.v` is written with the clock buffers included.

![run_cts](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L3 - Lab steps to run CTS -  run_cts with changed values.png>)

The `run_cts` procedure itself lives in OpenLANE's Tcl scripts, and the key output files can be checked in the run directory:

![run_cts Tcl command](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L4 - Lab steps to verify CTS runs - run_cts tcl command.png>)

![Exploring key files](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L4 - Lab steps to verify CTS runs - exploring some key files.png>)

![OpenLANE flow](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/SKY_L4 - OpenLane Flow.png>)

The full OpenSTA report from this lab is saved in [`report output log.txt`](<workshop_results/Day_4/SKY130_D4_SK3 - Clock tree synthesis TritonCTS and signal integrity/report output log.txt>).

### SK4 — Timing Analysis with Real Clocks Using OpenSTA

#### Hold Timing

The hold check makes sure new data does not reach the capture flop before it has safely latched the previous data:

> combinational delay + clock delay to launch flop > hold time + clock delay to capture flop

The difference in clock arrival between launch and capture flops (Δ1, Δ2) is the **skew** and enters both setup and hold checks.

![Hold timing analysis](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L2 - Hold timing analysis using real clocks.png>)

![Delay analysis delta 1](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L2 - Hold timing analysis using real clocks - delay analysis (delta 1).png>)

![Delay analysis delta 2](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L2 - Hold timing analysis using real clocks - delay analysis (delta 2).png>)

![OpenLANE flow](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L4 - OpenLane Flow.png>)

#### Lab — Post-CTS STA in OpenROAD

OpenSTA is integrated in OpenROAD. The post-CTS database is created once and then reused:

```tcl
openroad
read_lef ./designs/picorv32a/runs/<run-tag>/tmp/merged.lef
read_def ./designs/picorv32a/runs/<run-tag>/results/cts/picorv32a.cts.def
write_db pico_cts.db
read_db pico_cts.db
read_verilog ./designs/picorv32a/runs/<run-tag>/results/synthesis/picorv32a.synthesis_cts.v
read_liberty -max $::env(LIB_MAX)
read_sdc ./designs/picorv32a/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4
```

With `full_clock_expanded`, the report shows every clock buffer on the path (`clkbuf_16` at the root, `clkbuf_1` in the branches). Using the slow-corner library on its own gave a setup slack of **−5.7550 ns** for a flop-to-flop path:

![Checking in OpenROAD](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L4 - Lab steps to execute OpenSTA with right timing libraries and CTS assignment - checking in openroad.png>)

#### Lab — Impact of Bigger CTS Buffers

`CURRENT_DEF` was pointed back to the placement DEF before re-running CTS, so the new clock tree was built on the placed (pre-CTS) design rather than on top of an existing tree:

```tcl
echo $::env(CURRENT_DEF)
set ::env(CURRENT_DEF) /openLANE_flow/designs/picorv32a/runs/<run-tag>/results/placement/picorv32a.placement.def
```

With the correct DEF and SDC, the reported I/O paths met timing: **hold slack +2.7993 ns** (min path) and **setup slack +6.3945 ns** (max path).

![Impact of bigger CTS buffers](<workshop_results/Day_4/SKY130_D4_SK4 - Timing analysis with real clocks using openSTA/SKY_L5 - Lab steps to observe impact of bigger CTS buffers on setup and hold timing - using correct placement.def file.png>)

---

## Day 5 — Final Steps for RTL2GDS using TritonRoute and OpenSTA

### SK1 — Routing and Design Rule Check (DRC)

#### Maze Routing — Lee's Algorithm

Lee's algorithm labels grid cells outward from the source in a wave (1, 2, 3 …) until it reaches the target, then backtracks along decreasing labels to find the shortest path. Among equal-length paths, L-shaped routes with fewer bends are preferred. It always finds a path if one exists, but it is slow and memory-hungry on large designs, so production routers use faster variants.

![Grid routing concepts](<workshop_results/Day_5/SKY130_D5_SK1 - Routing and design rule check (DRC)/SKY_L2 - Lee’s Algorithm conclusion - Grid routing concepts.png>)

![Lee's algorithm result](<workshop_results/Day_5/SKY130_D5_SK1 - Routing and design rule check (DRC)/SSKY_L2 - Lee’s Algorithm conclusion – Lee’s algorithm - routing result consolidated.png>)

#### Design Rule Checks

Routing must obey the fab's rules:

- minimum **wire width** (set by lithography)
- minimum **wire spacing** and **wire pitch**
- **via** width and via spacing
- **signal shorts** — fixed by moving one net to another metal layer, which in turn must obey the via rules

![DRC rules](<workshop_results/Day_5/SKY130_D5_SK1 - Routing and design rule check (DRC)/SKY_L3 - Design Rule Check - wire width, spacing, pitch & shorting.png>)

### SK2 — Power Distribution Network and Routing

In OpenLANE v0.21 the PDN is generated after CTS, just before routing:

```tcl
gen_pdn
```

Power flows from the VDD/VSS pads to the **core ring**, through the vertical and horizontal **straps**, down to the **standard-cell rails**. The rail pitch is 2.72 µm (the `sky130_fd_sc_hd` cell height), which is why standard cells must be a multiple of that height.

![Power planning concepts](<workshop_results/Day_5/SKY130_D5_SK2 - Power Distribution Network and routing/SKY_L2 - Lab steps from power straps to std cell power - power planning concepts.png>)

#### Global and Detailed Routing

- **Global routing** (FastRoute) divides the chip into tiles (gcells), finds a coarse path for every net and produces **route guides**.
- **Detailed routing** (TritonRoute) draws the actual wires and vias inside those guides while meeting DRC rules.

![Fast and detailed routes](<workshop_results/Day_5/SKY130_D5_SK2 - Power Distribution Network and routing/SKY_L3 - Basics of global and detail routing and configure TritonRoute - Fast & Detailed routes.png>)

The routing strategy and other knobs are set through environment variables (`ROUTING_STRATEGY`, `GLB_RT_*`):

![Routing environment variables](<workshop_results/Day_5/SKY130_D5_SK2 - Power Distribution Network and routing/SKY_L3 - Basics of global and detail routing and configure TritonRoute - env variables for tweaking routing.png>)

### SK3 — TritonRoute Features

1. **Honours pre-processed route guides** — guides are split into unit widths, merged where they overlap, and bridged where they touch, so each guide has a preferred direction (for example, met1 vertical, met2 horizontal).
2. **Inter-guide connectivity** — two guides are connected if they overlap on the same layer or sit on adjacent layers with an overlapping projection; every unconnected pin must overlap a guide.
3. **Intra-layer parallel and inter-layer sequential routing** — panels on one layer are routed in parallel; layers are routed one after another.

![Pre-processing routes](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L1 - TritonRoute feature 1 - Honors pre-processed route guides - Preprocessing routes.png>)

![TritonRoute](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L1 - TritonRoute feature 1 - Honors pre-processed route guides - Triton Route.png>)

![Inter-guide connectivity](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L2 - TritonRoute Feature2 & 3 - Inter-guide connectivity.png>)

![Intra-layer parallel and inter-layer sequential routing](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L2 - TritonRoute Feature2 & 3 - Intra-layer parellel & inter-layer sequential routing.png>)

#### Handling Connectivity

TritonRoute's inputs are the LEF, DEF and pre-processed route guides; its outputs are the detailed routes with optimised wire length and via count. It uses access points (on-grid points where a pin or via can be reached) and solves a minimum-spanning-tree problem between access-point clusters.

![Connectivity considerations](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L3 - TritonRoute method to handle connectivity - considerations.png>)

![Handling connectivity](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L3 - TritonRoute method to handle connectivity - handling connectivity.png>)

![Routing topology algorithm](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Routing topology algorithm.png>)

![Fast routing guide](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Routing topology algorithm and final files list post-route - fast routing guide.png>)

### Lab — Running Routing

```tcl
echo $::env(CURRENT_DEF)
echo $::env(ROUTING_STRATEGY)
run_routing
```

TritonRoute runs repeated optimisation iterations. By the 33rd iteration every stage completed with **0 violations**:

![Routing iterations reduce violations](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Finally running run_routing in optimization iterations to reduce violations.png>)

![Zero DRC violations](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Finally running run_routing optimiation iterations achieve zero violations.png>)

![Routing completion](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Finally running run_routing completion  achieving best optimization.png>)

**Result:** `[INFO]: No DRC violations after detailed routing.`

| Layer | Wire length (µm) | Up-vias |
|---|---|---|
| li1 | 312 | 59,427 |
| met1 | 446,464 | 69,727 |
| met2 | 489,586 | 6,718 |
| met3 | 107,725 | 535 |
| met4 | 28,604 | 0 |
| met5 | 0 | — |
| **Total** | **1,072,693** | **136,407** |

Detailed routing took 13 min 17 s elapsed (48 min 21 s CPU) with a peak memory of 971.78 MB.

### Lab — SPEF Extraction and Post-Route STA

After routing, parasitics are extracted to a SPEF file ([`picorv32a.spef`](src/openlane/designs/picorv32a/runs/23-10_13-58/results/routing/picorv32a.spef)) and timing is re-run with the actual wire RC. In OpenLANE v0.21 the SPEF extractor is a separate Python script:

```bash
cd ~/Desktop/work/tools/SPEF_EXTRACTOR
python3 main.py <path-to-merged.lef> <path-to-routed.def>
```

![Generating SPEF](<workshop_results/Day_5/SKY130_D5_SK3 - TritonRoute Features/SKY_L4 - Routing topology algorithm and final files list post-route - generating spef file using python.png>)

Post-route STA with SPEF reported **WNS −16.90 ns** and **TNS −368.12 ns** (`reports/synthesis/31-opensta_spef_*.rpt`).

Final outputs of the run:

| File | Description |
|---|---|
| [`results/routing/picorv32a.def`](src/openlane/designs/picorv32a/runs/23-10_13-58/results/routing/picorv32a.def) | Routed layout (DEF) |
| [`results/routing/picorv32a.spef`](src/openlane/designs/picorv32a/runs/23-10_13-58/results/routing/picorv32a.spef) | Extracted parasitics |
| [`results/routing/picorv32a.def.png`](src/openlane/designs/picorv32a/runs/23-10_13-58/results/routing/picorv32a.def.png) | KLayout screenshot of the routed design |
| [`results/synthesis/picorv32a.synthesis_preroute.v`](src/openlane/designs/picorv32a/runs/23-10_13-58/results/synthesis/picorv32a.synthesis_preroute.v) | Final pre-route netlist |

---

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE v0.21** | Automated RTL-to-GDSII flow |
| **Yosys / ABC** | RTL synthesis and technology mapping |
| **OpenROAD 0.9.0** | Floorplan, placement, CTS, global routing, STA |
| **RePlAce / OpenDP** | Global and detailed placement |
| **TritonCTS** | Clock tree synthesis |
| **TritonRoute** | Detailed routing |
| **OpenSTA 2.3.0** | Static timing analysis |
| **Magic** | Layout viewing, editing, DRC and extraction |
| **ngspice** | SPICE simulation of the custom inverter |
| **KLayout** | Layout screenshots |
| **Sky130 PDK** | SkyWater 130 nm open-source process design kit |

---

## Key Learnings

- How a RISC-V core (`picorv32a`) moves from RTL to a routed, DRC-clean layout using only open-source tools
- How floorplan settings (core utilisation, aspect ratio, I/O mode) and placement density shape the physical design
- How a standard cell is built and characterised: layout in Magic, SPICE extraction, ngspice simulation, LEF generation
- How DRC rules are encoded in the Magic tech file, and how to find and fix an incorrect rule
- How to plug a custom cell into OpenLANE and verify it through synthesis, placement and timing
- How synthesis strategy, cell sizing, fanout limits, cell upsizing and CTS buffers affect setup and hold slack in OpenSTA
- How global and detailed routing work, and how post-route SPEF parasitics feed into final timing

---

## Acknowledgements

- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design), for the workshop content and labs
- **Nickson P Jose** — for the [`vsdstdcelldesign`](https://github.com/nickson-jose/vsdstdcelldesign) repository used in the Day 3 and Day 4 labs

---

## References

- [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/)
- [OpenLANE](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
- [Reference write-up by gowdanayu](https://github.com/gowdanayu/soc-design-and-planning-vsd)
