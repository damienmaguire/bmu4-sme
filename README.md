# BMW BMU4 H SME

Open-source reuse of the Panasonic SME / BMW BMU4 H as a contactor box and pack voltage / current monitor for EV conversions. Hardware and software. Not a full BMS replacement, and not talking to BMW modules yet.

Module markings: `8845283-05`, `BMU4 H`, `230911 12`, `SK`, `24.04.24`, `00314`.

Identified as Panasonic SME (Speicher Management Elektronik), BMW P/N 61278845283 / 8845283-05, Panasonic P/N 23091112. Master BMS on Gen5 ~400 V packs (i4, iX, iX1, iX3, i5, i7 and related). This work treats it as an SBox-class device: contactors, precharge, pack voltage and current, HVIL. Target behaviour is the ZombieVerter / OpenInverter SBox command set (PHEV `0x100` / `0x300` family), not a new protocol.

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

Contactors are not driven from the bottom PCB. The coils have their own loom into the logic board. The bottom board is an isolated HV sensor board. `CN22` / `CN12` is the 2×11, 1.25 mm pitch, 1.27 mm row-spacing SMT mezzanine between them. It carries the MC33664 TPL plus power, ground and a few miscellaneous lines. A Samtec CLP-111 / Minitek127 footprint is close enough to copy the pads. A boxed CN12 face will not mate a CLP socket; use a harness or a harvested CN22.

`ETH_WUP` pulled to 12 V through 10k keeps the board from staying down after a reset. Useful while probing.

Stock cell boards are almost certainly MC33771 / MC33772-class TPL slaves on a different PHY. Not in scope.

## Sensor-board SPI

Probed on the MC33664 test pads with a Saleae. Settings that decode: 16-bit words, MSB first, CPOL=0, CPHA=1, active-low enable, separate TX and RX analysers. `INTB` is not on a test pad and was not needed.

This is not the stock 40-bit BCC frame. The master sends two 16-bit words. Slave replies have bit 15 set. A block read is reply-count in word 0 and `(block << 8 | tag)` in word 1. Tag nibble 1 is the live half; the other nibble is often a pegged or stale copy, except where noted below.

Polled blocks:

| Request | Block | Role |
|---|---|---|
| `0004` | `03` | Status / heartbeat, rolling token |
| `0001` | `1D` | Flag |
| `0003` | `24` | ID / config |
| `0005` | `2D` | Free-running integrator plus status |
| `0006` | `3B` | Analog cluster, mostly bias so far |
| `000A` | `41` | Analog cluster, the one that moves |

The analog front end is powered by a flyback off the HV pack. The TPL digital side stays alive from the 12 V logic supply. At 20 V on the studs nothing analog moves. The pack word starts to leave its floor once the pack rail is well up (seen departing on the way to 325 V, alive at 360 V).

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

## What is not identified

Current. 10 A through the identified shunt, one direction, about 1 s on and 1 s off, did not step any analog word. `2D` word 0 is a free-running integrator at about 27468 counts/s and the slope did not change. Either the shunt amplifier is on a rail that 360 V has not woken, or 10 A for a second is below what that integrator shows.

Also still open: contactor and precharge drive mapping, HVIL, the CAN command set, and which of the three falling temperature words is which bar.

## Captures

Saleae Logic 2 exports in `logs/`. Odd-numbered files in a pair are the MOSI-only export. Even-numbered files are the full TX and RX export and are the ones the word map comes from. Columns are `Time [s]`, `Packet ID`, `MOSI`, `MISO`.

| File | Stimulus |
|---|---|
| `Decode2.csv` / `Decode2.txt` | Early short capture, framing only |
| `Decode3.csv`, `Decode4.csv` | Idle baselines |
| `Decode5_10Apulses.csv`, `Decode6_10Apulses.csv` | 10 A shunt pulses, no pack voltage. No analog step |
| `Decode7_20vbatt.csv`, `Decode8_20vbatt.csv` | 20 V on battery studs, down to 0 and back. AFE still asleep |
| `Decode9_325vbatt.csv`, `Decode10_325vbatt.csv` | Ramp 20 V to 325 V and back. Word 6 leaves its floor |
| `Decode11_360vbatt10Apulses.csv`, `Decode12_360vbatt10Apulses.csv` | Ramp to ~360 V, then 10 A shunt pulses. Word 6 tracks voltage, current still absent |
| `Decode13_360vbattCP2.csv`, `Decode14_360vbattCP2.csv` | Ramp to 360 V with CP2 tied to the battery studs. Word 4 tracks |
| `Decode15_360vbattCP1.csv`, `Decode16_360vbattCP1.csv` | Same with CP1 tied. Word 4 again, not a separate channel |
| `Decode17_360vbattDCCP.csv`, `Decode18_360vbattDCCP.csv` | Same with DC-CP tied. Word 0 nibble 1 tracks |
| `Decode19_360vbattHeatgun.csv`, `Decode20_360vbattHeatgun.csv` | Ramp to 360 V on battery only, heatgun on the underside busbars for the whole log |

## Licence

Hardware notes and captures here are released for reuse in EV conversions. No BMW firmware is included.
