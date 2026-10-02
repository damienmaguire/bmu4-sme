# BMW BMU4 H SME

Open-source reuse of the Panasonic SME / BMW BMU4 H as a contactor box and pack voltage / current monitor for EV conversions. Hardware and software. Not a full BMS replacement, and not talking to BMW modules yet.

Module markings: `8845283-05`, `BMU4 H`, `230911 12`, `SK`, `24.04.24`, `00314`.

Identified as Panasonic SME (Speicher Management Elektronik), BMW P/N 61278845283 / 8845283-05, Panasonic P/N 23091112. Master BMS on Gen5 ~400 V packs (i4, iX, iX1, iX3, i5, i7 and related). This work treats it as an SBox-class device: contactors, precharge, pack voltage and current, HVIL. Target behaviour is the ZombieVerter / OpenInverter SBox command set (PHEV `0x100` / `0x300` family), not a new protocol.

The donor unit is from a crashed car. The seller confirmed it. Both pyrotechnic safety switches in the module are blown.

<img width="4096" height="2304" alt="bmw_sme_box10" src="https://github.com/user-attachments/assets/335f0a59-81bc-4245-9d2d-aa4aac2adff1" />

## Scope

- Keep the pack contactors, the contactor loom, and the bottom HV sensor board.
- Replace the VIN- and crash-locked Panasonic logic board with an open master.
- No CSC / cell-module work until a pack exists. The cell TPL is a different path from the sensor-board link below.
- The i3 SBox is slaved to its SME and is not the same CAN as the PHEV SBox `0x100` / `0x300` set.

## Hardware

Logic board MCU is an Infineon `SAK-TC275TP-64F200W` (LQFP-176, HSM). Around it:

- Infineon TLE9263BQX system basis chip
- TLE925x CAN PHYs and an NXP TJA1145A (`T1145AF`) partial-networking transceiver
- NXP PCA21125 RTC (`PC21125`)
- Nuvoton KA84921UA stackable battery-monitor AFE, marked `AN84921UA` on the rear (SSOP-24, SPI / daisy, 2.5 Mbps)
- Bosch `48021` voltage AFE
- NXP MC33664ATL transformer physical layer (TPL) PHY, 2 Mbps, next to an ON Semi HC00AG (74HC00) glue gate
- 100BASE-T1 on `C1+` / `C1-`, plus `ETH_WUP`. PHY not named yet.

Contactors are not driven from the bottom PCB. The coils have their own loom into the logic board. The bottom board is an isolated HV sensor board. `CN22` / `CN12` is the 2×11, 1.25 mm pitch, 1.27 mm row-spacing SMT mezzanine between them. It carries the MC33664 `RDTX+` / `RDTX-` pair plus power, ground and a few miscellaneous lines. The pulse transformer is on the bottom board, confirmed. A replacement master is an STM32 plus an MC33664 only. No transformer on the new board.

A Samtec CLP-111 / Minitek127 footprint is close enough to copy the pads. A boxed CN12 face will not mate a CLP socket; use a harness or a harvested CN22.

`ETH_WUP` pulled to 12 V through 10k keeps the board from staying down after a reset. Useful while probing.

Stock cell boards are almost certainly MC33771 / MC33772-class TPL slaves on a different PHY. Not in scope. The bottom board is a different use of the same family, below.

## Pyros

Two pyrotechnic safety switches in this module, both fired. The monitor connectors are silkscreened `fuse PSS4` (top) and `fuse PSS1` (lower). PSS is the pyrotechnic safety switch. On an i4 the two inside the SME are PSS1 and PSS4. A third, PSS6, lives in the small external HV box and is not in this module.

Both monitor pairs read open, which is the fired state. The pairs land on the bottom board, so the crashed state is visible to the sensor board and is not only a TC275 flag. Looping a monitor pair with a resistor, HV studs left open, is the test for whether that loop is a continuity sense or the enable for the shunt channel. One pyro is the pack path, the other the DC charge path. Which connector is which is not marked yet.

## Bottom board

Two NXP `MC33772BTC0AE`, 48-pin LQFP-EP, 7 mm. Trace code on both is `CTZU2404A` (2024 week 04), ahead of the module date `24.04.24`. Same reel.

This orderable is the junction-box variant, not a cell monitor:

- TPL, not SPI, on the isolated side
- Zero cell channels, no balancing, no cell over/undervoltage
- Precision GPIO
- Current channel and coulomb counter fitted

The `B` silicon uses 40-bit BCC frames. The `TC0` suffix is TPL, current option, zero cells. Two parts match the two analog clusters on the wire (`3B` and `41`), one node per pyro well.

The pyro well is a divider pocket: milled creepage slots, a 215 kΩ string on `R106`, `R108`, `R109`, `R235`, `R236` (`2153`), and a small SO-8 `IC3`. Those strings are the pack, link and DC-CP dividers into the GPIO or stack-voltage pins. The busbar NTCs are the other GPIOs. The shunt is `ISENSE+` / `ISENSE-` on one of the two parts.

## Sensor-board SPI

Probed on the MC33664 test pads with a Saleae. Settings that decode: 16-bit words, MSB first, CPOL=0, CPHA=1, active-low enable, separate TX and RX analysers. `INTB` is not on a test pad and was not needed.

