# PCB Challenge 2026 – Submission

**Files**
- `pcb_challenge_final.kicad_pcb` / `.kicad_pro` / `.kicad_sch`: the KiCad 10 project. The schematic is unchanged from the challenge repo.
- `answer.md`: the logic deduction. The board is a 4-word × 4-bit asynchronous (dual-clock) FIFO.

## Board summary

| Item | Value |
|------|-------|
| Outline | Rectangle, **44 × 44 mm = 1936 mm²**. The Edge.Cuts line is 0.01 mm wide. |
| Content | 164 footprints, 411 vias (all 0.6/0.3 mm through vias) |
| Stack-up | 4 layers, 1.6 mm: F.Cu / 0.1 mm prepreg / In1.Cu / 1.24 mm core / In2.Cu / 0.1 mm prepreg / B.Cu |
| F.Cu | Components, signals, GND pour |
| In1.Cu | Solid GND plane. This is the reference for the USB D+/D− pair, which is routed on F.Cu with no vias, ≈28 mm long and length-matched. |
| In2.Cu | Signals plus a +3.3 V pour |
| B.Cu | Components, signals, GND pour |
| Rules (board setup + Default netclass) | 0.15 mm min track, 0.12 mm clearance, 0.6/0.3 mm vias (0.15 mm annular ring), 0.3 mm copper-to-edge, 0.25 mm hole-to-hole, 0.15 mm copper-to-hole |
| Power traces | +3.3 V / +5 V / GND are widened to 0.2–0.5 mm wherever clearance allows |
| Flip-flop pads (XSON6) | Only 0.15 mm tracks |

**Placement**
- Components are on both sides.
- The ESP32-S3 module sits in the top-right corner. Its antenna and antenna keep-out hang over the top board edge, which the rules allow.
- USB-C is on the bottom edge, below the module.

## Rule compliance

These were checked with KiCad's DRC and an independent script:
- **No unconnected items, no clearance, short or crossing errors.**
- **No vias inside any IC courtyard**, top or bottom. This includes the flip-flops' courtyard, which the library footprint places on the opposite side.
- **No foreign tracks inside an IC courtyard on the IC's own layer.** Each IC pin escapes straight out of its own pad.
- **The ESP32 module's ground pads** are a separate schematic net (`Net-(U25-GND-Pad2)`). They are joined by a small copper zone under the module, inside the pad ring.
- **Footprints are unmodified.**
  - SW2 and U25 were reloaded from the stock KiCad library, because the original PCB still had an old SW2 footprint.
  - Reference designators were moved to the Fab layers to keep the silkscreen clear. KiCad's library-mismatch check ignores this.

## Remaining DRC items (none come from the layout)
- `malformed_courtyard` on the XSON6 flip-flops (U1, U4…): this is the known flip-flop footprint issue mentioned in the challenge README.
- `lib_footprint_issues`: the custom `reg` / `adder` / `3v3_ldo` libraries are not installed on this machine.
- Schematic parity: symbol pins with no footprint pad. These are the NC pin 7 on the 6-pad muxes, J3's unused pins, and SW2 pins 3/4. They come from the provided schematic/footprints.
- Two `silk_edge_clearance` warnings. They are the ESP32's own silkscreen running past the board edge, where the antenna overhangs.
