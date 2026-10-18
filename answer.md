# Asynchronous (Dual-Clock) FIFO – 4 words × 4 bits

## Answer

The board is a **4-entry × 4-bit asynchronous FIFO** (clock-domain-crossing buffer).

- **Writer:** an external 4-bit parallel source on J4/J5, clocked by the on-board oscillator (`clk`).
- **Reader:** the ESP32-S3, which supplies its own read clock and can pass the data on over USB.

It uses classic async-FIFO structure:

- 3-bit read and write pointers (2 address bits + 1 wrap bit)
- adder-based pointer incrementers
- 2-flop pointer synchronisers in each direction
- full and empty flag logic

## Blocks

### Power, USB and MCU
- USB-C (J3) supplies VBUS (+5 V), with 5.1 kΩ CC pull-downs and 1 MΩ to the shield.
- U9 (TC1262) regulates +5 V down to +3.3 V. There is a power LED on each rail.
- U25 is the ESP32-S3-MINI-1, with:
  - native USB on D+/D−
  - BOOT switch on IO0 (SW1)
  - EN/reset RC and switch (SW3)
  - 32.768 kHz crystal on IO15/IO16

### Write clock domain (`clk`)
- **Write clock:** Y2 (KC2520Z oscillator) generates `clk`. It also goes to J5.3, with R7 as a pull-down.
- **Write pointer `wc[3:1]`:**
  - Stored in flip-flops U20/U19/U17 (74AUP1G80).
  - Each bit has a 2:1 mux (U15/U14/U16) with S0 = `R_en`. When S0 = 0 the bit holds (`wc`); when S0 = 1 it loads `wc+1`.
  - The increment comes from U28 (CD74HC283): A = `wc`, B = 0001, CIN = 0.
- **Memory (U10):** the "NMUX1308BQ" in SO-16 is the stand-in for the 16 flip-flops plus muxes of a 4×4 register file.
  - Write data: `b1..b4` from J4
  - Write address: `wc1, wc2`
  - Write enable: `R_en`, on `clk`
  - Read address: `rc1, rc2`
  - Read data: `out_1..out_4`
- **Read-pointer synchroniser (2 flip-flops per bit):**
  - Path: `rw_i` → mux (U29/U31/U35) → FF (U4/U33/U37) → mux (U30/U32/U36) → FF (U5/U34/U38) → `reg1..reg3`.
  - Each mux has `rst` on S0 and a 10 kΩ resistor to GND on Y1, which gives a synchronous clear.
- **Full flag:**
  - Logic: `full = (reg1==wc1)·(reg2==wc2)·(reg3≠wc3)`, i.e. the pointers are equal except for the wrap bit.
  - Gates: XOR U18/U52/U3, inverters U54/U53, ANDs U51/U27.
  - The result is registered in U1 and output on J5.4 (`???`).
- **Write enable:** `R_en = en · ¬full` (U26 inverter, U23 AND), with `en` from J5.1.
- **Write-side reset:** `rst` (J5.2) drives NPN pull-downs, each with a 10 kΩ emitter resistor and a 10 kΩ base pull-down. These clear the write pointer (`wr1..3`), the data lines `b1..b4` and the full-flag input.

### Read clock domain (ESP32)
- **Read clock:** IO6.
- **Read pointer `rc[3:1]`:**
  - Flip-flops U7/U6/U8, muxes U12/U11/U13 (S0 = `r_en`).
  - Increment comes from U22 (CD74HC283, +1).
- **Write-pointer synchroniser (2 flip-flops per bit):**
  - Path: `wr_i` → U39–U41 → U45–U47 → U42–U44 → U48–U50 → `wir1..3`.
  - IO8 acts as a synchronous clear.
- **Empty flag:**
  - Logic: `empty = (rc == wir)`, all 3 bits equal.
  - Gates: XOR U60/U56/U55, inverters U61/U58/U57, ANDs U62/U59/U63.
  - U64 forces `empty` = 1 while IO8 (reset) is high. The flag is registered in U65 and read by the ESP32 on IO9.
- **Read enable:** `r_en = IO7 · ¬empty` (U24 inverter, U21 AND).
- **Output data register:**
  - Muxes U73/U70/U71/U72 (S0 = `r_en`): load `out_k`, otherwise hold.
  - Flip-flops U69/U68/U67/U66 drive IO10..IO13.
  - IO8 clears these through NPN pull-downs.

## ESP32 pin map

| Pin | Function |
|-----|----------|
| IO6 | Read clock |
| IO7 | Read request |
| IO8 | Read-domain reset |
| IO9 | Empty flag (input) |
| IO10–IO13 | FIFO data `D[1..4]` (input) |

## Note on the flip-flops

The 74AUP1G80 only has an inverted output (Q̄). The pointer "hold" paths (D = Q̄) and the synchronisers only give the FIFO behaviour above if each register bit is read as a plain D flip-flop, which is clearly the intent. This is the same convention used in the warm-up design.
