# Ring Oscillator-Based Phase-Locked Loop (RO-PLL)

### Chipathon 2026 Project Proposal | GF180MCU (GlobalFoundries 180nm)

## 1. Project Title

**RO-PLL-GF180: A Fully Verified, Open-Source Ring Oscillator-Based Phase-Locked Loop in GlobalFoundries 180nm CMOS**

> *Empowering the next generation of analog/mixed-signal IC designers through open, transparent, and rigorously verified PLL design.*

## 2. Executive Summary

This proposal presents the design, simulation, verification, and tapeout of a fully integrated, open-source Ring Oscillator-Based Phase-Locked Loop (RO-PLL) implemented in GlobalFoundries' GF180MCU 180nm CMOS process. The system targets a wide frequency output range of **100 MHz to 1.2 GHz**, programmable division ratios from 1 to 32, and total power consumption below **8–10 mW**, specifications that make it immediately useful as clock generation IP for digital SoCs, microcontrollers, IoT endpoints, and educational mixed-signal platforms. The entire design flow, from schematic capture through post-layout simulation, will be conducted using freely available open-source EDA tools (Xschem, ngspice, Magic VLSI, KLayout, and OpenLane-compatible utilities), producing a reusable IP block and a comprehensive, well-documented design notebook accessible to anyone worldwide.

Every design decision, from the topology choice of a current-starved ring oscillator to the sizing of the charge pump's current mirrors will be documented, justified, and linked to simulation evidence. The verification campaign spans thousands of Monte Carlo runs for mismatch and process variation, full PVT corner analysis (FF/TT/SS × ±10% supply × −40°C to 125°C), transient lock-time measurements, phase noise characterization, and post-layout parasitic extraction. By taping out under the Chipathon 2026 banner, we contribute a fully tested, silicon-proven PLL tile that subsequent teams can instantiate, modify, and build upon; advancing the open-source analog IC ecosystem in a meaningful and lasting way.

## 3. Motivation and Target Applications

### Why a PLL?

The Phase-Locked Loop is arguably the most ubiquitous mixed-signal building block in modern semiconductor design. It sits at the intersection of analog and digital domains, synthesizing precise clock frequencies from a stable reference, suppressing jitter, and enabling clock multiplication; functions required by virtually every digital system. Despite its ubiquity, high-quality, openly available, silicon-proven PLL IP in open-source PDKs remains extremely scarce. Most educational treatments stop at SPICE schematics; few teams complete the full journey through layout, extraction, and verified post-layout simulation.

### Why Ring Oscillator?

While LC-tank VCOs offer superior phase noise, they require on-chip inductors that are large, lossy, and process-sensitive at 180nm. A **current-starved ring oscillator VCO** trades some phase noise performance for compactness, wide tuning range, straightforward layout, and excellent portability across process nodes, all ideal qualities for an educational open-source IP project. The design lends itself naturally to analysis, and its performance is entirely predictable from first-principles MOSFET theory, making it a superb teaching vehicle.

### Target Applications

| Application Domain | How the RO-PLL Helps |
| - | - |
| Digital SoC Clock Generation | Synthesizes on-chip clocks from a 10–50 MHz crystal reference |
| Microcontroller Clock Multiplication | Provides high-speed core clocks from a low-frequency oscillator |
| IoT & Edge Devices | Low-power PLL suits always-on clock domains |
| ADC/DAC Clocking | Wide tuning range accommodates sampling rate flexibility |
| Academic & Teaching Labs | Fully documented, reproducible design for coursework |
| Open-Source SoC Ecosystems | Reusable tile for Caravel/OpenROAD-based SoC integration |


## 4. Proposed Architecture

The RO-PLL is a classical integer-N charge-pump PLL with the following hierarchy:

