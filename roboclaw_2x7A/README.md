# Basicmicro RoboClaw 2x7A Motor Controller

The Basicmicro RoboClaw 2x7A is the motor controller for the goBILDA 5203 Series Yellow Jacket planetary gear motors of the NEURODEAD-TESTBED rover.

> **Status: bench configuration done for units A, B, C (2026-10-12).** Values marked *bench* are provisional and **must be set again** with the real battery and the real motors (see section 5).
> `TBD` = not done yet. Sources: manual = User Manual Rev 6.0, firmware 4.3.0+; datasheet = Data Sheet v3.0.
> Tags: **Decided** (by Antoine), **Proposed** (not validated yet), **Documented** (read in vendor documents), **Measured** (read on the bench), **Calculated** (computed from documented values).

---

## 1. Role in the system

| Item | Value |
|---|---|
| Quantity | 3 units, 6 motor channels, **6 motors** — *Decided* |
| Controlled by | MCU1 (STM32F446RE Nucleo-64) over one shared UART (Packet Serial, Multi-Unit mode) |
| Closed-loop speed control (PID) | **Inside the RoboClaw** (gains tuned later with Motion Studio, saved in the unit). MCU1 never computes a PID. — *Decided* (D-001) |
| Wheel speed computation | Kinematics node on RK1 (ROS2). MCU1 receives one speed per wheel and converts it to QPPS (encoder counts per second). — *Decided* |
| Safety function | Hardware E-stop on pin S3 driven by MCU1, plus serial timeout (section 6) |
| Maximum rover speed | 1.5 m/s (limit of the JPL Open Source Rover design). Lower limits may be set by software. — *Decided* |

---

## 2. Hardware identification

| Item | Unit A | Unit B | Unit C |
|---|---|---|---|
| Physical label | A | B | C |
| Part number | IMC404 | IMC404 | IMC404 |
| Packet Serial address | 128 (0x80) | 129 (0x81) | 130 (0x82) |
| Address set and saved (2026-10-12) | Yes | Yes (reported) | Yes (reported) |
| Firmware version | TBD | TBD | TBD |
| Wheels driven (M1 / M2) | Front left / front right | Middle left / middle right | Rear left / rear right |

- Addresses and labels: *Decided*. Valid range 0x80 to 0x87, default 0x80 (manual p.64).
- Wheel mapping: *Proposed*, to confirm at wiring.

---

## 3. Electrical limits and reference data

### 3.1 Controller (documented)

| Parameter | Value | Source |
|---|---|---|
| Main supply | 6 to 34 VDC | datasheet p.15 |
| Logic supply (optional, separate) | 6 to 34 VDC | datasheet p.15 |
| Motor current per channel | 7.5 A continuous, 15 A peak | datasheet p.15 |
| Current derating | Peak reduced as temperature approaches 85 °C; limit ramps to 0 A at 100 °C | datasheet p.15 notes 1-2 |
| I/O inputs | 0 to 5 V (5 V tolerant). Low ≤ 0.8 V, High ≥ 2 V | datasheet p.15 |
| I/O outputs | 3.3 V maximum | datasheet p.2, p.15 |
| Serial | 8N1 TTL, up to 460 800 baud | datasheet p.15 |
| Encoder input | Quadrature, 32-bit counters, up to 9.8 M pulses/s | datasheet p.2, p.15 |
| 5 V user output (BEC) | Not used (MCU1 has its own supply) | datasheet p.3 |

**Documentation discrepancies (use the datasheet, the more conservative value):**
- Encoder rate: web page says 19.6 M pulses/s, datasheet says 9.8 M.
- BEC current: web page says 800 mA, datasheet says 1 A.

### 3.2 Motor and wheel (reference data)

| Item | Value | Source |
|---|---|---|
| Motor | goBILDA 5203-2402-0027 (label: 223 RPM @ 12 VDC, 26.9:1) | Documented (motor label, vendor page) |
| Exact ratio | 26.85:1 | Documented |
| No-load speed | 223 rpm @ 12 V | Documented |
| Stall current / torque | 9.2 A / 38 kg·cm @ 12 V | Documented |
| Stall current @ 14.8 V | ≈ 11.3 A (9.2 × 14.8 / 12, linear assumption) | Calculated |
| Encoder | 751.8 PPR at the output shaft (28 × 26.85). Counting convention (per channel or quadrature) **TBD**: test T9 | Documented, to verify |
| Motor connector | 3.5 mm bullet connectors; encoder on 4-pin JST XH. **Encoder pinout: read the goBILDA PDF before wiring. Wrong polarity can destroy the encoder.** | Documented |
| Wheel | Diameter 192 mm, circumference 0.603 m | Documented |
| Speed at 1.5 m/s | 149 rpm (1.5 / 0.603 × 60), 67 % of no-load speed at 12 V | Calculated |
| QPPS at 1.5 m/s | ≈ 1 870 (149.3 / 60 × 751.8), or ≈ 7 480 if 751.8 is counted per channel and quadrature multiplies by 4 | Calculated, depends on T9 |

