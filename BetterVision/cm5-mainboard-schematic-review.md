# BetterVisionMainBoard — Main Sheet + IMU Schematic Review

**Reviewed:** 2026-08-22 · `BetterVisionMainBoard.kicad_sch` (mtime 2026-08-22) and `imu.kicad_sch`
**Scope:** CM5 strapping/config pins, camera, Ethernet, USB-C, IMU, sync connectors. Companion to `buck-boost-schematic-review.md`.
**Method:** pin-level netlist extracted from the `.kicad_sch` files, checked against `cm5-vision-schematic-handoff.md`, the CM5 datasheet (project file), and cross-referenced against two public CM5/CM4 carrier designs on BoardRepo — chiefly [piecol/cm5-minima-rev2](https://boardrepo.com/piecol/cm5-minima-rev2) (CERN-OHL-S, 144★), a shipping CM5 carrier that makes a good apples-to-apples comparison.

> **Correction to an earlier answer in this session:** I described U11 (SN74LVC1T45) as the camera *trigger* level-shifter. It isn't. DIR is tied to +1.8 V, so it translates **A → B**, i.e. camera **FSTROBE (1.8 V) up to GPIO17 (3.3 V)** — the exposure-timestamp path from handoff §5.3(b). The trigger path (§5.5) isn't in the schematic at all; see §3.1.

---

## 1. Will-not-work items

### 1.1 🔴 nRPIBOOT is shorted to GND through SW1 — the CM5 will never boot from eMMC

**Net evidence:** `SW1.3 → GND`, `SW1.4 → nRPIBOOT (CM5 pin 93)`. `SW1.1` and `SW1.2` are unconnected.

The `JLCImport:TS665ZJ` symbol draws pins **3 and 4 as a solid bar** — they are the same internal pole — with pins 1 and 2 as the actual switched contact (the moving-contact graphic and its two contact circles sit on the 1–2 row). Both of your connections land on the *same* pole.

**Consequence:** nRPIBOOT sits at 0 V permanently. The CM5 enters rpiboot (USB device-boot) mode on every power-up and never boots the eMMC. Since you have no SD slot, the board does nothing but enumerate as a USB device.

**Why nothing catches this:** KiCad does not model connectivity *through* a symbol, so the netlist shows GND and nRPIBOOT as separate nets, ERC passes, and the ratsnest looks fine. The short only exists in the physical switch. This is the kind of error that survives to first boards.

**Fix:** put the two nets on *opposite* poles — GND on pins 3 **and** 4, nRPIBOOT on pins 1 **and** 2 (wiring both pins of each pole is the robust habit). Verify against the TS665ZJ datasheet which pin pairs are common before committing.

**Worth reconsidering separately:** nRPIBOOT must be low *at power-up* to enter flash mode, so a momentary button is awkward — you'd have to hold it while power comes up. Your own spec §4 says "float (jumper-low to flash)". A 2-pin header with a shunt is the better fit, and it frees SW1 for PWR_BUT (see §2.1). cm5-minima uses a slide switch (SW101) for exactly this reason.

### 1.2 🔴 No pull-ups on the camera I²C bus — the OV9281 will not enumerate

**Net evidence:** `SCL0: FPC5.20, Module1.80` and `SDA0: FPC5.21, Module1.82` — two pins each, nothing else.

SDA0/SCL0 are open-drain. The CM5 datasheet §2.6.1 describes them as plain RP1 GPIO38/GPIO39 routed into the MIPI0 interface and does not promise on-module pull-ups. Without external ones the bus can never be released high, so no I²C transaction completes and libcamera won't find the sensor.

cm5-minima-rev2 puts **R501 and R502 from SDA/SCL to its 3.3 V rail**, and also fans the bus out to its display and camera connectors — the standard arrangement.

**Fix:** add 2× pull-ups from SDA0 and SCL0 to +3.3 V. **4.7 kΩ** is a safe default for a short two-device bus at 100–400 kHz; drop to 2.2 kΩ if you add more devices or long FPC runs.

### 1.3 🔴 U11's A input is floating

**Net evidence:** `FSTROBE: U11.3(A)` — the level shifter's 1.8 V-side input is on a net with **no other pin**. The `FSTROBE` label is dangling, and the connectors it should come from (CN1/CN2) are unwired (§3.1).

U11 is powered (VCCA from CM5 +1.8 V, VCCB from +3.3 V), so this is a live CMOS input floating at mid-rail: shoot-through current in the input stage, and it will chatter, driving spurious edges onto GPIO17. If your software timestamps FSTROBE edges, it will collect garbage rather than nothing — which is worse, because it looks like it's working.

**Fix:** finish the wiring to the camera-sync connector, or — if the sync path is deliberately deferred — add a pull-down resistor (10 kΩ to GND) on the A side so the input is defined on unpopulated boards. Keep the resistor even after wiring it: the camera connector is unplugged more often than you'd think.

### 1.4 🔴 The power LED is wired backwards

**Net evidence:** `LED1.1(K) → +3.3v`, `LED1.2(A) → R18 → LED_nPWR (CM5 pin 95)`.

I confirmed the symbol's polarity from its own graphics (cathode bar at the pin-1 end, triangle apex pointing into it), so pin 1 really is the cathode. With the cathode on +3.3 V and the anode driven *low* by the CM5, current can never flow — LED_nPWR would have to rise above 3.3 V + Vf.

Your Ethernet LEDs on the same sheet are wired correctly (RJ1.15/17 anodes to +3.3 V, cathodes through R3/R6 to the CM5's nLED pins), which is what makes this one stand out.

**Fix:** swap LED1 — anode to +3.3 V, cathode through R18 to LED_nPWR.

**Also verify:** the value field says `XL-2835RGBC-KS` (an **RGB** LED) but the symbol has two pins and the footprint is `LED-SMD_L3.5-W2.8-RD-A` (a 2-pad red part). Either the MPN or the footprint is wrong — you'd order a multi-die RGB LED and land it on a 2-pad footprint.

---

## 2. Significant

### 2.1 🟠 PWR_BUT is unconnected

`Module1.92 (PWR_BUT)` is on a net by itself. Handoff §4 calls for a momentary switch to GND for "bench on/off + force-shutdown". Your only switch went to nRPIBOOT instead. cm5-minima has both: SW501 on PWR_BUT, SW101 on nRPIBOOT.

Without it, the only way to power-cycle is at the breaker, and there's no way to trigger a clean software shutdown — which matters given handoff risk #9 (filesystem integrity on hard power-off). Add a second momentary switch, or move SW1 here and use a header for nRPIBOOT per §1.1.

### 2.2 🟠 No local decoupling anywhere on the CM5 rails

Three separate nets, all with zero capacitors on this sheet:

| Rail | What's on it | Missing |
|---|---|---|
| `+5v` | CM5 pins 77/79/81/83/85/87 — **and nothing else** | Handoff §3 explicitly calls for "bulk cap 10–30 µF" here. All your bulk (C20–C22, C24) is at the mux output on the buck_boost sheet, physically far away. |
| `+3.3v` | CM5 84/86 out, camera FPC pin 22, U11 VCCB, GPIO_VREF, 2 pull-ups, magjack LEDs | No bulk, no bypass. This rail powers the camera module. |
| `+1.8v` | CM5 88/90 out, U11 VCCA + DIR | No bypass on VCCA. |

For comparison, cm5-minima-rev2 puts **C701/C704/C705/C706 plus C101/C102/C501** on its +5 V net and **C601/C602** on +3V3.

**Suggested:** 2× 22 µF + 2× 100 nF distributed across the CM5 5 V pins; 10 µF + 100 nF on +3.3 V near the module, plus a 100 nF right at FPC pin 22; 1 µF + 100 nF on +1.8 V at U11. Note that GPIO_VREF (pin 78) is a *reference* input for the whole GPIO bank — it deserves its own 100 nF.

While you're there: cm5-minima has test points (TP101/TP102) on +5 V. Cheap, and you'll want them during the bench checks in the power review.

---

## 3. Unfinished vs. wrong

### 3.1 🟡 The entire camera-sync / trigger subsystem is unwired

- **CN1 and CN2** (`SM04B-SRSS-TB`, JST-SH 4-pin) — **all 12 pins unconnected**. I assume these are your camera-sync header and fan header.
- **`XVS`** — a label exists in the file but reaches no pin.
- **`FSTROBE`** — reaches only U11's input (§1.3).
- **`PWM`** — CM5 pin 19 plus R11 (10 kΩ pull-up to 3.3 V, correct for the open-collector Fan_PWM output per CM5 datasheet §2.11), but no fan connector.
- **`TACHO`** — CM5 pin 16 alone. Fine electrically (internal pull-up to CM5_3.3V), but again no connector.
- **The 1.8 V trigger buffer** (handoff §5.5: GPIO18 → 74LVC1G125 with VCC = 1.8 V → camera XVS) does not exist as a component. GPIO18 (pin 49) is unconnected.

None of this is *wrong* — it reads as work in progress — but it's the highest-risk item in your own handoff doc (risk #2: "OV9281 external-trigger software … biggest schedule risk"), so it's worth knowing that the hardware side is currently absent rather than merely untested. Also note the fan: handoff risk #8 says the BCM2712 at full AprilTag load needs cooling, and there's currently no way to attach a fan.

### 3.2 🟡 IMU deviations from spec (all minor)

Everything on `imu.kicad_sch` that matters is correct — see §4 — but two small departures from handoff §5.4:

- **Decoupling:** you have C21 = 10 nF, C22 = 1 µF, C23 = 2.2 µF, all in parallel across +3.3 V/GND. The spec (and the ICM-42688-P datasheet) asks for **0.1 µF** + 2.2 µF on VDD and 10 nF on VDDIO. The 1 µF is standing in for the 0.1 µF; a 0.1 µF has lower ESL and is the part that actually handles the high-frequency edges. Cheap to add.
- **Pin 9 (INT2/FSYNC/CLKIN)** is tied hard to GND. The spec called for a **0 Ω** here so the CLKIN option stays open. Costs nothing to put the 0 Ω back.

### 3.3 🟡 USBLC6 uses pins 1 & 6 instead of 1 & 3

`D1.1 → D−`, `D1.6 → D+`, `D1.2 → GND`, `D1.5 → VBUS`, pins 3 & 4 NC. Handoff §5.1 specifies pin 1 = D+, pin 3 = D−, pins 4 & 6 NC.

Pins 1 and 6 are diagonally opposite corners of the SOT-23-6, so the two legs of the pair take noticeably different internal paths — a small, avoidable skew and impedance asymmetry on a 90 Ω pair. VBUS and GND are correct, and D+/D− polarity into the CM5 is correct (D+ → pin 105, D− → pin 103). Free to fix: move D− from pin 1 to pin 3 and D+ from pin 6 to pin 1.

### 3.4 🟡 Magjack cable-side center taps float

`RJ1.11–14` (VC1–VC4) are each on their own unconnected net. The PHY-side taps are handled (RJ1.4/5 → C3 100 nF → GND). Floating cable-side taps are common and will work, but a Bob Smith termination — 75 Ω from each VC to a common node, then a 1 nF/2 kV cap to chassis ground — measurably helps emissions and cable-borne ESD. On a robot dragging an Ethernet cable through a motor-heavy environment, that's worth the four resistors.

### 3.5 🟡 Strapping pins that are correctly floating, but implicitly

`EEPROM_nWP` (20), `PMIC_ENABLE` (99), `USB_OTG_ID` (101), `SD_VDD_Override` (73), `SD_PWR_ON` (75), `VBAT` (76), `LED_nACT` (21), `SYNC_OUT` (18) and the PCIe/HDMI/USB3/SD pins are all unconnected — which **matches handoff §4** for the three that matter (78/99/101) and is right for the unused interfaces.

The catch is that "deliberately floating" and "forgot to wire it" look identical in a netlist, and ERC will bury real problems in a wall of unconnected-pin warnings. Two habits worth adopting:

- Put **no-connect flags** on every intentionally-unused pin, so ERC output becomes meaningful.
- For strap pins specifically, add **DNP resistor pads** to both rails. cm5-minima does this — R104 gives it the option of pulling EEPROM_nWP low, and R102/R103 let it select GPIO_VREF between 3.3 V and 1.8 V. It costs nothing at fab and saves a respin if a strap turns out to need the opposite state.

---

## 4. Verified correct

- **Ethernet is complete and correct.** All four pairs map exactly per handoff §5.2 — CM5 3/5 → Pair3, 4/6 → Pair1, 11/9 → Pair2, 12/10 → Pair0 — into RJ1's TD4/TD2/TD3/TD1 with consistent polarity, ESD arrays (U3/U4) tapped on each pair, PHY-side center taps bypassed by C3, and the LEDs wired the right way round (nLED3 pin 15 = activity, nLED2 pin 17 = speed, both through 470 Ω).
- **The 22-pin camera FPC is textbook.** GND on 1/4/7/10/13/16/19; D0 on 2/3; D1 on 5/6; CLK on 8/9; D2 on 11/12; D3 on 14/15; CAM_GPIO0 on 17 and CAM_GPIO1 on 18 (which is exactly what CM5 datasheet §2.9.3 recommends); SCL0 on 20, SDA0 on 21, 3V3 on 22. Lane polarity is consistent throughout. Nothing to change but the pull-ups.
- **IMU SPI matches handoff §5.4 pin for pin:** GPIO11→SCLK, GPIO10→SDI, GPIO9→SDO, GPIO8→CS, GPIO25→INT1, VDD and VDDIO both on 3.3 V, pin 7 RESV to GND, pins 2/3/10/11 to GND. Cross-sheet connection is by global label on both sides, so it genuinely connects.
- **USB-C:** CC1 → pin 94 and CC2 → pin 96 direct with no external Rd — correct for CM5, and confirmed by cm5-minima doing exactly the same. D+/D− polarity correct, both connector rows tied, VBUS routed to the mux rather than to the CM5, shell tabs grounded.
- **GPIO_VREF (78) → +3.3 V**, per handoff §4. This is the one strap that must not float and it's right.
- **FAN_PWM pull-up** (R11, 10 kΩ to 3.3 V) is the correct treatment for CM5's open-collector fan output.
- **Cross-sheet nets are all global labels** where they need to be (+5v, +3.3v, +1.8v, VBUS, GPIO8–11, GPIO16, GPIO17, GPIO25). No accidental local-label islands.

---

## 5. Change list

| Item | Now | Change | Why |
|---|---|---|---|
| **SW1** | GND on pin 3, nRPIBOOT on pin 4 | GND on 3+4, nRPIBOOT on 1+2 (or move to a header) | §1.1 — never boots |
| **SDA0 / SCL0** | no pull-ups | 2× 4.7 kΩ to +3.3 V | §1.2 — camera undetectable |
| **U11 pin 3 (A)** | floating | wire to sync connector, and add 10 kΩ to GND | §1.3 — floating CMOS input |
| **LED1** | K to +3.3 V | swap: A to +3.3 V, K through R18 | §1.4 — can't light |
| **LED1 MPN/footprint** | RGB part, 2-pad footprint | reconcile | §1.4 |
| **PWR_BUT (92)** | floating | momentary to GND | §2.1 |
| **+5 V at CM5** | nothing | 2× 22 µF + 2× 100 nF | §2.2 |
| **+3.3 V / +1.8 V** | nothing | 10 µF + 100 nF / 1 µF + 100 nF; 100 nF at GPIO_VREF and at FPC pin 22 | §2.2 |
| **IMU C22** | 1 µF | add 0.1 µF | §3.2 |
| **IMU pin 9** | direct to GND | 0 Ω to GND | §3.2 |
| **D1 (USBLC6)** | pins 1 & 6 | pins 1 & 3 | §3.3 |
| **RJ1 VC1–4** | floating | Bob Smith termination | §3.4 |
| **unused pins** | bare | no-connect flags; DNP pads on straps | §3.5 |

---

## 6. Still open from the handoff doc

- **Risk #2 (OV9281 external trigger)** — hardware for the trigger path isn't in the schematic yet (§3.1). Your doc says to prove this on an off-the-shelf module *before committing layout*; that's still the right call, and it's still the thing most likely to cost you a respin.
- **Risk #3 (XVS/FSTROBE levels)** — can't be checked against the schematic until §3.1 is wired. Meter the actual module first; U11's VCCA is committed to 1.8 V.
- **Risk #4 (SYNC_OUT)** — CM5 pin 18 is unconnected. The doc wanted a test point + DNP level-shifter; neither is present. A bare test point costs nothing and preserves the option.
- **Risk #8 (thermals)** — no fan connector on the board (§3.1).
- **Risk #6 (diff-pair routing)** — a PCB matter, not reviewed here. The netlist has no net classes assigned beyond default, so KiCad isn't currently enforcing 90/100 Ω impedance or length matching. Worth setting up before layout review.

---

*Companion documents: `buck-boost-schematic-review.md`, `cm5-vision-schematic-handoff.md`, `cm5-frc-vision-design-reference.html`.*
*Public reference consulted: [piecol/cm5-minima-rev2](https://boardrepo.com/piecol/cm5-minima-rev2) via BoardRepo.*
