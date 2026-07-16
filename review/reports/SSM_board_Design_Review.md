# SolidStateSwitchMatrix_board (SSM_board) Design Review

**Project:** SSM_board / title block name "SolidstateSwitchMatrix_board", rev 0.1 (2026-03-29)
**KiCad version:** 9.0 (file format 20250114), hierarchical design — root sheet + `Conn` + `Frontend`, with `Frontend` instantiating one shared `driver.kicad_sch` template 8× (`Driver`, `Driver1`–`Driver7`)
**Board:** 100 × 100 mm, 4-layer (F.Cu / In1.Cu / In2.Cu / B.Cu), 1.586 mm finished thickness, 184 footprints, fully routed (0 unrouted nets)
**Date of this review:** 2026-07-16
**Analyzers run:** `analyze_schematic.py`, `analyze_pcb.py --full --proximity`, `cross_analysis.py`, `analyze_emc.py` (skill `emc`), `analyze_thermal.py`, `lifecycle_audit.py`. SPICE **not run** — no ngspice/LTspice/Xyce installed in this environment.

## Overview

This is a 64-channel solid-state relay (photoMOS/OptoMOS) switch matrix card. Eight `STPIC6C595TTR` power-logic shift registers (one per repeated `Driver`/`Driver1-7` sub-sheet, cascaded SER_OUT→SER_IN into a single 64-bit shift chain clocked by `SSR_SCK`/`SSR_CS`/`SSR_MOSI`) each sink-drive 8 `GAQY212GS` 1-Form-A photoMOS relays (60 V / 0.8 A, SOP-4) through a 470 Ω LED series resistor. Relay load (switched) contacts are bussed in groups and brought out to two DA-15 D-Sub connectors (J1, J2) plus a large 80-pin backplane connector (X1) that supplies every rail used on the board (+5 V, +12 V, −12 V, +3.3 V, GND, Earth, Earth_Protective) — **no on-board regulation exists**, confirmed by the schematic analyzer finding no `power_regulators` detections anywhere in the design. The `HV` net class (0.55 mm track / 0.35 mm clearance, applied via the `Net-(J*` pattern in `SSM_board.kicad_pro`) confirms the designer treats the relay load wiring to J1/J2 as higher-voltage/higher-current than logic signals.

## Previous Review Delta

No prior review file or prior analyzer run was found in the project directory — this is the first recorded review.

## Critical Findings

| Severity | Issue | Section |
|----------|-------|---------|
| WARNING | No hardware power-on default/reset for the 64 relay outputs — `G̅` (output enable) is hard-wired to GND on all 8 shift registers and `CLR̅` is only pulled high (not held low), so outputs can show whatever the shift/storage registers power up with until firmware initializes them | Signal Analysis Review → Design Observations |
| WARNING | J1 (and by extension the relay-matrix I/O it carries) has no ferrite/CM choke/ESD protection within 25 mm — unfiltered external I/O cable | EMC / Cross-Domain Analysis |
| WARNING | `/Conn/SSR_SCK` (the clock shared by all 8 cascaded shift registers) has 9 of 10 layer transitions with no nearby ground stitching via | EMC / Cross-Domain Analysis |
| WARNING | 4-layer stackup has two adjacent signal layers (F.Cu, In1.Cu) with no reference plane between them | EMC / Cross-Domain Analysis |
| WARNING | Ground-plane gaps under `-12V`, `Earth`, and 4 relay-drain nets on the `Driver5` sub-sheet (Q4–Q7) | EMC / Cross-Domain Analysis |
| SUGGESTION | Relay symbols (`Relay_SolidState:CPC1006N`) still carry the stock CPC1006N `Datasheet`/`Description` properties (0.075 A, 10 Ω) instead of the fitted GAQY212GS's real ratings (0.8 A, 0.24 Ω) — documentation only, pin function is unaffected (see verification below) | Analyzer Verification |
| SUGGESTION | 2/8 unique BOM parts still missing MPNs (`SS-002`, 75% coverage); `WLBD1608HCU101TL` and `CRCW060310K0FKTABC` could not be matched against LCSC's catalog during this review | Component Summary |

No CRITICAL (board-will-not-function) issues were found. The two-datasheet-verified critical ICs (shift register and relay) are wired correctly.

## Component Summary