### 3.3 Power sizing (calculated, worst cases)

| Case | Current |
|---|---|
| One motor stalled at 14.8 V | ≈ 11.3 A |
| 6 motors stalled, no current limit | ≈ 68 A |
| 6 channels at the 7.5 A continuous limit | 45 A |

These values size the wiring, the main fuse and the battery discharge rating. They must be checked before the battery and the wires are chosen. The final **Max current** setting needs a measured running current (rover mass, ground, slopes). — TBD

---

## 4. Wiring

### 4.1 Rules that damage the unit if ignored (documented)

1. **Never disconnect the negative (B−) wire while USB or any other ground is connected.** Disconnect the positive lead only. (manual p.10)
2. **Never reverse the main supply polarity.** (manual p.10)
3. USB does **not** power the motor stage; a main supply is required. (datasheet p.3)
4. Keep wheels off the ground during development. (manual p.12)
5. Keep motor and battery wires as short as possible (voltage spikes). (manual p.10, p.12)
6. **Never connect the +5 V pin of an I/O row to GND** (short circuit of the 5 V output). For tests, connect only the signal pin and GND.

### 4.2 Regenerative braking and the bench power supply

The RoboClaw sends energy back to the supply when a motor slows down. A battery absorbs it. **A bench power supply usually does not**, and the bus voltage can rise and damage the unit (manual p.11, p.34).

- Options from the manual: a battery in parallel with the supply, or a voltage clamp (VClamp) circuit on S4 or S5.
- Not a problem for tests **without motors**. **Must be solved before the first motor test on the bench supply.** — TBD
- Max regen is still at the factory value (−15 A). — TBD

### 4.3 UART (Packet Serial, 3 units on one port)

| RoboClaw pin | Function | Connects to |
|---|---|---|
| S1 | RX of the RoboClaw | MCU1 TX |
| S2 | TX of the RoboClaw | MCU1 RX |
| GND | Signal ground | Common ground with MCU1 |

- Signal header, from the board edge inward: I/O, +5 V, GND (datasheet p.6, manual p.67). **Check the silkscreen on the board before wiring.** Do not connect the +5 V pin if MCU1 has its own supply.
- Multi-Unit mode is enabled on every unit. S2 then becomes open-drain, so a pull-up resistor of 1 to 4.7 kΩ is required on the MCU1 RX line (manual p.68).
- The manual diagram ties that pull-up to 5 V. The RoboClaw outputs reach 3.3 V only, so the pull-up goes to **3.3 V**. — *Proposed*
- Do not use the UART connected to the ST-LINK virtual COM port of the Nucleo. — TBD: UART number

### 4.4 E-stop line (pin S3), shared by the 3 units

```
             3.3 V
               │
       [internal pull-up, very weak, x3 units in parallel]
               │
 S3 (A,B,C) ───┼───────────── MCU1 timer output (push-pull)
               │
         [R_pd external pull-down]
               │
              GND
```

| State of MCU1 | Level on S3 | Result |
|---|---|---|
| Running and armed, drives the pin HIGH | HIGH | Motors allowed |
| Fault detected by MCU1, drives the pin LOW | LOW | **E-stop** |
| Reset, watchdog reset, boot, no power (pin floating) | LOW (pull-down wins) | **E-stop** |

**Bench measurements, unit A** (lab supply 14.8 V, no motor, multimeter UNI-T UT131C, DC input impedance about 10 MΩ, to confirm in its manual; voltage between the pin and the GND pin of the same row):

| Condition | Voltage |
|---|---|
| S3, S4, S5 = Disabled | ≈ 0.15 V |
| S1, S2 (RC mode, before configuration) | 0 V |
| **S3 = E-Stop, nothing connected** | **2.05 V** (read as HIGH, no stop) |
| S3 wired to GND | E-stop active, 3 LEDs on, "Emergency Stop" in Motion Studio |