![Proposed RO-PLL Architecture](https://raw.githubusercontent.com/OlonadeKelvin/ro-pll-gf180/main/Images/RO-PLL.drawio.webp)
### 4.1 Current-Starved Ring VCO (CS-RVCO)

The heart of the system is a **current-starved differential/inverter-based ring oscillator** with selectable 5-stage or 7-stage configurations (controlled by a digital pin). Each stage is a pseudo-differential current-starved inverter: PMOS pull-up and NMOS pull-down transistors are loaded by cascode current sources, whose gate voltages are driven by the VCO control voltage (V\_ctrl) through a voltage-to-current converter. This architecture provides:

- **Wide tuning range**: oscillation frequency is controlled by varying the bias current through each stage.

- **5-stage mode**: optimized for the upper frequency band (~600 MHz – 1.2 GHz) with lower stage delay.

- **7-stage mode**: optimized for the lower frequency band (~100 MHz – 600 MHz) with improved phase noise due to more averaging stages.

- **Configurable gain (K\_VCO)**: achieved through a switched bank of tail current sources.

### 4.2 Phase-Frequency Detector (PFD)

A standard **dual-D-flip-flop with reset** topology is adopted, with special attention to dead-zone elimination. The key innovations are:

- **Carefully sized reset path delay**: a tunable delay buffer in the reset path ensures both UP and DOWN pulses have a finite minimum width (~200–500 ps), preventing the charge pump from operating in its nonlinear dead zone.

- **Matched layout**: the UP and DOWN signal paths are kept symmetric in layout to minimize static phase offset.

- Implementation entirely in digital standard cells, enabling straightforward functional verification.

### 4.3 Charge Pump (CP)

The charge pump converts UP/DOWN pulses into a current that charges or discharges the loop filter. Key features:

- **Current matching**: PMOS and NMOS current sources are sized and biased to achieve \< 2% static mismatch under typical conditions.

- **Cascode current sources**: improve output impedance, reducing static phase offset due to current mismatch across the output voltage swing.

- **Programmable current (I\_CP)**: 3-bit binary-weighted switch bank allows I\_CP to be set to 10, 20, 40, or 80 µA, enabling loop bandwidth tuning without hardware changes.

- **Switch bootstrapping**: reduces charge injection and clock feedthrough from the switching transistors.

### 4.4 Passive Loop Filter (PLF)

A **third-order passive loop filter** (one zero, two poles) is used to attenuate reference spurs and shape the open-loop transfer function:

- **Second-order core (R1, C1, C2)**: provides the stabilizing zero and dominant pole.

- **Third-order extension (R2, C3)**: adds a second pole for additional reference spur suppression.

- Component selection guidelines will be provided in the documentation, derived from the loop bandwidth and phase margin targets (φ\_m ≥ 55°).

- All passive components are sized for on-chip integration in GF180MCU's MIM capacitor and poly resistor layers.

### 4.5 Programmable Integer Frequency Divider

A **synchronous divide-by-N counter** with N programmable from 1 to 32 (5-bit), implemented using D-flip-flops in the GF180MCU digital cells. Features:

- Glitch-free output through registered decode logic.

- Divide-by-2 prescaler at the VCO output to relax timing at the highest frequencies.

- Full functional verification via digital simulation (ngspice/Icarus Verilog).

### 4.6 Lock Detector

A **digital windowed lock detector** monitors the phase error between REF and DIV signals. When the phase error remains below a programmable threshold for a configurable number of consecutive reference cycles, the LOCK output asserts high. This provides a clean, debounced lock indicator suitable for system integration.

### 4.7 Power-Down Mode

A global **PWR\_DN** pin places the entire PLL into a low-leakage state: the VCO bias current is cut, the charge pump is disabled, and the digital logic is clock-gated. Wake-up is achieved by deasserting PWR\_DN; the PLL re-acquires lock within the designed lock time.

### 4.8 Output Buffer

A **tapered inverter chain** with programmable drive strength buffers the VCO output to a 50-Ω-compatible swing for probing and system use. An optional differential CML output stage is included for lower jitter at high frequencies.

## 5. Key Specifications

| Parameter | Target | Notes |
| - | - | - |
| **Process** | GF180MCU 180nm CMOS | Open-source PDK, supported by ngspice, Magic, KLayout |
| **Supply Voltage** | 1.8 V (core), 3.3 V (I/O) | GF180MCU standard |
| **Reference Frequency** | 10 – 50 MHz | External crystal or TCXO |
| **Output Frequency Range** | 100 MHz – 1.2 GHz | 5-stage and 7-stage modes combined |
| **Division Ratio (N)** | 1 – 32 (integer) | 5-bit programmable |
| **VCO Gain (K\_VCO)** | 200 – 800 MHz/V (typ.) | Depends on mode and process corner |
| **Loop Bandwidth** | 1 – 5 MHz | Programmable via I\_CP selection |
| **Phase Margin** | ≥ 55° | Ensures stable lock across PVT |
| **Lock Time** | \< 10 µs | From cold start at 10 MHz reference |
| **Total Power** | \< 8 mW @ TT, 1.8V, 27°C | Including VCO, CP, divider, buffer |
| **Phase Noise (1 GHz out)** | \< −85 dBc/Hz @ 1 MHz offset | Ring oscillator limited; acceptable for digital clocking |
| **Reference Spur Rejection** | \< −40 dBc | Third-order loop filter |
| **Charge Pump Current (I\_CP)** | 10 – 80 µA (4 settings) | 3-bit programmable |
| **Charge Pump Mismatch** | \< 2% (typ.) | Cascode current mirror matching |
| **Operating Temperature** | −40°C to +125°C | Full industrial range |
| **Supply Variation** | ±10% of nominal | 1.62V – 1.98V |
| **Core Area (target)** | \< 0.15 mm² | Including loop filter passives |
| **Output Swing** | CMOS rail-to-rail | Buffered single-ended output |


## 6. Design Features and Innovations

### 6.1 Configurable Ring Oscillator Topology

Offering both 5-stage and 7-stage modes in a single instantiation, selectable at runtime, is a key differentiator. This allows the same silicon to serve two frequency sub-bands, maximizes testability (both modes can be characterized on every die), and provides insight into how stage count affects oscillation frequency, phase noise, and power.

### 6.2 Charge Pump Current Programmability

The 3-bit I\_CP switch bank allows in-system loop bandwidth adjustment without changing hardware, a practical feature for educational experimentation. Teams can observe, in real time, how changing I\_CP affects lock time, reference spur levels, and phase noise.

### 6.3 Dead-Zone-Free PFD Implementation

The reset-path delay buffer is independently tunable (via a switched capacitor load on the reset wire), and its effect on PFD linearity will be characterized through transient simulation and measured via phase noise at the output. This is an often-overlooked subtlety that will be explicitly taught and documented.

### 6.4 Comprehensive Verification Strategy

This is a cornerstone of the project philosophy. **No block will be taped out without passing a defined verification checklist.**

#### Block-Level Verification

| Block | Simulations | Pass Criteria |
| - | - | - |
| Ring VCO | Tuning curve (V\_ctrl sweep), K\_VCO extraction, startup transient, phase noise (using PSS/PNOISE or Spectre-equivalent) | Frequency range covers spec across all corners |
| PFD | Functional (lead/lag detection), dead zone characterization, timing diagram vs. phase error | Zero dead zone at reset delay \> 200 ps |
| Charge Pump | I\_CP vs. V\_out, UP/DOWN mismatch, charge injection | Mismatch \< 2% over V\_out swing |
| Loop Filter | AC transfer function (Bode plot), noise contribution | Phase margin ≥ 55°, correct pole/zero placement |
| Divider | Divide ratio 1–32, max toggle frequency | Correct division at all N, no glitches |
| Lock Detector | Lock/unlock detection transient | Correct assertion within 2 reference cycles of lock |
| Full PLL | Lock transient, output spectrum, phase noise, spur levels | All specs in Table 1 met |


#### PVT Corner Analysis

All blocks will be simulated across the full **27-corner matrix**:

```
Process:     FF, TT, SS  (3 corners)  
Voltage:     1.62V, 1.80V, 1.98V  (3 levels, ±10%)  
Temperature: −40°C, 27°C, 125°C  (3 points)  
─────────────────────────────────────────────  
Total:       3 × 3 × 3 = 27 corners
```

Pass/fail status for each spec will be recorded in a **corner summary table** committed to the GitHub repository.

#### Monte Carlo Simulations

Two Monte Carlo campaigns will be run using the GF180MCU mismatch and process variation models:

- **Mismatch Monte Carlo (N = 500 runs)**: Device-to-device Pelgrom mismatch within a die. Focuses on charge pump current mismatch, VCO stage delay variation, and PFD timing asymmetry.

- **Process Monte Carlo (N = 500 runs)**: Die-to-die global process parameter variation (V\_th, µ\_n, µ\_p, t\_ox). Evaluates yield of meeting frequency range, power, and lock-time specs.

Statistical results (mean, σ, min, max, yield at ±3σ) will be reported for all key specs.

#### Phase Noise Analysis

Phase noise will be estimated via:

1. **Hand calculation** using Leeson's model extended to ring oscillators.

2. **ngspice transient noise simulation**: noise sources injected and FFT of output jitter.

3. **Post-tapeout measurement** using an available spectrum analyzer or phase noise analyzer.

### 6.5 Open-Source Reproducibility

Every simulation, schematic, layout file, and result will be version-controlled on GitHub with:

- Automated regression scripts (shell/Python) to re-run all simulations from scratch.

- Simulation result plots committed as PDF/PNG.

- Design documentation in Jupyter Notebooks for interactive learning.

- A README.md with step-by-step instructions for running every simulation.

## 7. Implementation Plan

### 7.1 EDA Toolchain

| Task | Tool | Notes |
| - | - | - |
| Schematic Capture | Xschem | Open-source, GF180MCU PDK integration |
| SPICE Simulation | ngspice | Transient, AC, noise, Monte Carlo |
| Layout | Magic VLSI | Full custom analog layout |
| DRC | Magic + KLayout | Both tools run DRC against GF180MCU rules |
| LVS | Netgen | Schematic vs. layout comparison |
| Parasitic Extraction | Magic (RCX) | RC extraction for post-layout simulation |
| Digital RTL | Icarus Verilog | Divider and lock detector functional sim |
| Documentation | Jupyter Notebook, LaTeX/Markdown | Living design notebook |
| Version Control | Git + GitHub | All files, results, and scripts |


### 7.2 Project Phases and Timeline

| Phase | Duration | Key Deliverables |
| - | - | - |
| **Phase 0**: Environment Setup & Learning | 2 weeks | PDK installed, tools working, inverter simulated |
| **Phase 1**: Block Design & Simulation | 8 weeks | All blocks designed, schematic-level sim passing all corners |
| **Phase 2**: Integration & PLL Simulation | 3 weeks | Full PLL lock transient, phase noise, spur analysis |
| **Phase 3**: Layout | 6 weeks | Full custom layout, DRC/LVS clean |
| **Phase 4**: Post-Layout Verification | 3 weeks | Parasitic-extracted sim, corner re-check |
| **Phase 5**: Tapeout Preparation | 1 week | GDS export, final DRC/LVS, documentation freeze |
| **Phase 6**: Documentation & Open-Source Release | Ongoing | GitHub, Jupyter Notebooks, final report |


### 7.3 Team Structure

| Role | Responsibilities |
| - | - |
| Sirajul| Phase detector implementation, divider/lock detector design, system-level PLL simulation, and lock analysis. |
| Nandan| Ring oscillator design, tuning, phase noise characterization, and output buffer implementation. |
| Kelvin| Charge pump design, dead zone characterization, and loop filter design. |

Links
[Github repo(s)](https://github.com/OlonadeKelvin/ro-pll-gf180)
[Proposal Slide Link]()