| Type | Count |
|---|---|
| GAQY212GS photoMOS relay (SOP-4) | 64 |
| STPIC6C595TTR shift register (TSSOP16) | 8 |
| Resistor 50 kΩ (LED bias, parallel to relay input) | 64 |
| Resistor network 470 Ω × 4 (`YC164-FR-07470R`, LED series) | 16 (64 elements) |
| Resistor 10 kΩ (CLR̅ pull-up, one per shift-register) | 8 |
| Ferrite bead 100 Ω@100 MHz (per-IC VCC filter) | 8 |
| Capacitor 100 nF (per-IC VCC bypass) | 8 |
| Capacitor 1 nF/2 kV (Earth ↔ Earth_Protective) | 3 |
| Connectors (J1, J2 = DA-15; X1 = 80-pin backplane) | 3 |
| Mounting holes | 2 |
| **Total** | **184** |

Nets: 261 (schematic) / 245 (PCB, power-symbol collapse accounts for the difference) · Wires: 1385 · No-connects: 57 · Unique BOM line items: 8 · MPN coverage: 6/8 (75%, `SS-002`) · Datasheet coverage: manually completed for the two ICs during this review (see below); 4/8 unique parts now have a verified datasheet.

Power rails present: `+12V`, `+3.3V`, `+5V`, `-12V`, `Earth`, `Earth_Protective`, `GND` — all sourced externally through the 80-pin `X1` backplane connector (confirmed by net trace: `X1` pins 1/2→+12V, 3/4→+5V, 5/6→+3.3V, 7/8→−12V, `EARTH`→Earth, 9/10/12/14/15/16/74-80→GND). `RS-001` flagging these rails as having "no declared source" is **expected, not a defect** — this is a plug-in driver card, not a self-powered board.

## Power Tree

```
X1 (80-pin backplane connector, external supply)
 ├─ +12V ──────────────────────────────► (not consumed on this sheet — routed through, no local load found)
 ├─ -12V ──────────────────────────────► (not consumed on this sheet — routed through, no local load found)
 ├─ +3.3V ─────────────────────────────► (not consumed on this sheet — routed through, no local load found)
 ├─ +5V ─┬─► FB1..FB8 (100R@100MHz) ─┬─► IC1..IC8 VCC (STPIC6C595TTR pin 1)
 │       │                          └─► C1..C8 (100nF) → GND  [local decoupling, verified 1.45mm from IC pins on PCB]
 │       └─► R1..R8 (10k) ──────────► IC1..IC8 CLR̅ (pin 7)  [pull-up, no local cap → no RC delay on CLR̅, see finding below]
 │       └─► U1xx..U7xx pin1 (LED anode, all 64 relays) ─┬─► RN_x (470R) ─► IC_x DRAINn (sink to GND when ON)
 │                                                        └─► R_x (50k, parallel bias, ≈0.1mA leakage)
 └─ GND / Earth / Earth_Protective ── C9,C10,C11 (1nF/2kV) bridge Earth↔Earth_Protective (EMI/safety bonding, NOT relay snubbers)
```

No on-board regulator was detected anywhere in the design (`power_regulators` detector: 0 findings) — every rail is a straight pass-through from `X1`. `+12V`/`-12V`/`+3.3V` are present at `X1` but no consuming component was found on this board's own sheets; they are most likely used downstream on `Frontend` for signal-conditioning circuitry outside the scope of the 64-channel relay matrix (the `Frontend.kicad_sch` sheet is 678 KB — substantially larger than the 45 KB `driver.kicad_sch` — and clearly contains additional circuitry not covered by this relay-matrix-focused deep dive; see Review Limits).

## Analyzer Verification

### Component Count
Schematic analyzer: 184 total components. PCB analyzer: 184 footprints. **Exact match.** Raw-file spot check confirmed the hierarchy the analyzer reported: `SSM_board.kicad_sch` instantiates only `Frontend` and `Conn`; `Frontend.kicad_sch` (line 36799 onward) instantiates `driver.kicad_sch` eight times as `Driver`/`Driver1`–`Driver7`, each contributing 8 relays + 1 shift register + 1 pull-up + 1 ferrite bead + 2 resistor networks — accounting for all 64 relays / 8 ICs / 8×(FB, R, 2×RN).

### Component Pinout Verification (datasheet-verified)