Interpretation (hypothesis, not proven):
- A configured pin seems to get an active pull-up; a Disabled pin does not.
- 2.05 V is below 3.3 V because the meter loads a **very weak** pull-up. Rough estimate: R_pu ≈ 10 MΩ × (3.3 / 2.05 − 1) ≈ 6 MΩ (assumes 3.3 V open circuit and 10 MΩ meter input).
- 2.05 V is close to the logic-high threshold (2 V). **The internal pull-up must not be trusted.**

Design rules:
- The external pull-down is **mandatory**: with non-latching mode, a floating S3 reads HIGH, which means "no stop". — *Decided* (D-005)
- Sizing: the low level must stay below 0.8 V.

  `V_low = 3.3 V × R_pd / (R_pd + R_pu/3) < 0.8 V`

  `V_low`: voltage on S3 when MCU1 does not drive it. `R_pd`: external pull-down. `R_pu`: internal pull-up of one unit (the factor 3 is for 3 units in parallel).
- Candidate: **R_pd = 1 kΩ**. Final value after measuring `R_pu` (test T10). MCU1 output current when HIGH: 3.3 V / 1 kΩ ≈ 3.3 mA, check against the STM32F446RE pin limits.
- Arming sequence on MCU1: S3 stays LOW until initialization and self-tests succeed, then goes HIGH. — *Proposed*

---

## 5. Configuration (Motion Studio, one unit at a time over USB)

Motion Studio runs on Windows only. Save the settings, then **power-cycle the unit** for them to take effect (manual p.21). Re-read the settings after the power cycle.

When a configuration is copied from one unit to another, **change the address first**: two units at the same address conflict on the bus.

| Setting | Value saved in unit A | Class | Status |
|---|---|---|---|
| Control mode | Packet Serial | Final | Measured |
| Multi-Unit mode | ON | Final | Measured |
| Packet address | 128 (B = 129, C = 130) | Final | Measured |
| Baud rate | 115 200 | Final (230 400 if needed) | Measured |
| Serial timeout | 0.1 s | Final (check against stop distance) | Measured |
| S3 mode | E-Stop (non-latching), invert OFF | Final (D-005) | Measured; release test T4c TBD |
| S4 / S5 | Disabled (one may become the voltage clamp) | Final until the clamp exists | Measured |
| Encoder 1 and 2 | Quadrature, invert OFF | Final (invert at wiring) | Measured |
| Autodetect max battery | OFF | Final | Measured |
| Max main battery | 16.1 V | **Bench** (supply 14.8 V) | See note 1 |
| Min main battery | 12.8 V | **Bench** | See note 2 |
| Max logic / min logic battery | 33.8 V / 5.4 V | Factory-like | Measured |
| Max current (M1, M2) | 3.0 A | **Provisional** (first tests only) | TBD |
| Max regen | −15.0 A | **Provisional** (factory value) | TBD |
| Idle mode / idle delay | Free wheeling / 1.0 s | **Open decision** (free wheeling or brake) | TBD |
| Default speed / accel / decel | 100 % / 2000 / 2000 | Factory | Not reviewed |
| Reverse motor, swap encoder channels | OFF | At wiring | TBD |
| PID gains and QPPS | Factory | At motor tuning | TBD |

Units B and C: same settings as A with their own address (reported by Antoine, not individually verified in this document). — TBD: read back and confirm.

Notes:
1. The field showed 16.4 V after the first save and 16.1 V later. Cause unknown (not entered by hand). — TBD: re-enter, save, power-cycle, read back.
2. Final limits depend on the battery. A full 4S LiPo is 16.8 V, so a **maximum of 16.8 V would trip the over-voltage protection** when the battery is full or when braking. Define the maximum above the full voltage plus a regeneration margin. Under load the battery voltage drops, so 12.8 V as a minimum can cut the motors too early.
3. Serial timeout range: 0 to 25.5 s, 0 = disabled (manual p.60, p.64).
4. The S3 mode can also be set from MCU1 with command 74 (`0xC1` = E-Stop latching, `0x41` = E-Stop non-latching) (manual p.83).
5. Keep a screenshot of each Motion Studio tab (General, PWM, Velocity, Position) for each unit in the repository, next to this file.

---

## 6. Safety design

| # | Mechanism | Covers | Status |
|---|---|---|---|
| 1 | Pin S3 (non-latching E-Stop) + external pull-down | MCU1 fault, reset, boot, power loss, cut wire | Decided, pull-down to install and test |
| 2 | STM32 independent watchdog (IWDG) | MCU1 frozen | Decided (MCU1 design) |
| 3 | RoboClaw serial timeout 0.1 s | UART task dead, cable cut | Set, to test (T7) |
| 4 | Main power contactor / kill switch | Operator, everything else | Recommended by vendor (manual p.10) |

