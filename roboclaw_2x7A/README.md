# Basicmicro RoboClaw 2x7A Motor Controller

The Basicmicro RoboClaw 2x7A Motor Controller is a motor controller for the 5203 Series Yellow Jacket Planetary Gear Motor.

> **Status: DRAFT (pre-configuration, 2026-10-11).**
> `TBD` = to be filled during bench work. Every other value cites its source (manual = User Manual Rev 6.0, firmware 4.3.0+; datasheet = Data Sheet v3.0).
> Each item is tagged **Decided** (by Antoine), **Proposed** (not yet validated on the bench) or **Documented** (read in the vendor documents).

---

## 1. Role in the system

| Item | Value |
|---|---|
| Quantity | 3 units (6 motor channels) — **TBD: confirm the number of motors (project notes say 4 gearmotors, 3 units give 6 channels)** |
| Controlled by | MCU1 (STM32F446RE Nucleo-64) over one shared UART (Packet Serial, Multi-Unit mode) |
| Closed-loop speed control (PID) | **Inside the RoboClaw** (gains tuned with Motion Studio, saved in the unit). MCU1 never computes a PID. — *Decided* |
| Wheel speed computation | Kinematics node on RK1 (ROS2). MCU1 only receives one speed per wheel and converts it to QPPS (encoder counts per second). — *Decided* |
| Safety function | Hardware E-stop on pin S3, plus serial timeout (see section 6) |

---

## 2. Hardware identification

| Item | Unit A | Unit B | Unit C |
|---|---|---|---|
| Physical label | A | B | C |
| Part number | IMC404 | IMC404 | IMC404 |
| Packet Serial address | 0x80 | 0x81 | 0x82 |
| Firmware version (read after update) | TBD | TBD | TBD |
| Firmware update date | TBD | TBD | TBD |
| USB serial number (optional field in Motion Studio) | TBD | TBD | TBD |
| Wheels driven (M1 / M2) | TBD | TBD | TBD |

Addresses and labels: *Decided*. Valid address range 0x80 to 0x87, default 0x80 (manual p.64).

---

## 3. Electrical limits (documented)

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
| 5 V user output (BEC) | Not used in this project (MCU1 has its own supply) | datasheet p.3 |

**Documentation discrepancies found (use the datasheet, the more conservative value):**

- Encoder rate: product web page says 19.6 M pulses/s, datasheet says 9.8 M.
- BEC current: product web page says 800 mA, datasheet says 1 A.

---

## 4. Wiring

### 4.1 Rules that damage the unit if ignored (documented)

1. **Never disconnect the negative (B−) wire while USB or any other ground is connected.** Disconnect the positive lead only. (manual p.10)
2. **Never reverse the main supply polarity.** (manual p.10)
3. USB does **not** power the unit; a main supply is required. (datasheet p.3)
4. Keep wheels off the ground during development. (manual p.12)
5. Keep motor and battery wires as short as possible (voltage spikes). (manual p.10, p.12)

### 4.2 Regenerative braking and the bench power supply

The RoboClaw sends energy back to the supply when a motor slows down. A battery absorbs it. **A bench power supply usually does not**, and the bus voltage can rise and damage the unit (manual p.11, p.34).

- Options from the manual: a battery in parallel with the supply, or a voltage clamp (VClamp) circuit.
- S3 is used for the E-stop, so the voltage clamp function would use S4 or S5.
- Voltage limits when running from a supply: minimum 2 V below the supply voltage, maximum 2 V above it (manual p.34).
- Not a problem for tests **without motors**. **Must be solved before the first motor test on the bench supply.** — TBD

### 4.3 UART (Packet Serial, 3 units on one port)

| RoboClaw pin | Function | Connects to |
|---|---|---|
| S1 | RX of the RoboClaw | MCU1 TX |
| S2 | TX of the RoboClaw | MCU1 RX |
| GND | Signal ground | Common ground with MCU1 |