| Ref (×qty) | Value | Pins | Datasheet Verified | Status |
|---|---|---|---|---|
| U101–U712 (×64) | GAQY212GS | 1=LED Anode(+), 2=LED Cathode(−), 3/4=MOSFET Load (no polarity) | **Verified (datasheet)** — GJ Semiconductor GAQY212GS Rev 1 Dec 2016, p.1 pin diagram | Pin function matches; see symbol-mismatch note below |
| IC1–IC8 (×8) | STPIC6C595TTR | 1=VCC,2=SER IN,3-6=DRAIN0-3,7=CLR̅,8=G̅,9=SER OUT,10=RCK,11-14=DRAIN4-7,15=SRCK,16=GND | **Verified (datasheet)** — STMicroelectronics STPIC6C595 Rev 5, Figure 1 (identical pinout for SO-16 and TSSOP16) | **Fully matches schematic net assignment, pin-for-pin** |
| RN1–RN16 (×16, 64 elements) | 470R (YC164-FR-07470R, isolated ×4 array) | 4 independent 2-terminal resistors, pins (1,8)(2,7)(3,6)(4,5) | Raw-file verified (net trace) | Correct isolated (non-bussed) pack topology confirmed via pin-to-net mapping |
| R101–R712 (×64) | 50k | 2-pin passive, parallel with relay LED input | Skipped (2-pin passive) | See LED-bias explanation below |
| J1, J2 | DA15_Pins_MountingHoles | Standard KiCad D-Sub symbol/footprint | Skipped (standard connector) | Pin 0/14/15 = Earth_Protective (shield/ground) |
| X1 | Conn:UCB_Conn (80-pin) | Custom project symbol | Unverified — no external datasheet, but pin-net assignment is internally consistent and matches rail names | Plausible; low risk (passive pinout, no signal-integrity-critical function) |

**Symbol/datasheet mismatch (SUGGESTION, not a wiring defect):** All 64 relay instances use the stock KiCad library symbol `Relay_SolidState:CPC1006N`. Its embedded `Datasheet` property points to Littelfuse/IXYS's CPC1006N datasheet, and its `Description` field still reads "60V, 0.075A, 10Ohm" — the CPC1006N's own ratings, not the fitted GAQY212GS's (60V, **0.8A**, **0.24Ω**). This is purely a documentation/BOM-traceability issue: I independently pulled the actual CPC1006N datasheet (Littelfuse/IXYS DS-CPC1006N, via Mouser/IXYS mirror) and confirmed its pin assignment is **identical** to GAQY212GS's (1=Anode, 2=Cathode, 3/4=Load) — both are examples of the near-universal SOP-4 1-Form-A OptoMOS pinout convention, so there is no rewiring risk. Recommend updating the `Value`/`Description`/`Datasheet` properties on the 64 relay symbols to reference GAQY212GS directly so a future BOM export or datasheet pull doesn't surface the wrong current/resistance rating.

### Net Tracing — representative channel (U101/IC1, verified in full, then confirmed structurally identical across all 64 channels via net trace)

```
+5V ──┬── U101.1 (LED Anode) ══[LED]══ U101.2 (LED Cathode) ──┬── net "101"
      └── R101 (50k) ─────────────────────────────────────────┘
                                                          net "101" ── RN1 element R2 (470R) ── IC1.3 DRAIN0 (sink→GND when ON)
U101.3 ── shared bus net (also carries U201.3, U301.3, U401.3, U501.3) ── J1 pin 6
U101.4 ── shared bus net (also carries U102-108.4) ── J1 pin 1
```