**S3 is used only for serious faults** (kill switch, watchdog, MCU1 failure). Holes and obstacles are handled by a normal speed-zero command over the serial link, not by S3.

**Consequences for firmware:**
- MCU1 sends a valid command to every unit **at every cycle, even at zero speed**. If it stays silent, the RoboClaw stops the motors (mechanism 3).
- The latch is in MCU1 software: after a fault, S3 stays LOW until an explicit re-arm. After the re-arm, the old speed command must not resume (the 0.1 s timeout covers this, to verify with T4d).

### E-stop behaviour (documented and measured)

- An E-stop is a **fault**: motor outputs are disabled and the motors are **stopped by braking** (manual p.38-39).
- Indicators: all three LEDs lit (manual p.39), `ESP` flag in Motion Studio (manual p.20). The error bit `0x00000001` of command 90 is cleared when read (manual p.76).
- *Measured:* in **latching** mode the stop stays after the wire is removed and is cleared by a **power cycle**. The manual describes no serial command to clear it. This is why D-005 uses non-latching mode.

---

## 7. Communication protocol (Packet Serial)

Frame: `[address][command][data...][CRC16 high][CRC16 low]`

- CRC: CRC16-CCITT, polynomial `0x1021`, initial value `0`, computed over address + command + data. Multi-byte values are big-endian. (manual p.64-66)
- Write commands are answered with one byte `0xFF`. Read commands answer with data + CRC. (manual p.64)
- After a communication error, wait 10 ms before sending a new packet. A byte arriving more than 10 ms after the previous one makes the unit discard the packet. (manual p.64)
- At 38 400 baud or below, the unit adds at least 1 ms before replying (manual p.67). No processing time is documented above that: TBD, measure with an oscilloscope.

| Cmd | Use | Request | Answer |
|---|---|---|---|
| 21 | Read firmware version | address, 21 | text up to 48 bytes ending with LF + NUL, CRC |
| 37 | Set speed M1 and M2 (QPPS, signed) | address, 37, 4 bytes, 4 bytes, CRC | `0xFF` |
| 49 | Read motor currents (value / 100 = A) | address, 49 | 2 + 2 bytes, CRC |
| 24 | Read main battery voltage (0.1 V units) | address, 24 | 2 bytes, CRC |
| 90 | Read status and errors | address, 90 | status bits, CRC |
| 74 | Set S3/S4/S5 modes | address, 74, 5 mode bytes, CRC | `0xFF` |
| 94 | Write settings to flash (**causes a reset**; wears the flash, do not use in normal operation) | address, 94, key `0xE22EAB7A`, CRC | `0xFF` |

**Test vectors** (CRC computed and self-checked, a full frame including its CRC gives 0):

| Frame | Bytes |
|---|---|
| Read version, address 0x80 | `80 15 59 0C` |
| Read version, address 0x81 | `81 15 6A 3D` |
| Read currents, address 0x80 | `80 31 3D EA` |
| Speed +1000 / −1000 QPPS, address 0x80 | `80 25 00 00 03 E8 FF FF FC 18 82 90` |

### Bus time budget (3 units, command 37 + acknowledgement, then command 49 per unit)

| Baud | Cycle time | Maximum rate | Bus load at 100 Hz |
|---|---|---|---|
| 57 600 | 10.9 ms | 91 Hz | 109 % (not possible) |
| **115 200** | **5.5 ms** | **183 Hz** | **55 %** |
| 230 400 | 2.7 ms | 366 Hz | 27 % |
| 460 800 | 1.4 ms | 731 Hz | 14 % |

Assumptions: 10 bits per byte (8N1), frame sizes from the manual, RoboClaw processing time ignored (TBD: measure).

---

## 8. Bench validation checklist

| # | Test | Expected result | Result |
|---|---|---|---|
| T1 | Voltage on S3/S4/S5 disabled and configured | See section 4.4 | **Done, unit A, 2026-10-12** |
| T2 | Power the unit alone from the bench supply, no motor | Unit boots, no fault LED | **Pass, unit A** (supply 14.8 V, about 12 mA) |
| T3 | Wire S3 to GND | 3 LEDs lit, `ESP` flag | **Pass, unit A** |
| T4a | Latching mode: stop stays after wire removed | Stop persists | **Pass, unit A** |
| T4b | Latching mode: power cycle | Stop cleared | **Pass, unit A** |
| T4c | Non-latching mode: wire removed | Stop clears by itself | **Pass, unit A**|
| T4d | Motors when S3 goes HIGH again (wheels off the ground) | No resume of the old speed command | TBD (needs motors) |
| T5 | Power-cycle after saving the settings | Settings unchanged | **Pass, unit A**; B, C TBD |
| T6 | First packet `80 15 59 0C` from MCU1 | Firmware version string returned | TBD |
| T7 | Stop sending commands for longer than the timeout | Unit stops the motors | TBD |
| T8 | Reset MCU1 (and trigger the IWDG) | S3 goes LOW, unit stops | TBD |
| T9 | Turn the output shaft one turn by hand, read the encoder count | ≈ 752 (or ≈ 3 007 if counted per channel) | TBD |
| T10 | Measure `R_pu` with known resistors on S3, then choose `R_pd` | Value of `R_pu` | TBD (no resistors yet) |