The MC33664 is only the PHY. The TC275 sends 16-bit SPI words into it, and the isolated side is the BCC link to the two `MC33772BTC0AE`. Slave replies have bit 15 set. A block read is reply-count in word 0 and `(block << 8 | tag)` in word 1. Tag nibble 1 is the live half. The other nibble is often a pegged or stale copy, except where noted below.

Polled blocks:

| Request | Block | Role |
|---|---|---|
| `0004` | `03` | Status / heartbeat, rolling token |
| `0001` | `1D` | Flag |
| `0003` | `24` | ID / config |
| `0005` | `2D` | Coulomb counter plus status |
| `0006` | `3B` | Analog cluster, one MC33772 node |
| `000A` | `41` | Analog cluster, the other node, the one that moves |

The analog front end is powered by a flyback off the HV pack. The TPL digital side stays alive from the 12 V logic supply. At 20 V on the studs nothing analog moves. The pack word starts to leave its floor once the pack rail is well up (seen departing on the way to 325 V, alive at 360 V).

Public register map and the TPL frame are in the MC33772B datasheet. A BCC driver for this family already exists: [outlandnish/nxp-bcc-mc3377xb-driver](https://github.com/outlandnish/nxp-bcc-mc3377xb-driver).

## What is identified

All of these are in `000A / 41`.

| Word | Tag nibble | Signal | Scale |
|---|---|---|---|
| 6 | 1 | Pack / battery studs | ~51 counts/V, floor ~32800 |
| 4 | 1 | CP1 and CP2, one shared sense | ~23 counts/V, floor ~49300 |
| 0 | 1 | DC-CP, CCS inlet | same scale as the link, floor ~49300 |
| 1 | 2 | Busbar / shunt temperature | NTC direction |
| 5 | 1 | Busbar / shunt temperature | NTC direction |
| 0 | 2 | Busbar / shunt temperature | NTC direction |
| 7 | 1 | Temperature, opposite sign | smaller drift, not named |

CP1 and CP2 landing on the same word fits one AC-inlet divider. DC-CP on its own fits the CCS voltage check before the DC contactors close. Word 0 nibble 1 (DC-CP) and word 0 nibble 2 (temperature) are different channels, not a copy.

The three temperature words fell about 1000 counts over 30 s with a heatgun on the underside busbars, while pack voltage was held and word 6 sat still. They were flat to within 20 counts on a battery-only run. Which bar is which is not split yet.

## Current

The shunt is on pack negative and 10 A did go through it. No analog word stepped. `2D` word 0 holds 26594 counts/s from the start of the 360 V log to the end, wraps twice, and does not move by 10 counts during the pulses. That is the coulomb counter at its idle rate.

On this part the current channel stays off until a configuration write enables it. In `0006 / 3B`, words 1, 2 and 3, tag nibble 2, sit on `0x8000` for the whole 360 V plus 10 A capture. `2D` word 3 sits on `0xFFFF`. Same poll, the pack divider and the busbar temperatures are live, so the link and the HV flyback are up. `0x8000` is the channel-off marker, not a midscale zero.

The crashed TC275 is a sufficient reason for that write never to be sent. The PSS monitor pairs also land on the bottom board, so an open pyro loop may be holding the channel off as well. Next test is the PSS loops closed with a resistor, then a BCC init that sets the current-channel enable. Pushing more current at the disabled channel will not move it.

## What is still open

Contactor and precharge drive mapping, HVIL, the CAN command set, which of the three falling temperature words is which bar, and which of PSS1 / PSS4 is the pack path. The current-channel enable itself is a known register on a known part, not an unidentified amplifier.

## Captures

Saleae Logic 2 exports in `logs/`. Odd-numbered files in a pair are the MOSI-only export. Even-numbered files are the full TX and RX export and are the ones the word map comes from. Columns are `Time [s]`, `Packet ID`, `MOSI`, `MISO`.

| File | Stimulus |
|---|---|
| `Decode2.csv` / `Decode2.txt` | Early short capture, framing only |
| `Decode3.csv`, `Decode4.csv` | Idle baselines |
| `Decode5_10Apulses.csv`, `Decode6_10Apulses.csv` | 10 A shunt pulses, no pack voltage. No analog step |
| `Decode7_20vbatt.csv`, `Decode8_20vbatt.csv` | 20 V on battery studs, down to 0 and back. AFE still asleep |
| `Decode9_325vbatt.csv`, `Decode10_325vbatt.csv` | Ramp 20 V to 325 V and back. Word 6 leaves its floor |
| `Decode11_360vbatt10Apulses.csv`, `Decode12_360vbatt10Apulses.csv` | Ramp to ~360 V, then 10 A shunt pulses. Word 6 tracks voltage, current channel stays at `0x8000` |
| `Decode13_360vbattCP2.csv`, `Decode14_360vbattCP2.csv` | Ramp to 360 V with CP2 tied to the battery studs. Word 4 tracks |
| `Decode15_360vbattCP1.csv`, `Decode16_360vbattCP1.csv` | Same with CP1 tied. Word 4 again, not a separate channel |
| `Decode17_360vbattDCCP.csv`, `Decode18_360vbattDCCP.csv` | Same with DC-CP tied. Word 0 nibble 1 tracks |
| `Decode19_360vbattHeatgun.csv`, `Decode20_360vbattHeatgun.csv` | Ramp to 360 V on battery only, heatgun on the underside busbars for the whole log |

## Licence

Hardware notes and captures here are released for reuse in EV conversions. No BMW firmware is included.