- **LED drive current check:** IF = (5V − VF,typ 1.4V) / 470Ω ≈ **7.7 mA**, squarely inside GAQY212GS's recommended 5–30 mA (typ. 7 mA) range and close to the datasheet's own worked example (600Ω → 6mA @ 5V). **Verified correct, no under/over-drive.**
- **R101 (50k) function:** electrically in parallel with the relay's LED, between +5V and node "101". At LED forward conduction its 50k branch is negligible (<2% of total current) — it is not a current-limiting element. Its role is to weakly bias node "101" toward +5V when the driving DRAIN output is open (Hi-Z), keeping the LED near 0V bias instead of floating when off. Reasonable design practice, not a defect.
- **C9/C10/C11 (1nF/2kV) — corrected initial assumption:** I initially expected these (given only 3 exist, not one per channel) to be the CR snubbers the GAQY212GS datasheet recommends for inductive loads (p.5 "Protection Circuit"). Net trace shows they are **not** — C9 bridges `Earth` (pin1) to `Earth_Protective` (pin2), i.e. all three are Y-capacitor-style EMI/safety bonding caps between the signal-earth and protective-earth domains, unrelated to individual relay snubbing. **No per-channel snubber/clamp diode exists on this board** — per the datasheet's own guidance ("clamp diode... CR Snubber... connected in parallel with the load... installed near the MOS RELAY to be effective"), any inductive-load protection is expected to live off-board, near the actual load. Worth confirming with the system integrator that downstream wiring/loads don't require on-card protection.
- **Bussed load topology:** relay pin-3/pin-4 outputs are bussed in groups (e.g. all `Uxx1.3` from sheets 1–5 tied together to J1 pin 6; all `U10x.4` from sheet 1 tied together to J1 pin 1) rather than each channel getting a dedicated 1:1 connector pin. This is consistent with a genuine crosspoint/matrix switch topology (rows/columns), matching the board's own title ("SolidstateSwitchMatrix") and the GAQY212GS datasheet's listed applications ("Telecom/Datacom switching, Multiplexers"). Full row/column mapping across all 64 channels was not exhaustively re-derived pin-by-pin (see Review Limits) — the representative trace above and the "all 64 pin-1→+5V" structural check (below) are what this review verified directly.
- **Structural consistency across all 64 channels (programmatic, not manual sampling):** every one of the 64 `Uxxx.1` pins connects to `+5V` with zero exceptions — confirms the LED-drive topology verified above for channel U101 applies uniformly, not just to the sampled instance.

### Connector Pin Tables

**X1 (80-pin backplane, `Conn:UCB_Conn`)** — key pins only (full 80-pin map available in `analysis/`):

| Pin(s) | Net | Function |
|---|---|---|
| 1, 2 | +12V | Power in |
| 3, 4 | +5V | Power in (logic) |
| 5, 6 | +3.3V | Power in |
| 7, 8 | −12V | Power in |
| EARTH | Earth | Signal earth reference |
| 9,10,12,14,15,16,74-80 | GND | Digital ground |
| 33 | SSR_MOSI | Shift-register serial data in |
| 35 | SSR_SCK | Shift-register clock |
| 36 | SSR_CS | Register clock (latch) |
| 11,13,17,19...73 (odd) / 18,20...72 (even) | unnamed, per-pin | Additional Frontend I/O — out of scope for this driver-focused review |

**J1 / J2 (DA-15, D-Sub 15-pin male, low-density)**

| Pin | J1 net | J2 net | Note |
|---|---|---|---|
| 0 (shield/PAD) | Earth_Protective | Earth_Protective | Chassis/shield bond |
| 1 | bus: U101-108.4 | (per-channel, sheet-specific) | Relay-bank common |
| 6 | bus: U101/201/301/401/501.3 | — | Cross-sheet channel-1 common |
| 14, 15 | Earth_Protective | Earth_Protective | Shield/chassis |
| 2–13 (remaining) | per-channel bus nets | per-channel bus nets | Matrix row/column outputs |

**Note on `CG-AUD` false positive:** the schematic analyzer flagged "Connector J1/J2 has no ground pins" (`CG-AUD`, warning). This is a false positive — both connectors *do* have shield/chassis-ground pins (0, 14, 15 → `Earth_Protective`); the detector's heuristic only recognizes literal `GND`-named nets and doesn't treat `Earth_Protective` as ground-equivalent. Dismissed.

### PCB Verification
- Footprint count matches schematic component count exactly (184 = 184).
- Board outline: clean rectangular 100×100mm, single closed edge (`Edge.Cuts`), 4-layer stackup confirmed from `pcb.setup.stackup` (F.Cu/In1.Cu/In2.Cu/B.Cu, 1.586mm total).
- Pad-to-net spot check for IC1/IC4 (STPIC6C595) and U101 (GAQY212GS) confirmed against the schematic pin-net map — no mismatches found.
- Routing 100% complete, 0 unrouted nets, DFM tier "standard" with 0 violations (min track 0.3mm, min drill 0.3mm — comfortably within JLCPCB standard-tier capability).
- HV load traces to J1 (`Net-(J1-Pad1/2/6/7/8...)`) use 0.55mm width on multiple layers (F.Cu/B.Cu/In2.Cu), consistent with the `HV` net class definition (0.55mm track/0.35mm clearance). Per IPC-2221A, 0.55mm 1oz external-layer copper supports well over 1A at a conservative 10°C rise — comfortably covers the 0.8A per-channel GAQY212GS rating, including margin for multiple bussed channels.

## Signal Analysis Review

