# CM5 FRC Vision Carrier — Schematic Review Handoff

> **Purpose:** self-contained design spec to seed a fresh chat for a KiCad schematic review.
> Paste this whole file (or attach it) into a new conversation, then use the kickoff prompt at the bottom.
> Status: paper design, **not yet validated**. Items marked ⚠ are derived/assumed and must be checked against the relevant datasheet during review.

---

## 1. Project summary

Custom carrier board for a **Raspberry Pi Compute Module 5** (CM5, **with eMMC**, not CM5Lite; 4 GB target) used as an FRC multi-camera AprilTag/6-DOF vision coprocessor.

- **Camera:** OV9281 global-shutter mono, CSI-2 (MIPI0), 2-lane.
- **IMU:** TDK ICM-42688-P on SPI0.
- **Comms:** Gigabit Ethernet (primary UDP path to roboRIO; PTP/IEEE-1588 time sync). USB-C for power + debug/eMMC flashing.
- **Power in:** 12 V FRC battery → protection eFuse → buck-boost (5 V) → power mux (OR'd with USB-C 5 V) → CM5.
- **Sync model:** every event timestamped on the PTP Hardware Clock (PHC); reconcile in software. No inter-unit sync wires.

---

## 2. Key BOM

| Function | Part | LCSC | Notes |
|---|---|---|---|
| Compute module | Raspberry Pi CM5 (4 GB, eMMC) | — | **eMMC variant**, not Lite |
| Buck-boost 5 V | TPS630701RNMR | — | fixed 5 V; **16 V input max** ⚠ tight on FRC rail |
| 12 V protection eFuse | **TPS26631PWPR** | — | adjustable OVP **cutoff**; NOT -636 (fixed 39 V clamp) |
| Reverse-pol power FET (Q1) | BSC012N06NS *or* CSD18543Q3A | C6471337 | N-ch, 60 V, low Rds(on); carries load |
| Reverse-pol pulldown FET (Q2) | 2N7002 | C8545 | small-signal N-ch, gate pulldown for Q1 |
| Power mux (OR-ing) | TPS2116DRLR | C3235557 | 1.6–5.5 V, 2.5 A, priority |
| USB-C receptacle | DEALON USB-TYPE-C-018 | C2927038 | 16-pin USB 2.0 |
| USB ESD array | USBLC6-4SC6 | — | 4-line; use 2 ch + VBUS |
| Input TVS (D1) | SMBJ16CA *or* SMCJ16CA | C284001 (SMBJ) | bidirectional, 16 V standoff, surge clamp at eFuse input |
| FSTROBE level shifter | **SN74LVC1T45** (dual-supply translator) | — | 1.8→3.3 V up-shift, fixed dir A→B, SC70-6 |
| IMU | ICM-42688-P | C1850418 | SPI0 |
| Ethernet | 1:1 MagJack + ESD/TVS | — | built by owner, not reviewed here |
| Camera-sync connector | locking JST-GH/SH class | — | XVS (**NC — reserved**) / FSTROBE / GND |

---

## 3. Power chain

```
12V Batt+ ──[Fuse/breaker upstream]──> Q_rev (N-FET, reverse pol)
   └─> TPS26631 eFuse (IN/IN_SYS) ──OUT──> [input TVS] ──> TPS630701 VIN
TPS630701 ──5V──> TPS2116 VIN1 (priority)
USB-C VBUS ──────> TPS2116 VIN2
TPS2116 VOUT ────> CM5 5V pins (77/79/81/83/85/87)  + bulk cap 10–30µF
```

- Battery is **priority** (VIN1) so a weak USB host is never asked to source full load.
- eFuse protects the 16 V-max converter; PDP/PDH breaker is upstream overcurrent backstop.

---

## 4. CM5 config / strapping pins (CRITICAL — common silent-failure points)

| CM5 pin | Signal | Setting | Why |
|---|---|---|---|
| 78 | GPIO_VREF | **→ CM5_3.3V** | Must NOT float/GND; sets GPIO bank to 3.3 V |
| 99 | PMIC_EN | **float** | Internal 100k pull-up powers up; breaker kills 12 V |
| 101 | USB_OTG_ID | **float** | Keeps device mode for rpiboot; **do NOT ground** |
| 93 | nRPIBOOT | float (jumper-low to flash) | Only eMMC flash path (no SD slot) |
| 20 | EEPROM_nWP | **float** | Write-enabled for field firmware updates |
| 92 | PWR_BUT | momentary → GND | Bench on/off + force-shutdown |
| — | CM5 variant | **eMMC**, not Lite | SDIO/SD-card pins are Lite-only |

---

## 5. Net-by-net connection list

### 5.1 USB-C (power + debug/flash)
| Connector net | → | Destination |
|---|---|---|
| VBUS (all pads tied) | → | TPS2116 VIN2 (not direct to CM5) |
| GND (all + shell tabs) | → | GND plane |
| CC1 | → | CM5 pin 94 (no external 5.1k Rd) |
| CC2 | → | CM5 pin 96 (no external Rd) |
| DP1+DP2 (tied) = D+ | → | CM5 pin 105 (USB_P), 90 Ω pair |
| DN1+DN2 (tied) = D− | → | CM5 pin 103 (USB_N), no P/N swap |
| SBU1/2 | → | NC |

**USBLC6-4SC6 (SOT-23-6):** pin1(I/O1)=D+, pin3(I/O2)=D−, pin5=VBUS(5V), pin2=GND, **pins 4&6 NC**. Tap close to receptacle.

### 5.2 Ethernet (primary UDP / PTP)
| CM5 pin | Net | → |
|---|---|---|
| 3/5 | Eth Pair3 P/N | MagJack |
| 4/6 | Eth Pair1 P/N | MagJack |
| 11/9 | Eth Pair2 P/N | MagJack |
| 12/10 | Eth Pair0 P/N | MagJack |
| 15 / 17 | nLED3 / nLED2 | activity / speed |
| 18 | SYNC_OUT | **test point + DNP level-shifter** (validate-first) |

100 Ω diff pairs; MagJack + ESD already implemented by owner.

### 5.3 Camera — CSI-2 (MIPI0) + control
| CM5 pin | Net |
|---|---|
| 129/127 | MIPI0_C P/N |
| 117/115 | MIPI0_D0 P/N |
| 123/121 | MIPI0_D1 P/N (OV9281 = 2-lane; D0/D1 only) |
| 135/133, 141/139 | MIPI0_D2, D3 — route for future, unused now |
| 82 / 80 | SDA0 / SCL0 (camera I²C / SCCB) |
| 97 / 100 | CAM_GPIO0 / CAM_GPIO1 |

**Camera module vias → carrier (locking connector XVS/FSTROBE/GND):**
- **XVS** = external trigger IN — **NOT USED (v1).** Left disconnected; sensor free-runs. Pin retained on the 3-pin connector as a no-connect, reserved for a future externally-triggered iteration. No level shifter populated on this leg.
- **FSTROBE** = exposure strobe OUT (~1.8 V, level confirmed): (a) → LED-driver enable (illumination strobe); (b) up-shift 1.8→3.3 V via **SN74LVC1T45** → **GPIO17 (CM5 pin 50)**, PHC-timestamped via RP1 PIO (true exposure time). With XVS unused, FSTROBE is the **sole** per-frame timing authority.
- **SIOD/SIOC** = same I²C as 82/80; **GPO** = NC; **3v3** = module power; **GND** = shared reference (required).

### 5.4 IMU — ICM-42688-P (SPI0)
| CM5 pin | → | IMU pin | Function |
|---|---|---|---|
| 38 (GPIO11) | → | 13 (AP_SCLK) | SPI clock ≤24 MHz |
| 44 (GPIO10) | → | 14 (AP_SDI) | MOSI |
| 40 (GPIO9) | → | 1 (AP_SDO) | MISO |
| 39 (GPIO8) | → | 12 (AP_CS) | chip select |
| 41 (GPIO25) | → | 4 (INT1) | data-ready interrupt |
| 3.3 V | → | 8 (VDD), 5 (VDDIO) | both 3.3 V; decouple 0.1µF+2.2µF (VDD), 10nF (VDDIO) |
| GND | → | 6 (GND), 7 (RESV) | pin 7 RESV must be GND; 2/3/10/11 NC or GND |
| GND (via 0 Ω) | → | 9 (INT2/FSYNC/CLKIN) | FSYNC unused; 0 Ω keeps CLKIN option |

4-wire SPI; Linux `inv_icm42600` driver.

### 5.5 Frame timing (free-run + FSTROBE capture)
- **Camera free-runs (v1).** No external frame trigger into the sensor; XVS unused. The OV9281 sets its own frame cadence; per-frame exposure time is recovered from **FSTROBE** → up-shifted 1.8→3.3 V via SN74LVC1T45 → **GPIO17 (CM5 pin 50)**, captured by RP1 PIO and stamped against the PHC. FSTROBE is the sole frame-timing authority.
- **GPIO18 (pin 49) = unconnected (NC) in v1.** With XVS dropped there is no frame trigger to generate, so GPIO18 drives nothing; the earlier PEROUT/SYNC_OUT bench-validation test point is removed. (Multi-unit time sync is handled in-band by **PTP over Ethernet** — it does not use GPIO18.)
- **IMU** is **NOT** triggered and FSYNC is tied to GND (0 Ω) — rides the PHC timeline via FIFO timestamps anchored in software. The GPIO18→FSYNC mode-select idea is dropped for v1.

### 5.6 TPS2116 power mux
| Pin | Net |
|---|---|
| 3 (VIN1) | battery 5 V (priority) |
| 6 (VIN2) | USB-C VBUS |
| 2,7 (VOUT) | CM5 5 V rail + 10–30 µF bulk |
| 1 (GND) | GND |
| 5 (MODE) | → VIN1 (priority mode) |
| 4 (PR1) | 47 kΩ → VIN1 (internal 15 kΩ = ~4.13 V threshold); 1 MΩ PR1→ST for hysteresis |
| 8 (ST) | via 1 MΩ to PR1 (hysteresis); else open |

---

## 6. Component values (12 V protection — TPS26631)

| Item | Value | Result |
|---|---|---|
| OVP/UVLO divider R1 (IN_SYS→UVLO) | 402 kΩ | — |
| R2 (UVLO→OVP) | 60.4 kΩ | UVLO ≈ 6.0 V |
| R3 (OVP→GND) | 40.2 kΩ | OVP ≈ 15.0 V |
| ILIM resistor | 4.99 kΩ | I_LIM ≈ 3.6 A (law I≈18000/R_ILIM, datasheet-confirmed) |
| dVdT cap | ~10 nF (start) | inrush ≈ C_OUT × dV/dt < ILIM |
| Reverse FET Q1 (power) | BSC012N06NS / CSD18543Q3A | source→IN_SYS, drain→IN, gate→B_GATE |
| Reverse FET Q2 (signal) | 2N7002 (C8545) | across B_GATE/DRV/IN_SYS; gate→DRV; fast pulldown for Q1 |
| Input TVS (= TI's "D1") | SMBJ16CA (C284001) or SMCJ16CA | bidirectional, 16 V standoff, ~26 V clamp; before Q1; no standalone diode in the Q1/Q2 sub-circuit |
| MODE | → GND | auto-retry (self-recovers after transient trip; not latch-off) |
| SHDN divider R4 (IN_SYS→SHDN) | 100 kΩ | enable ~3.35 V @13.5 V, ~1.74 V @7 V sag |
| SHDN divider R5 (SHDN→GND) | 33 kΩ | **Zener BZT52C4V7 (C173408) in parallel with R5** (both across SHDN→GND); **cathode→SHDN, anode→GND**; clamps SHDN <5.5 V max on a transient. NOT in series, NOT both ends to GND. |
| PGTH divider R6 (OUT→PGTH) | 31.6 kΩ | threshold ≈ 5.0 V (V(PGTHR)=1.2 V) — below 7 V sag floor so PGOOD stays high through sag |
| PGTH divider R7 (PGTH→GND) | 10 kΩ | enables fast brownout-recovery + sets PGOOD threshold; don't tie PGTH to GND |
| PGOOD | → buck-boost EN (open-drain, w/ pull-up) | converter starts only on valid input |
| FLT | → CM5 GPIO16 (pin 29) w/ **10 kΩ** pull-up to 3.3 V | logged fault flag (UV/OV/overload/PLIM/thermal); slow flag, 10 kΩ relaxed sweet spot |
| IMON | **FLOAT (unused, v1)** | analog out; RP1 has no ADC — not GPIO-readable. No R_IMON, no bypass cap |

Refs: OVP, UVLO, and PGTH thresholds all **1.2 V (datasheet-confirmed)**; ILIM law 18000 A·Ω **datasheet-confirmed** (3 kΩ→6 A, 4.02 kΩ→4.5 A, 30 kΩ→0.6 A). MODE: open=latch, GND=auto-retry (confirmed).

---

## 7. Assumptions & verify-before-fab (the honest risk list)

1. **eFuse divider math** — OVP/UVLO/PGTH 1.2 V refs and the 18000 ILIM constant are now datasheet-confirmed (no longer derived). Still cross-check the final resistor picks during schematic capture.
2. **OV9281 external-trigger software** — *no longer on the critical path:* camera free-runs in v1 (XVS unused), so the patchy libcamera/Pi external-trigger support is sidestepped. Revisit only if a future revision re-enables XVS.
3. **FSTROBE logic level + via mapping** — I/O confirmed **1.8 V**; FSTROBE up-shifted to 3.3 V via SN74LVC1T45. Still meter the actual module's via pinout before layout. (XVS dropped, so its level no longer matters.)
4. **SYNC_OUT PEROUT at frame rate** — *moot:* camera free-runs, so no PHC-disciplined frame trigger is generated. GPIO18 left unconnected; this item is dropped for v1.
5. ⚠ **TPS630701 16 V ceiling vs FRC transients** — tight; eFuse mitigates but rail not characterized.
6. **Diff-pair routing** — MIPI 100 Ω, USB/PCIe 90 Ω, Ethernet 100 Ω; intra-pair length match per CM5 datasheet §4.2.2.
7. **Buck-boost + MagJack subsystems** — built by owner, NOT reviewed in this spec.
8. **Thermal** — BCM2712 at full AprilTag load (~900 mA+) needs cooling (Fan_PWM GPIO19 / Tacho, or heatsink). Fan is **optional** — with a heatsink, GPIO19 is unconnected/free. **Conflict:** GPIO16 is already the eFuse FLT flag, so if a fan is fitted the tacho must move to another free GPIO.
9. **Filesystem** — hard power-off at breaker; plan overlayfs / read-only root.

---

## 8. Kickoff prompt for the new (KiCad) chat

> I'm doing a schematic review of a custom Raspberry Pi CM5 carrier board (FRC vision coprocessor). The full design intent — BOM, net-by-net connections, config-pin settings, component values, and known open risks — is in the spec I'm pasting/attaching below. I'll share my KiCad schematic (screenshots or netlist).
>
> Please review my schematic **against this spec**, checking specifically: (1) every CM5 strapping/config pin matches Section 4; (2) the net connections match Section 5; (3) the TPS26631 protection values match Section 6 and are consistent with the datasheet equations; (4) flag anything in the Section 7 risk list that my schematic doesn't address; and (5) call out any power-sequencing, decoupling, or diff-pair issues you see. Be skeptical — this is an unvalidated paper design and I want errors caught.

---

*Companion file: the full annotated design reference (block diagram + expanded notes) is the `cm5-frc-vision-design-reference.html` doc.*
