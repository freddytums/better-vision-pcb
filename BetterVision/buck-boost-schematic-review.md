# BetterVisionMainBoard — `buck_boost.kicad_sch` Schematic Review

**Reviewed:** 2026-08-22 · file mtime 2026-08-21
**Scope:** electrical review of the `buck_boost` sheet only (12 V input protection → 5 V conversion → power mux). No PCB layout, no BOM sourcing.
**Method:** netlist extracted programmatically from the `.kicad_sch` (pin-level connectivity from symbol geometry, wires, junctions and labels), then checked against the TI datasheets and the project's `cm5-vision-schematic-handoff.md`. All 149 component pins on the sheet resolve to a net.

---

## 0. What's actually on this sheet

The sheet has diverged from the handoff spec, and the name no longer matches the contents:

| Function | Handoff spec (§2) | Schematic as built | Verdict |
|---|---|---|---|
| 12 V eFuse | TPS26631PWPR | **U10 TPS26631PWPR** | matches |
| 5 V converter | TPS630701 buck-boost (16 V max) | **U5 LMR33640ADDAR** — synchronous *buck*, 3.8–36 V in, 4 A, 400 kHz | **changed** |
| Power mux | TPS2116 (5.5 V, 2.5 A) | **U1 TPS2121** (22 V, 4.5 A) | **changed** |
| Input TVS | SMBJ16CA / SMCJ16CA | **D2 SMCJ16CA** at IN_SYS | matches |
| Reverse-pol FETs | BSC012N06NS + 2N7002 | **Q1 BSC012N06NSATMA1 + Q2 2N7002** | matches |

Both swaps look like improvements, and the buck swap in particular **retires risk #5 in the handoff doc** ("TPS630701 16 V ceiling vs FRC transients — tight"). The LMR33640 is rated 36 V continuous / 42 V absolute, so the FRC rail is no longer anywhere near the converter's ceiling. Consequences are discussed in §2.

---

## 1. Must fix before fab

### 1.1 🔴 D3 (SHDN zener) is backwards — the board will not power on

**Net evidence:** `D3.1 → GND`, `D3.2 → SHDN`.

In the `JLCImport:BZT52C4V7_C173408` symbol, pin **1 is the cathode** and pin **2 is the anode**. I confirmed this from the symbol's own graphics rather than assuming: pin 1 sits at x = −5.08 next to the zener bar polyline (the bent bar at x = −1.27), and pin 2 sits at x = +5.08 behind the triangle, whose apex points left into the bar. In a SOD-123 BZT52 the banded (cathode) end is pad 1, so the footprint agrees with the symbol.

So as drawn, the **anode is on SHDN and the cathode is on GND** — the opposite of what the handoff doc calls for ("cathode→SHDN, anode→GND"), and the opposite of what a clamp needs.

**Consequence:** the zener is forward-biased from SHDN to ground. With ~110 µA coming down through R15 it clamps SHDN to roughly 0.5–0.6 V. The TPS2663 needs SHDN ≥ 2 V to enable and treats ≤ 0.8 V as a commanded shutdown, so **the eFuse never turns on and the board is dead** — no 12 V to the buck, no 5 V, nothing. It would look like a completely non-functional board with no obvious cause.

**Fix:** rotate D3 180° so pin 1 (cathode) goes to SHDN and pin 2 (anode) to GND.

**10-second verification:** in Eeschema, hover D3 and check the pin numbers, or open the footprint's 3D view and confirm the banded end lands on the SHDN side. If your symbol's pin 1 turns out to be the anode after all, then D3 is already correct — but then several other JLCImport diode symbols on the board would need the same check, since D4 (SS24) uses explicit A/K pin names and follows the pin-1-is-cathode convention.

### 1.2 🔴 SHDN divider browns out at 8.1 V — well above your 6.0 V UVLO

R15 = 100 kΩ (IN_SYS→SHDN), R16 = 33 kΩ (SHDN→GND) ⇒ V(SHDN) = 0.248 × V_batt.

| V_batt | V(SHDN) | TPS2663 state |
|---|---|---|
| 13.8 V | 3.42 V | on |
| 12.0 V | 2.98 V | on |
| **8.06 V** | **2.00 V** | **guaranteed-on floor** |
| 7.0 V | 1.74 V | indeterminate |
| 6.0 V | 1.49 V | indeterminate |
| 3.2 V | 0.80 V | guaranteed off |

The handoff doc noted "~1.74 V @ 7 V sag (above ~1 V enable thr — **confirm**)". Now confirmed, and it's not 1 V: the TPS2663 datasheet specifies **SHDN ≥ 2 V to enable, ≤ 0.8 V to shut down** (abs max 5 V — the doc's 5.5 V figure is also slightly off). Between 0.8 V and 2 V the behaviour is not specified.