### Design Observations (manual finding, not from an automated detector)
**No hardware power-on-reset/default-safe-state for the 64 relay outputs.** On every one of the 8 `STPIC6C595TTR`s: `G̅` (output enable) is hard-wired directly to `GND` (net trace: `IC1.8 → GND`), meaning outputs are **always enabled** — there is no way to force all drains off at the hardware level. `CLR̅` is pulled to VCC through a 10kΩ resistor (`R1`–`R8`) with **no capacitor on the CLR̅ node itself** (confirmed by net trace: `C1`'s far pin bypasses the *local VCC* node, not CLR̅ — see false-positive note below), so there is no incidental RC power-on delay either. Per the STPIC6C595 datasheet, the shift/storage registers' power-up state is not specified as all-zero. Combined with `G̅` permanently low, this means the 64 relay outputs could briefly show an undefined pattern immediately after power-up, until firmware actively clocks/latches a known-safe (all-off) word. Given this board switches external loads through J1/J2, recommend confirming firmware initializes the shift chain (`SSR_SCK`/`SSR_CS`/`SSR_MOSI`) within a bounded, short time of power-good, or consider adding a hardware power-on-clear network if a guaranteed-safe startup state matters for the application.

### RC Filters — false positive
`detect_rc_filters` reported "RC filter R1/C1 at 159.15Hz" (and R2/C2 … R8/C8, 8 total, one per driver sub-sheet). Net trace shows this is a **topology misdetection**: `R1` (10k) sits between `CLR̅` and local-VCC; `C1` (100nF) sits between local-VCC and GND. They share the local-VCC node but are not arranged as a series R→C low-pass with a distinguishable filtered output — `R1` is a digital pull-up, `C1` is a supply bypass cap. No genuine 159Hz filter exists in this circuit. (This also confirms the "no RC delay on CLR̅" statement above.)

### Decoupling Analysis
| Rail | Cap count | Notes |
|---|---|---|
| Local VCC (per IC1-8, behind ferrite) | 1× 100nF each | PCB-verified placement: 1.45mm from IC pins, shares GND + VCC net with the IC — **good practice, verified correct** |
| Relay Uxxx (64×) | 0 (intentional) | GAQY212GS has no VCC/GND supply pin — pins are LED I/O (1,2) and floating MOSFET load (3,4). Decoupling does not apply. |

**EMC false-positive (major, see EMC section):** the EMC analyzer's `DC-001`/`DC-002` decoupling checks fired 64 times — once for **every** GAQY212GS relay — because its heuristic treats every `category:"ic"` component as needing local VCC bypass. Since the relays have no power-supply pin to decouple, all 64 of these findings are false positives. This single pattern accounts for 64 of the EMC analyzer's 73 "error"-severity findings (88%) — see EMC section for full triage.

### ESD Protection
`audit_esd_protection` reports "none" coverage on J1, J2, X1 (info-level, generic detector — doesn't know these are a relay-matrix output vs. a digital I/O connector). No TVS/ESD array was found on any of the three connectors. Given J1/J2 carry externally-wired relay loads (not digital signal lines, per the `HV` net class), standard digital-I/O ESD guidance doesn't directly apply, but the EMC analyzer's `IO-001` finding (below) — no ferrite/CM choke/protection near J1 at all — is the more relevant, real concern for these cable-facing connectors.

### Label Aliasing — expected, not a defect
`detect_label_aliases` (`LB-001`, 10 findings, all info) flags the `SER_OUT`→`SER_IN` net-name changes at each hierarchical sheet boundary in the shift-register daisy chain, plus the `SSR_SCK`/`SSR_CS`/`SSR_MOSI` bus fan-out to all 8 sheets. This is exactly how a cascaded-shift-register bus should look across hierarchical sheets — confirmed benign.

## Power Analysis

**PDN impedance, power budget, sleep current, inrush:** not applicable in the conventional sense — this board has no on-board regulator and no sleep/low-power mode; all rails are supplied externally through `X1` with no local generation to characterize. The only local "regulation" is the FB+100nF filter on each IC's VCC, which is a straightforward supply-noise filter (0603 100Ω@100MHz bead + 100nF, verified in place and correctly placed on all 8 sheets).

**Voltage derating:** the 1nF/2kV caps (C9-11) bridging Earth/Earth_Protective are rated far above any plausible earth-to-earth potential difference in normal operation — comfortable margin, no concern.

## EMC / Cross-Domain Analysis

EMC analyzer (`emc` skill): 130 findings (73 error / 52 warning / 5 info), `risk_level: high` as reported. **This headline is misleading before triage:**

| Rule | Count | Disposition |
|---|---|---|
| `DC-002` "No decoupling cap found near [relay]" | 59 | **False positive** — all 59 target `Uxxx` GAQY212GS relays, which have no VCC/GND pin (see Decoupling Analysis above) |
| `DC-001` "Decoupling cap too far from [relay]" | 5 | **False positive** — same root cause, targets `U101/U201/U301/U401/U501` |
| `GP-001` "Signal has significant reference plane gap" | 6 error + 20 warning | **Real** — see below |
| `RP-001` "Missing stitching via at layer transition" | 1 error + 21 warning | **Real** — see below |
| `IO-001` "No EMC filtering near J1" | 1 | **Real** |
| `SU-001` "Adjacent signal layers without reference plane" | 1 | **Real** |
| `BE-001` "Signal near board edge" | 10 | Informational — expected for a board with edge-mounted D-Sub connectors |
| `CK-002`, `SU-002`, `EE-001` | 1 each | Minor, see `analysis/2026-07-16_1342/emc.json` for detail |

After removing the 64 relay-decoupling false positives, the real error-level EMC finding count is **9**, not 73:

- **`-12V`**: 50% plane coverage over 4.9mm of routing (loop-antenna risk on this segment)
- **`Earth`**: 61% plane coverage over a 152.4mm run — the longest flagged gap on the board
- **`Driver5` Q4/Q5/Q6/Q7** (relay drain nets on one specific driver sub-sheet instance): 67-79% plane coverage over ~13mm each
- **`/Conn/SSR_SCK`**: 9 of 10 layer transitions on the shared shift-register clock have no ground stitching via within 1.0mm — this is the one finding I'd prioritize, since `SSR_SCK` fans out to all 8 cascaded ICs and a noisy/radiating clock return path affects the whole chain
- **`IO-001`**: J1 has no ferrite bead, CM choke, or ESD/TVS protection within 25mm. The analyzer's own risk framing ("common-mode current as low as 5µA can exceed FCC Class B") is worth taking seriously if this board will be sold/certified as a product rather than used as an internal test fixture
- **`SU-001`**: F.Cu and In1.Cu are adjacent signal layers with no plane between them in the 4-layer stackup (order is Sig-Sig-Power-Sig, not Sig-Power-Power-Sig) — inherent to the chosen stackup, a rework would require re-stacking, not just re-routing

**Cross-domain (`cross_analysis.py`, 3 findings, all warning, `PS-002` plane-split):** `+5V` (3 islands, 2 crossing signals), `GND` (37 islands, 25 crossing signals), `Earth` (2 islands, 1 crossing signal). 37 GND islands on a 64-channel, 245-net, 4-layer board is not unusual for a densely packed repetitive layout and is plausibly a byproduct of the many small isolated copper pours around each relay/connector rather than a genuine split-ground defect — but this was not visually confirmed against the layout image, so I'm reporting it as a flagged-but-unverified item rather than dismissing it outright.

## PCB Layout Analysis

### Footprint Placement
184 footprints, all on the front side (0 back-side), 180 SMD / 2 THT / 2 mounting holes. Complexity score 28/100 ("hand assembly feasible"), 4 unique footprints, dominant packages 0603 (88×) and SOP (72× — the relays).

Placement (`PM-002`) flags `J1`/`J2` at **−6.93mm** from the board edge and `X1` at 0.0mm. Negative distance means the connector footprint extends past the `Edge.Cuts` outline — expected and intentional for panel/edge-mounted D-Sub connectors that are meant to protrude through a chassis cutout, and `X1` sitting exactly at the edge is standard for a card-edge backplane connector. **Likely benign**, but worth a quick visual confirmation that the mechanical overhang matches the intended enclosure cutout.

`KO-001` flags 2 vias and 2 mounting holes (H1, H2) inside a keepout named `stitch_zone_0`. Without visual layout inspection I can't confirm whether this keepout was meant to exclude the mounting holes themselves (in which case this is a real rule violation worth fixing) or only meant to exclude *other* copper near the holes (in which case it's a self-referential false positive). Flagged as unverified — recommend a quick look at the keepout's actual rule-area definition in KiCad.

### Via Analysis
534 vias total, all 0.3mm drill per the load-net sample above (`Net-(J1-Pad6)` etc. show consistent 0.3mm through-vias). No via-in-pad or blind/buried vias detected — straightforward through-hole stitching, consistent with a 4-layer board.

### Thermal
`analyze_thermal.py`: **0 findings, thermal score 100/100.** Consistent with expectations — logic-level currents through TSSOP16 shift registers and SOP-4 relays at typical channel loads (well under GAQY212GS's 0.8A max and STPIC6C595's 100mA/output continuous rating) don't produce meaningful self-heating on this board.

### Manufacturing / DFM
- DFM tier: standard, 0 violations, min track 0.3mm / min drill 0.3mm / min annular ring 0.15mm — all within JLCPCB standard-tier capability.
- `FD-001`: no fiducials on F.Cu despite 180 SMD parts including 0.65mm-pitch TSSOP16 (medium-pitch). Recommend adding 2-3 fiducials for placement-accuracy margin on the 8 shift-register ICs.
- `TE-001`: 0/241 nets have test points. For a 64-channel board, even a handful of test points on the shared bus (`SSR_SCK`/`SSR_CS`/`SSR_MOSI`) and a couple of representative relay drain nets would meaningfully help bring-up/debug — SUGGESTION, not a blocker.
- Ordering notes: standard tier, 4-layer, 1.586mm, all-SMD-front assembly (single-side pick-and-place). Surface finish / mask color were not present in the analyzed stackup metadata — confirm with the fab order form.

## Component Lifecycle
Attempted via `lifecycle_audit.py --only lcsc`. All 6 queried MPNs returned `unknown` status. This environment's LCSC API integration is not reliable right now (see Review Limits/Not Performed below for the underlying cause) — **lifecycle status could not be determined for any part in this review.** Recommend re-running `lifecycle_audit.py` from an environment with working LCSC/DigiKey API access before treating any part as confirmed active/NRND/EOL.

## Not Performed / Review Limits

- **SPICE simulation**: not run — no ngspice/LTspice/Xyce installed in this environment. This board has no RC filters, dividers, or op-amp circuits that would materially benefit from it (the one "RC filter" the analyzer detected was a false positive, see above), so the impact is low.
- **Automated datasheet sync failed** for all distributors with API keys configured in this environment (none were configured — no `DIGIKEY_CLIENT_ID`/`MOUSER_SEARCH_API_KEY`/`ELEMENT14_API_KEY`), and the no-auth LCSC sync script (`sync_datasheets_lcsc.py`) failed due to an API response-schema mismatch in this environment (the `jlcsearch` search endpoint no longer returns a `datasheet`/`extra.datasheet.pdf` field the way the script expects). I worked around this manually — resolved LCSC part IDs via direct API calls, pulled `pdfUrl` from LCSC's product-detail endpoint, and downloaded PDFs via `curl` (Python's `urllib`/`ssl` module failed on this host's TLS renegotiation behavior against LCSC's CDN; `curl`/schannel handled it fine) — and obtained datasheets for the two functionally-critical ICs (GAQY212GS, STPIC6C595TTR) plus tried but could not obtain PDFs for the two passive parts (`1206B102K202`, `YC164-FR-07470R` — low verification value, standard passives) or find LCSC listings at all for `WLBD1608HCU101TL`/`CRCW060310K0FKTABC`. **Net effect: the two highest-risk components (both ICs, 72 of 184 board instances) are datasheet-verified; the passives are not, but are low-risk commodity values.**
- **Lifecycle audit**: ran but returned no usable status data for any part (see above) — treat as not performed.
- **`Frontend.kicad_sch` was not reviewed in depth.** At 678KB it is by far the largest sheet in the project (15× the size of the 8 driver sub-sheets combined) and clearly contains substantial circuitry beyond the 64-channel relay matrix this review focused on — likely the consumer of the +12V/-12V/+3.3V rails and the ~40 unnamed X1 signal pins that weren't traced. If a review of that circuitry is wanted, it should be scoped as its own pass rather than assumed covered here.
- **PCB layout was not visually rendered/inspected** in this pass (text/JSON analysis only) — the `KO-001` keepout-violation and `PS-002` GND-plane-split findings above would benefit from a visual check that this review couldn't perform.
- **Row/column matrix mapping was not exhaustively derived** for all 64 channels across J1/J2 — the bussing pattern was confirmed structurally (one full channel traced end-to-end, all 64 confirmed to share the same LED-drive topology) but the complete crosspoint address table was not built.

