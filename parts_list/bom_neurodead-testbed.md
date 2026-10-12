# NEURODEAD-TESTBED — Hardware Reference

> Reference list: exact component on the rover, so a datasheet/manual can be found without digging through invoices, and to give a readable view of the architecture (recruiter, GitHub README).
> Purchase tracking (price, supplier, order number, status) is handled separately in a spreadsheet.
> `[TBD]` = reference not finalized or unknown — to be filled in.
> Last updated: 2026-10-02.

---

## 1. Compute & real-time stack

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| Cluster board | 1 | Turing Pi 2.5 | |
| Main compute module | 1 | Turing RK1 (RK3588, 32GB RAM) | Node 1, always-on OBC, ROS2 orchestrator |
| GPU module | 1 | Jetson Orin Nano 8GB (SO-DIMM module) | On-demand GPU inference |
| Neuromorphic chip | 1 | BrainChip Akida AKD1000, M.2 B+M form factor | Rear M.2 slot on the RK1 |
| NVMe SSD (Jetson) | 1 | [TBD — capacity/model] | |
| MCU1 (motor control) | 1 | STM32F446RE Nucleo-64 | 1kHz motor PID, hardware e-stop, ultrasonic safety reflexes |
| MCU2 (sensor fusion) | 1 | STM32F446RE Nucleo-64 | 200Hz IMU EKF, CCSDS telemetry |

---

## 2. Actuators

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| Motor controller | 3 | RoboClaw 2×7A | One per drive axle (propulsion) |
| Gearmotor + encoder | 4 | goBILDA 5203 Yellow Jacket | Propulsion only |
| Steering servo | 4 | Dynamixel XL430-W250-T | Corner-wheel pivot (steering) |
| Dynamixel bus interface | 1 | U2D2 | Signal conversion |
| Bus power injector | 1 | U2D2 PHB | Stacks onto the U2D2 |

---

## 3. Sensors

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| LiDAR | 1 | Livox Mid-360S | Ethernet + power |
| Stereo camera | 1 | OAK-D Pro W (OV9782 sensor) | Global shutter |
| IMU | 2 | ICM-20948 (9-axis) | Center of gravity |
| Magnetometer | 2 | MMC5983MA | Forward position |
| Barometer | 2 | BMP390 (or BMP388 substitute) [TBD] | Mounted outside the chassis |
| MEMS microphone | 1 | INMP441 (I2S) | Acoustic wake trigger for Akida |
| Ultrasonic | 4–5 | HC-SR04 | 5V echo → level shifter required |
| DC wattmeter | 1 | PZEM-051 (Modbus RTU/RS485) | In-series on the main bus |

---

## 4. Power

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| LiPo battery | 2 | Spektrum Smart 4S 5000mAh | In parallel via EC5 Y-harness |
| Parallel Y-harness | 1 | EC5 Y-harness | |
| Main bus fuse | 1 | 30A | |
| Main switch | 1 | [TBD] | |
| 12V regulator | 3 | Pololu D36V50F12 | Turing Pi (via PicoPSU) / DYNAMIXEL bus / LiDAR+fans |
| 5V regulator | 2 | Pololu D24V22F5 | Ultrasonic modules, split 2+2 for redundancy |
| PicoPSU power supply | 1 | PicoPSU 24-pin, 12V regulated input | For Turing Pi |

---

## 5. Electrical distribution & wiring

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| High-current distribution hub | 1 | ≥30A, for raw bus | |
| Low-current distribution hub | 3 | for regulated rails | |
| Motor→RoboClaw cables | 6 | goBILDA bullet lead 3.5mm (MH-FC, 300mm) | |
| Encoder signal wire | — | 22–24AWG | 20AWG in stock is incompatible with SXH-001T-P0.6 terminals |
| Encoder connectors | — | JST-XH (SXH-001T-P0.6) sensor side / Dupont 2.54mm header RoboClaw side | Two different connector types on the same cable |

---

## 6. Fabrication

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| 3D printer | 1 | Bambu P1S + hardened steel nozzle | |
| Structural filament | — | PA-CF | Load-bearing parts |
| Cover filament | — | PETG-CF | Covers |
| Flexible filament | — | TPU | Flexible mounts |
| Active heated dry box | 1 | [TBD] | |
| Passive dry box | 1 | [TBD] | |

---

## 7. Tooling

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| JST-XH crimping tool | 1 | IWISS-type | Verify Dupont contact compatibility |
| Benchtop variable AC/DC power supply | 1 | [TBD] | |

---

## Out of rover scope (potential homelab reuse)

| Component | Qty | Reference / model | Notes |
|---|---|---|---|
| Used PC with GTX 1070 Ti | 1 | [TBD — exact model] | Candidate sim/training machine for HIL (Gazebo Harmonic) |

---

## Obsolete components (no longer in use)

| Component | Reason |
|---|---|
| JPL OSR Brain board | Replaced by the STM32 stack |
| JPL OSR Motor board | Replaced by the STM32 stack |
| BrainChip AKD1500 | Dropped in favor of the AKD1000 |
| SynSense (neuromorphic chip) | Toolchain maturity judged insufficient |
| Innatera (neuromorphic chip) | Toolchain maturity judged insufficient |

---

## Remaining `[TBD]` items to resolve

1. BMP390 vs BMP388 (barometer).
2. Jetson NVMe SSD — exact reference.
3. Microphone, main switch, high-current distribution hub, benchtop power supply.
4. Active/passive dry box — models.