---

## 9. Decision log

| ID | Date | Decision | Alternatives rejected | Reason |
|---|---|---|---|---|
| D-001 | 2026-10-11 | PID runs in the RoboClaw, kinematics on RK1 | Software PID on MCU1 | One core cannot hold mixing, PID, safety and sensors; avoids two PID loops in cascade |
| D-002 | 2026-10-11 | S3 E-stop in latching mode | Non-latching | **Superseded by D-005** |
| D-003 | 2026-10-11 | Start at 115 200 baud | 57 600 (too slow), 230 400 (not needed yet) | 55 % bus load at 100 Hz |
| D-004 | 2026-10-11 | Packet Serial, 3 units, Multi-Unit mode | One UART per unit | Fewer wires and pins; manual supports up to 8 units |
| D-005 | 2026-10-12 | **S3 = non-latching E-stop. MCU1 holds S3 LOW until an explicit re-arm (latch in software). S3 is for serious faults only.** | (A) Latching: no software reset, only a power cycle. (B) Relay on the unit supply: motors freewheel during the cut. (C) Command 94 (save + reset): flash wear, not proven to clear the stop | MCU1 can re-arm. Safe if MCU1 fails (pull-down). No dependence on an undocumented method. Bench test T4: a latching stop is cleared only by a power cycle |
| D-006 | TBD | Idle mode: free wheeling or brake | — | Open: on a slope free wheeling lets the rover roll |

---

## 10. Open questions

1. Internal pull-up value of S3 (T10), then choose `R_pd`. Buy a resistor kit.
2. T4c and T4d (non-latching release, behaviour of the motors).
3. Why did Max main battery change from 16.4 V to 16.1 V?
4. Battery choice: final voltage limits, discharge rating, wire and fuse sizing (section 3.3). Needs the estimated rover mass.
5. Regeneration protection before the first motor test; final Max regen.
6. Stop distance: `d = v·t + v²/(2a)`. Needs measured `t` (sensor cycle + processing + serial) and measured `a` (braking). At 1.5 m/s the margin of 10 to 30 cm for hole detection does not hold; lower speed limits near holes. — D3
7. Ultrasonic task on MCU1: input = mask of active sensors (default all), output = each measurement with its age. Reading only the sensors in the direction of motion can be added later, with the rule that a move needs fresh data from the sensors of that direction.
8. Encoder pinout (goBILDA PDF) and counts per turn (T9).
9. RoboClaw processing time (oscilloscope).
10. Idle mode (D-006).
11. MCU1 firmware: board target (Nucleo-F446RE or other), UART number for the RoboClaw bus.

Manual sections not read yet: first start and Motion Studio overview (p.14-20), PID/QPPS settings (p.25-33), wiring and VClamp (p.40-45), encoders and auto-tune (p.53-58).

---

## References

- Basicmicro, *RoboClaw Series User Manual*, Revision 6.0, firmware 4.3.0 and newer.
- Basicmicro, *RoboClaw 2x7A Dual Channel Motor Controller Data Sheet*, Version 3.0.
- Basicmicro product page, RoboClaw 2x7A (part IMC404).
- goBILDA, 5203 Series Yellow Jacket Planetary Gear Motor, SKU 5203-2402-0027 (product page and PDF).

---

## French lexicon (for reading, can be removed)

| English | Français |
|---|---|
| floating | flottante (reliée à rien) |
| pull-up / pull-down | résistance qui tire la ligne vers le haut / le bas |
| latching / non-latching | verrouillé / non verrouillé |
| fail-safe | sûr en cas de panne |
| arming / re-arm | armement / réarmement |
| watchdog | chien de garde |
| timeout | délai d'arrêt automatique |
| freewheel | roue libre |
| regen | énergie renvoyée par le moteur quand il freine |
| stall | rotor bloqué |
| QPPS | impulsions d'encodeur par seconde |