## Positive Findings

1. Both functionally-critical ICs (GAQY212GS ×64, STPIC6C595TTR ×8) are wired pin-for-pin correctly against their manufacturer datasheets — zero pinout errors found across the entire shift-register/relay drive chain.
2. LED drive current (≈7.7mA through the 470Ω network resistors) sits squarely in the GAQY212GS's recommended 5-30mA range, close to its typical 7mA design point.
3. All 8 shift-register VCC supplies are properly filtered (ferrite bead + 100nF) and the decoupling cap is measured at 1.45mm from the IC pins on the PCB — correct placement.
4. CLR̅ pull-up (10k) is a genuine series resistor to VCC, not a short — correct pull-up implementation.
5. Board is 100% routed with 0 DFM violations at JLCPCB standard tier.
6. HV-classed relay-load traces (0.55mm) comfortably exceed the current capacity needed for the 0.8A-rated relay channels.
7. Thermal analysis found zero hotspots — expected and confirmed for this current/package combination.
8. J1/J2 do have proper chassis-ground/shield connections (pins 0/14/15 → Earth_Protective) despite the analyzer's generic "no ground pin" flag.

## Analyzer Gaps

1. The EMC analyzer's decoupling check (`DC-001`/`DC-002`) has no concept of "IC without a power pin" and fired on all 64 photoMOS relays — a systematic false-positive pattern worth being aware of on any board using solid-state relays or similar 4-pin optoisolated devices.
2. The connector-ground-pin audit (`CG-AUD`) only recognizes literal `GND`-named nets, missing `Earth_Protective`/shield-style ground connections.
3. The RC-filter detector (`detect_rc_filters`) paired a pull-up resistor and an unrelated bypass capacitor that merely share a node, producing 8 phantom "159Hz filter" findings.
4. `GP-001` ground-plane-gap findings don't include component/location context beyond the net name — cross-referencing to a physical board location required manual net-length/via lookups.