- Signal header, from the board edge inward: I/O, +5 V, GND. **Do not connect the +5 V pin if MCU1 has its own supply.** (datasheet p.6, manual p.67)
- Multi-Unit mode must be enabled on every unit. S2 then becomes open-drain, so a pull-up resistor of 1 to 4.7 kΩ is required on the MCU1 RX line (manual p.68).
- The manual diagram ties that pull-up to 5 V. The RoboClaw outputs only reach 3.3 V, so the pull-up goes to **3.3 V**. — *Proposed*
- Do not use the UART connected to the ST-LINK virtual COM port of the Nucleo. — TBD: UART number

### 4.4 E-stop line (pin S3), shared by the 3 units

```
             3.3 V
               │
       [internal pull-up, value NOT documented, x3 units in parallel]
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
| Reset, watchdog reset, boot, no power (pin floating) | LOW (pull-down wins) | **E-stop** |

- S3 in E-Stop mode is **active low** (default), and every input has an internal pull-up: a floating pin means "no stop". (datasheet p.6, manual p.35) — *Documented*
- The external pull-down is therefore mandatory for a fail-safe behaviour. — *Proposed*
- Sizing: the low level must stay below 0.8 V.

  `V_low = 3.3 V × R_pd / (R_pd + R_pu/3) < 0.8 V`, which gives `R_pd < 0.32 × (R_pu / 3)`

- Internal pull-up `R_pu`: **TBD (measure with a multimeter)**. `R_pd`: **TBD**.
- Upper limit of `R_pd` driven by the STM32 pin current (`3.3 V / R_pd`): TBD, check the STM32F446RE datasheet.
- Arming sequence on MCU1: S3 stays LOW until initialization and self-tests succeed, then goes HIGH. — *Proposed*

---

## 5. Configuration (Motion Studio, one unit at a time over USB)

Motion Studio runs on Windows only. Save the settings, then **power-cycle the unit** for them to take effect (manual p.21). Re-read the settings after the power cycle to confirm they were stored.

| Setting | Target | Measured (read back) | Verified after power cycle | Status |
|---|---|---|---|---|
| Control mode | Packet Serial | TBD | TBD | Decided |
| Multi-Unit mode | ON | TBD | TBD | Decided |
| Packet address | A = 0x80, B = 0x81, C = 0x82 | TBD | TBD | Decided |
| Baud rate | 115 200 | TBD | TBD | Decided (first validation; 230 400 if needed) |
| Serial timeout | 0.2 s | TBD | TBD | Proposed (check against stop distance) |
| S3 mode | E-Stop **(Latching)** | TBD | TBD | **Decided** |
| S3 invert | OFF (active low) | TBD | TBD | Proposed |
| S4 / S5 | Disabled (one may become voltage clamp) | TBD | TBD | Proposed |
| Main battery min / max | Bench supply −2 V / +2 V (LiPo values set later) | TBD | TBD | Proposed |
| Max motor current | Low value for first tests | TBD | TBD | TBD (needs motor stall current) |
| Encoder mode | Quadrature | TBD | TBD | Proposed |
| Encoder invert / swap channels | Determined at first motor test | TBD | TBD | TBD |

Notes:

- The manual is ambiguous about the factory state of S3: the table says "E-Stop (latching) in serial modes" (p.35), a screenshot shows "Disabled" (p.21). **Set S3 explicitly.**
- Serial timeout: range 0 to 25.5 s, 0 = disabled (manual p.60, p.64). Resolution 0.1 s is an assumption, TBD: verify in Motion Studio.
- The same S3 mode can be set from MCU1 with command 74 (code `0xC1` = E-Stop latching, `0x41` = E-Stop non latching) (manual p.83).
- PID gains and QPPS: tuned later, with motors under load. Not part of this first configuration.

---

## 6. Safety design

| # | Mechanism | Covers | Status |
|---|---|---|---|
| 1 | Pin S3 + external pull-down | MCU1 reset, boot, power loss | Proposed, to test |
| 2 | STM32 independent watchdog (IWDG) | MCU1 frozen | Decided (MCU1 design) |
| 3 | RoboClaw serial timeout | UART task dead, cable cut | Proposed, to test |
| 4 | Main power contactor / kill switch | Operator, emergency | Recommended by vendor (manual p.10) |

**Consequence for firmware:** MCU1 must send a valid command to every unit **at every cycle, even at zero speed**. If it stays silent, the RoboClaw stops the motors (mechanism 3).

### E-stop behaviour (documented)

- An E-stop is a **fault**: motor outputs are disabled and the motors are **stopped by braking** (manual p.38-39).
- **Latching mode** (decision D-002): the unit stays stopped until it is **reset** AND the E-stop signal is cleared (manual p.35, p.38).
- Indicators: all three LEDs lit (manual p.39), `ESP` flag in Motion Studio (manual p.20). The error bit `0x00000001` of command 90 is cleared when read (manual p.76), so read it once during the stop.
- **How to reset a latched E-stop is not described in the manual sections read.** Working hypothesis: full power cycle of the unit (the logic part is powered from the main supply unless a logic battery is connected). — TBD: validate on the bench.

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
| 94 | Write settings to flash (**causes a reset**) | address, 94, key `0xE22EAB7A`, CRC | `0xFF` |

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

| # | Test | Expected result | Result / date |
|---|---|---|---|
| T1 | Measure the floating S3 pin and the internal pull-up | Value of `R_pu` | TBD |
| T2 | Power the unit alone from the bench supply, no motor | Unit boots, no fault LED | TBD |
| T3 | Force S3 LOW by hand | All 3 LEDs lit, `ESP` flag | TBD |
| T4 | Find the reset method for the latched E-stop | Unit runs again | TBD |
| T5 | Power-cycle after saving the settings | Settings unchanged | TBD |
| T6 | First packet `80 15 59 0C` from MCU1 | Firmware version string returned | TBD |
| T7 | Stop sending commands for longer than the timeout | Unit signals the timeout condition | TBD |
| T8 | Reset MCU1 (and trigger the IWDG) | S3 goes LOW, unit stops | TBD |

---

## 9. Decision log

| ID | Date | Decision | Alternatives rejected | Reason |
|---|---|---|---|---|
| D-001 | 2026-10-11 | PID runs in the RoboClaw, kinematics on RK1 | Software PID on MCU1 | One core cannot hold mixing, PID, safety and sensors; avoids two PID loops in cascade |
| D-002 | 2026-10-11 | S3 E-stop in **latching** mode | Non-latching | Manual reset only after a stop; matches the safety principle |
| D-003 | 2026-10-11 | Start at 115 200 baud | 57 600 (too slow), 230 400 (not needed yet) | 55 % bus load at 100 Hz |
| D-004 | 2026-10-11 | Packet Serial, 3 units, Multi-Unit mode | One UART per unit | Fewer wires and pins; manual supports up to 8 units |

---

## 10. Open questions

1. Number of motors and wheel-to-channel mapping.
2. Value of the internal pull-up on S3 (measure), then choose `R_pd`.
3. Reset method for the latched E-stop.
4. RoboClaw processing time (oscilloscope).
5. Motor stall current vs 7.5 A continuous / 15 A peak: needs the exact goBILDA SKU. Sets the max current setting.
6. Regeneration protection for bench tests with motors.

Manual sections not read yet: first start and Motion Studio overview (p.14-20), PID/QPPS settings (p.25-33), wiring and VClamp (p.40-45), encoders and auto-tune (p.53-58).

---

## References

- Basicmicro, *RoboClaw Series User Manual*, Revision 6.0, firmware 4.3.0 and newer.
- Basicmicro, *RoboClaw 2x7A Dual Channel Motor Controller Data Sheet*, Version 3.0.
- Basicmicro product page, RoboClaw 2x7A (part IMC404).