**Consequence:** you carefully set UVLO to 6.0 V with the R8/R9/R10 divider, but SHDN overrides it and starts pulling the eFuse down at ~8 V. FRC battery sag into the 8–9 V range under hard drivetrain current is routine, so the vision coprocessor could hard-reboot during exactly the moments a match is most demanding. Because the CM5 is powered from this eFuse, that's a full boot cycle (~20–30 s), not a glitch.

**Fix — one resistor:** change **R16 to 68.1 kΩ**. Then V(SHDN) = 0.405 × V_batt: 2.00 V at 4.94 V input (comfortably below the 6.0 V UVLO, so UVLO becomes the real cutoff again), and 5.59 V at 13.8 V — which the (correctly oriented) 4.7 V zener clamps to ~4.3–4.5 V at the ~93 µA available, safely under the 5 V abs max.

**Fix — more robust alternative:** drop R16 entirely (or make it ≥ 1 MΩ) and let the zener set the level: R15 = 100 kΩ into a **BZT52C3V3**, cathode→SHDN, anode→GND. SHDN then sits at ~2.7–3.1 V for any battery voltage above ~3.6 V and never approaches abs max. This makes SHDN a clean "power is present" flag and leaves undervoltage entirely to the UVLO pin, which is the pin designed for it.

---

## 2. Design-level items worth a decision

### 2.1 🟠 Buck instead of buck-boost — where the rail actually gives up

Worth confirming the trade you've made, since the sheet name still says buck-boost:

- **Gained:** 36 V input rating (was 16 V). Retires handoff risk #5. Also 4 A capability vs the TPS630701's ~2 A at 5 V.
- **Lost:** no boost. Below roughly V_out + dropout the 5 V rail follows the input down instead of holding.

The dropout math is more forgiving than it sounds. LMR33640 dropout at 5 V/1 A is ~150 mV, HS RDS(on) is 95–160 mΩ, plus inductor DCR. At a 2 A CM5 load, expect the rail to hold to roughly **5.4–5.6 V input** (the part also folds switching frequency back to ~140 kHz in dropout to extend this). That is far below any credible FRC battery sag, so **the buck alone is fine** — provided §1.2 is fixed. Right now the SHDN divider cuts you off at 8 V, which is 2.5 V above where the converter would actually have quit.

**Follow-on:** the 15.0 V OVP setting existed to protect the 16 V-max TPS630701. That constraint is gone, and 15.0 V is close enough to regen/charging spikes on an FRC rail to cause nuisance disconnects (auto-retry, so each one is a CM5 reboot). Raising OVP to ~16.5 V keeps you below the SMCJ16CA's 17.8 V minimum breakdown so the TVS isn't asked to conduct before the eFuse acts:

> R8 = 442 kΩ, R9 = 69.8 kΩ, R10 = 40.2 kΩ → OVP = 16.5 V, UVLO = 6.02 V (unchanged).

### 2.2 🟠 5 V setpoint vs the CM5's 4.75 V floor

R4 = 100 kΩ / R5 = 24.9 kΩ ⇒ **V_out = 5.016 V** — exactly TI's reference divider for 5 V, good.

But regulation is sensed at the *buck* output, and the CM5 sits downstream of the mux. TPS2121 RON is 56 mΩ at 25 °C and **100 mΩ at 125 °C**. At a 2 A load that's 112–200 mV, plus PCB drop. The CM5 datasheet requires the 5 V rail to "rise monotonically to at least 4.75 V and remain above this level during operation," so at temperature you're working with only ~70 mV of margin.

**Suggested:** R5 → **24.3 kΩ** ⇒ V_out = 5.115 V, giving ~4.92 V at the CM5 at 2 A hot. Still 0.63 V below the OV1 trip (§2.3) and 0.89 V below the CM5's 6.0 V absolute maximum.

### 2.3 🟠 Mux OV1 trip band is tight against transient overshoot

With V_REF ≈ 1.06 V (1.01–1.10 V):

| | Divider | Nominal trip | Worst-case band |
|---|---|---|---|
| OV1 (buck rail) | 22.1 k / 5.11 k | 5.64 V | 5.38 – 5.86 V |
| OV2 (USB VBUS) | 22.6 k / 5.11 k | 5.75 V | 5.48 – 5.96 V |