## All Issues & Suggestions

| Severity | Issue | Detail |
|----------|-------|--------|
| WARNING | No hardware default-safe state for 64 relay outputs | `G̅` tied to GND (always enabled) on all 8 ICs, `CLR̅` has no capacitor (no incidental delay either) — confirm firmware clears the shift chain promptly after power-up, or add a hardware POR-clear network if a guaranteed safe startup matters |
| WARNING | J1 unfiltered, no ESD/CM protection within 25mm | `IO-001` — add ferrite/CM choke and ESD/TVS protection near J1 if this board interfaces with external cabling in the field |
| WARNING | `SSR_SCK` clock net missing ground stitching vias at 9/10 layer transitions | `RP-001` — add stitching vias near the through-vias on this net; affects signal integrity for all 8 cascaded shift registers |
| WARNING | Adjacent signal layers (F.Cu/In1.Cu) with no reference plane between them | `SU-001` — inherent to the 4-layer stackup choice; would require re-stackup to fully resolve |
| WARNING | Ground-plane gaps under `-12V`, `Earth`, and 4 nets on `Driver5` | `GP-001` (6 error-level) — review copper pour continuity under these specific routes |
| WARNING | Unconfirmed: `KO-001` keepout violations on H1/H2 mounting holes and 2 vias | Needs visual layout check to determine if this is a self-referential false positive or a real rule conflict |
| SUGGESTION | Relay symbol Datasheet/Description metadata is stale (references CPC1006N's ratings, not GAQY212GS's) | Update BOM/schematic properties for traceability; no functional impact |
| SUGGESTION | 2/8 unique parts missing MPNs; 2 more have MPNs but no matched LCSC listing | Populate/verify before fab ordering |
| SUGGESTION | No fiducials despite medium-pitch (TSSOP16) parts present | Add 2-3 fiducials for assembly placement accuracy |
| SUGGESTION | No test points on any of 241 nets | Consider adding test points on the shared SPI-like bus and a few representative relay drains for bring-up/debug |
| SUGGESTION | No per-channel snubber/clamp diode for inductive loads (only board-level Earth-bonding caps exist) | Confirm with system integrator whether downstream loads need on-card protection per the GAQY212GS datasheet's own recommendation |

---
*Datasheets consulted: GAQY212GS (GJ Semiconductor, Rev 1, Dec 2016), STPIC6C595 (STMicroelectronics, Rev 5, Mar 2009), CPC1006N (Littelfuse/IXYS, via Mouser/IXYS mirror, for symbol cross-check only). All three saved to `datasheets/` in the project directory alongside a `manifest.json` for the two independently-fetched relay/shift-register PDFs.*