The intent is right — CM5 VIN absolute max is 6.0 V, and both bands stay under it, which is exactly the protection you want. The concern is only at the **bottom** of the OV1 band: a 5.02 V rail with an 80 µF output bank can overshoot to ~5.3 V on a large load release, which is within ~80 mV of a 5.38 V worst-case trip. An OV trip drops IN1 and hands the CM5 to USB (or to nothing).

**Suggested:** make R26 **22.6 kΩ** to match R28 — OV1 becomes 5.75 V nominal, band 5.48–5.96 V, still under the 6.0 V limit but with 100 mV more room at the bottom. Combined with §2.2's higher setpoint the margins stay balanced. Then confirm overshoot on the bench with a 0→2 A load step; if it's marginal, a 100 pF feedforward cap across R4 will tighten it (TI only *requires* C_FF above 100 kΩ, so it's optional here).

---

## 3. Smaller items

| # | Item | Detail |
|---|---|---|
| 3.1 🟡 | **eFuse ILIM changed** | R12 = 4.02 kΩ ⇒ I_LIM = 18/4.02 = **4.48 A**. Handoff spec §6 says 4.99 kΩ ⇒ 3.6 A. Not a defect (device is 6 A rated, Q1 and the TVS are fine with it), but it's now ~2.5× the real maximum load (~1.8 A at 12 V for a fully loaded 4 A/5 V converter). Either restore 4.99 kΩ for a tighter fault response or update the spec doc. |
| 3.2 🟡 | **PGTH changed** | R1/R2 = 100 k/36.5 k ⇒ PGOOD threshold **4.49 V**; spec §6 says 31.6 k/10 k ⇒ 5.0 V. Both are below the sag floor so the "PGOOD stays high through a sag" intent still holds. Just reconcile the doc. |
| 3.3 🟡 | **L1 saturation unverified** | Ripple at 12 V→5 V, 400 kHz, 6.8 µH is 1.07 A p-p, so peak inductor current is 4.54 A at 4 A load. TI wants I_sat ≥ the HS current limit (**≈5.5 A**). `ZEMS0730-6R8M` is presumably 7.0×7.0×3.0 mm; I could not find a datasheet for it. Parts that size are often 4.5–5.5 A I_sat, which would be marginal. **Confirm I_sat ≥ 5.5 A and I_rms ≥ 4 A before ordering.** |
| 3.4 🟡 | **No local bypass on the eFuse IN pins** | The datasheet's 0.1 µF-minimum-at-IN_SYS requirement is met by C8 (1 µF/50 V). The IN node between Q1's drain and U10 pins 1–3 has no capacitor at all. A 0.1 µF there is cheap insurance against hot-plug ringing on the far side of the blocking FET. |
| 3.5 🟡 | **PG and ST go nowhere** | U5.PG is pulled to VCC through R19 (100 k) and terminates. U1.ST is pulled to +3.3 V through R21 (10 k) and terminates. You already spend a GPIO on the eFuse FLT (GPIO16, 10 k pull-up on the main sheet — verified present and correct). Routing these two to spare CM5 GPIOs would tell you *which* stage failed instead of just "something did," and would let software log "running on USB" vs "running on battery." |
| 3.6 🟡 | **IMON floats without a no-connect flag** | Matches the spec's intent (unused in v1), but ERC will flag it. Add a no-connect marker so a real problem doesn't hide in the warning list. |
| 3.7 🟡 | **`BetterVisionMainBoard.csv` is stale** | It still lists U1 = TPS630701RNMR, U7 = TPS2116DRLR, L1 = 1.5 µH, R12 = 4.99 k, R14 = 31.6 k — the pre-change design. Regenerate before it gets used for an order. |
| 3.8 🟢 | Sheet name | `buck_boost.kicad_sch` and the "DC-DC Buck Boost Converter" text label now describe a buck. Rename to avoid a future misunderstanding. |

---

## 4. Verified correct

These I checked and they're right — no action needed:

- **Reverse-polarity network matches TI §8.3.5 exactly.** Q1: source→IN_SYS, drain→IN, gate→B_GATE. Q2: gate→DRV, source→IN_SYS, drain→B_GATE. All four Q1 source pins and all four drain pins are tied. This is the single easiest thing to get wrong on this part and it's correct.
- **OVP = 15.003 V, UVLO = 5.995 V** from R8/R9/R10 = 402 k/60.4 k/40.2 k against the 1.2 V reference. Exactly on spec.
- **MODE → GND** (auto-retry, not latch-off) per spec.
- **TVS D2 at IN_SYS**, ahead of Q1, per both the datasheet's simplified schematic and the spec.
- **PGOOD → buck EN**, correctly gated: PGOOD is open-drain active-high; R13/R14 = 100 k/36.5 k puts EN at 3.21 V when released, against a 1.2–1.26 V threshold, and EN is always < VIN so the VIN+0.3 V limit can't be violated.
- **FLT# → GPIO16** with the 10 kΩ pull-up to +3.3 V on the main sheet, as specified.
- **Buck passives are TI's recommended set:** 100 nF BOOT (16 V, fine for a ~5 V BOOT-SW), 1 µF/16 V VCC, 6.8 µH for this operating point, input bank of 2×10 µF/50 V + 0.22 µF + 47 µF electrolytic — which is precisely the "≥10 µF ceramic + 220 nF close-in + 20–100 µF electrolytic for damping" guidance, damping resonance included.
- **Output capacitance ~88 µF nominal** across C4/C12/C13/C14 (+C18) — well inside the 10×-design-value ceiling, and inrush through the mux soft-start (100 nF ⇒ ~780 V/s) is only ~47 mA.
- **Mux current limit:** R20 = 29.8 kΩ ⇒ I_LM = 65.2/29.8^0.861 = **3.51 A**, matching the datasheet's own 29.8 kΩ → "3 to 4 A" table entry. Sensible against a 4 A converter and a ~2 A load.
- **Battery priority is correct.** PR1 = 0.360 × V_IN1 (9.09 k/5.11 k), CP2 = 0.327 × V_IN2 (10.5 k/5.11 k). At 5 V on both, PR1 = 1.81 V > CP2 = 1.64 V ⇒ IN1 selected. Priority mode is armed whenever V_IN1 > 2.95 V. Handover to USB occurs when the buck rail falls below 4.55 V — a last-resort threshold, which is the right character for this. (If you'd rather hand over before the CM5 leaves spec, R22 → 9.76 kΩ moves it to ~4.8 V.)
- **D4 (SS24) orientation is correct:** anode on the 5 V rail, cathode on the 12 V rail, so it only conducts if the output ever exceeds the input — the standard protection for a collapsing input. (Its A/K pin naming is also what let me cross-check the D3 symbol convention.)
- **Every IC pin is accounted for.** TPS26631 1–21, LMR33640 1–9, TPS2121 1–12 all map to nets consistent with their datasheet pinouts. No swapped or orphaned pins.
- **Capacitor voltage ratings** are appropriate throughout: 50 V parts on the 12 V rail (C1/C2/C8/C9/C16/C17), 25–35 V on the 5 V rails.

---

## 5. Suggested change list

| Ref | Now | Suggested | Why |
|---|---|---|---|
| **D3** | anode→SHDN | **rotate 180°** | §1.1 — board is dead as drawn |
| **R16** | 33 kΩ | **68.1 kΩ** | §1.2 — SHDN holds through sag; UVLO becomes the real cutoff |
| R5 | 24.9 kΩ | 24.3 kΩ | §2.2 — 5.115 V, restores margin over the CM5's 4.75 V floor |
| R26 | 22.1 kΩ | 22.6 kΩ | §2.3 — more room under transient overshoot, still <6.0 V |
| R8 / R9 | 402 k / 60.4 k | 442 k / 69.8 k | §2.1 — OVP 15.0 → 16.5 V, fewer nuisance trips, UVLO unchanged |
| R12 | 4.02 kΩ | 4.99 kΩ *(or update doc)* | §3.1 — restore the specified 3.6 A limit |
| L1 | ZEMS0730-6R8M | verify I_sat ≥ 5.5 A | §3.3 |
| — | — | add 0.1 µF, IN → GND | §3.4 |
| — | — | no-connect flag on IMON | §3.6 |

---

## 6. Bench checks once boards arrive

1. Confirm the eFuse enables at all (SHDN voltage at 12 V should read ~5.6 V clamped to ~4.4 V after the R16 change — if it reads 0.6 V, D3 is still backwards).
2. Sweep the input 13.8 V → 5 V on a bench supply and record where the 5 V rail leaves 4.75 V. Expect ~5.5 V with the fixes; anything above 6.5 V means something else is cutting out early.
3. 0 → 2 A load step at the mux output; scope the buck rail for overshoot against the OV1 trip point.
4. Reverse the input leads at 13.8 V (current-limited) and confirm no current flows and nothing heats.
5. Hot-plug a charged FRC battery through the real wiring harness and check that the dVdT ramp doesn't trip the 4.48 A limit.
6. Pull the battery with USB-C attached and confirm clean handover with no CM5 reboot.

---

*Companion documents: `cm5-vision-schematic-handoff.md` (design intent), `cm5-frc-vision-design-reference.html` (annotated reference).*
