# de-link PCB — Schematic Review

**Source of truth:** `minRead_pcb.kicad_sch` (KiCad 9.0.6, edited 2026-09-09) and `minRead_pcb.pdf` (printed 2026-09-09).
**Method:** netlist regenerated from the live schematic with `kicad-cli sch export netlist`, then every net and every device pin traced by hand; ERC run with `--severity-all`; all active-device pinouts checked against manufacturer datasheets.

> Note: this file replaces an earlier review that described an *LM27313 / MCP73832 / AO3401 / 22 µH L2* design. Those parts are no longer on the board and that review was invalid.

**Severity key:** 🔴 fix before fab · 🟡 fix or consciously accept · 🟢 verify / polish

---

## Bottom line

Topologically the schematic is **correct**. I traced all 126 nets and all 169 parts and found **no wiring errors** — the DW01A/FS8205A protection, the back-to-back FET battery disconnect, the TPS2116 power mux, the TP4056 charger, the e-paper charge pump, the SDIO bus, native USB, and the resistor-ladder buttons are all wired the way their datasheets specify. Pinouts are correct on every IC I checked, including the two easy ones to get wrong (FS8205A and DS3231MZ).

What's left is a set of **value / margin / documentation defects**, several of which will produce wrong behaviour on the first board:

* the ~~USB-status ADC window is mis-scaled~~ ✅ **fixed** (`R17`=150 k) — and the `R38` change made the ambiguous second state physically unreachable
* the LED driver is a 24.5 V part still **documented** as 30 V, with a PWM note below its datasheet minimum ⬜ *2 of 3 note edits outstanding*
* ~~a CMOS inverter input floats at boot~~ ✅ **fixed** (`R75` 100 k)
* ~~the USB-priority threshold is set above the voltage USB_VBUS reaches under load~~ ✅ **fixed** (`R38`=240 k)
* ⬜ **the 3.3 V LDO reaches thermal shutdown around 300–400 mA, and the LED boost still runs through it** — the `Q7` deletion (§6.14) fixes this
* ⬜ **the ±15–22 V e-paper capacitors still have no specified voltage rating** — the `50V` annotation in the file is on `C9`, not on `C13`–`C17`/`C20`
* ⬜ **ERC still reports 1 error** — the `TPS923610` symbol's `GND` pin is typed `power_output`

Full status table in **section 5**; per-block detail in **section 6**.

---

## 1. Confirmed correct (so you don't re-check these)

Verified pin-by-pin against datasheets — these are **right**, including the ones that commonly get built wrong:

| Block | Verdict |
|---|---|
| **FS8205A pinout** (`1=S1, 2=D12, 3=S2, 4=G2, 5=D12, 6=G1`) | ✔ Matches Fortune FS8205 rev ≥1.7 / LCSC C32254. `S1→B−`, `S2→GND(P−)`, `G1→OD`, `G2→OC` is the correct DW01A convention. Drains 2/5 intentionally floating. |
| **DW01A CS resistor** | ✔ `CS → R16 1 k → GND (P−)`, **not** B−. This is the one everybody gets backwards. Datasheet §12.1/§11.5 confirm CS must sit at pack negative. |
| **DS3231MZ `VCC→GND`, `VBAT→3V3`** | ✔ Not a bug. This is the datasheet's *Figure 5 single-supply VBAT-only* configuration, which explicitly requires VCC **grounded** (not floating). See §5 for the firmware consequence. |
| **`J2` pin 4 unconnected** | ✔ Correct — pin 4 is NC on the panel; the symbol's `VGL` label is wrong. See §2.4. |
| **Back-to-back Q3/Q8 (AO3419)** | ✔ Correct bidirectional blocking pair; both channels conduct in either direction when on, and block on reverse battery. Q3 gate referenced to B− (cell), so it stays on when the protection FETs open — no oscillation loop. |
| **TP4056** | ✔ `TEMP→GND` correctly disables the NTC (datasheet-sanctioned). `CE→VCC` is active-high enable, and CE is rated to 10 V so a 5 V tie is in spec. `R6 = 12 k → 1200/12k = 100 mA` ✔ matches your annotation. EPAD→GND with thermal vias ✔. |
| **TPS2116 MODE→USB_VBUS** | ✔ Safe in all four quadrants. On unplug, PR1 falls below its threshold before MODE crosses V_IL, so you never land in the `MODE=low + PR1=high` **shutdown** state. |
| **ADC pin selection** | ✔ All five analog nets are on **ADC1** (`IO1/CH0, IO2/CH1, IO4/CH3, IO8/CH7, IO9/CH8`). ADC2 is unusable with Wi-Fi — you avoided it. |
| **Boot/reset straps** | ✔ `IO0`: 10 k pull-up + SW6 via 100 Ω to GND. `EN`: 10 k + 1 µF (10 ms) + SW11 via 100 Ω. Both correct. |
| **e-paper connector J2** | ✔ 22 of 24 pins verified correct: `BS→GND` (4-wire SPI), VDDIO/VCI→3V3, VSS→GND, and every panel rail decoupled. Pin 4 is correctly left NC. |
| **LED strip J3** | ✔ `C+`/`W+` both to LED_SW (common anode), `C−`/`W−` switched into the sense resistor. Correct for a bicolour strip. |
| **ESD coverage** | ✔ Genuinely thorough: USB D±/CC (U6), all 4 SD data (U1), SD CLK+CMD (U9), dev-header GPIO (U8), touch FFC (U7), plus TSD05C on VBUS/3V3/P+ and TVS on the LED lines. SD CLK/CMD *are* protected. |
| **USB-C sink** | ✔ CC1/CC2 5.1 k pull-downs, shield via 1 M ∥ 1 nF. Correct. |
| **74LVC1G04 pinout** | ✔ `1=NC, 2=A, 3=GND, 4=Y, 5=VCC`. |
| **Option jumpers** | ✔ All alternate-config resistors (R43/R45/R58/R66/R72/R74) are correctly flagged DNP, so no rail-to-rail shorts. R73(0 Ω→GND) populated / R74 DNP is the correct default. |

---

## 2. 🔴 Will produce wrong behaviour

### 2.1 ~~USB-status ADC ladder~~ — **RESOLVED** (`R17` = 150 k, `R38` = 240 k both fitted)

**Topology** (traced from the netlist):

```
3V3 ──R70 100k──┬── USB_STAT (U4.38 / IO2) ──C23 2.2n── GND
                ├──R17 150k── ST     (U2.8,  TPS2116, open drain)
                ├──R67  56k── CHRG   (U11.7, TP4056,  open drain)
                └──R71  22k── STDBY  (U11.6, TP4056,  open drain)
```

**Signal semantics — datasheet-verified:**

* **`CHRG`** (TP4056 pin 7): *"When the battery is being charged, the pin is pulled low by an internal switch, **otherwise pin is in high impedance state**."* → LOW = charging, Hi-Z = not charging.
* **`STDBY`** (TP4056 pin 6): *"When the battery Charge Termination, the pin is pulled low … otherwise pin is in high impedance state."* → LOW = charge complete, Hi-Z otherwise. **One exception**, see the no-battery row below.
* **`ST`** (TPS2116 §7.3.3): *"Pulled low when **VIN1 is not being used**… When the TPS2116 is powering the output using VIN2 **or both channels are disabled**, the ST pin will be pulled low. **During thermal shutdown, the ST pin will be pulled low regardless of the channel being used**."*
  → **ST reports which input the mux selected, not whether USB is physically attached.** In this design `MODE` is tied to `USB_VBUS`, so "both channels disabled" is unreachable; ST therefore means *"the mux is running from the battery"*, which is true whenever `USB_VBUS` falls below the `PR1` threshold (now 3.40 V typ, 3.13–3.67 V) — plus the thermal-shutdown case.

**Reachable states, as built:**

| Scenario | ST | CHRG | STDBY | `USB_STAT` |
|---|:--:|:--:|:--:|---:|
| USB in, charger not running (VBUS < ~4.0 V) | HiZ | HiZ | HiZ | **3.300 V** |
| Unplugged, running on battery | **LOW** | HiZ | HiZ | **1.980 V** |
| USB in, actively charging | HiZ | **LOW** | HiZ | **1.185 V** |
| USB in, charge complete | HiZ | HiZ | **LOW** | **0.595 V** |
| **No battery** + ≥10 µF on BAT, blink phase A | HiZ | HiZ | **LOW** | 0.595 V |
| **No battery** + ≥10 µF on BAT, blink phase B | HiZ | **LOW** | **LOW** | **0.450 V** |

Worst-case adjacent separations: **1320 / 795 / 590 / 145 mV**. Only the last (0.595 ↔ 0.450) is tight, and both members mean "USB present, not charging normally" — so a single firmware bucket for < 0.7 V is sufficient.

**The `R38` fix closed the ambiguous state.** `ST`+`CHRG` (0.956 V) required the mux to be on battery *while* the charger still ran — i.e. `USB_VBUS` simultaneously below the switchover threshold and above the TP4056's ~4.0 V operating minimum:

| | Switchover (worst case) | Charger minimum | Overlap window |
|---|---:|---:|---|
| Before (`R38` = 310 k) | 4.43 V | 4.00 V | **[4.00, 4.43] — reachable** |
| **Now (`R38` = 240 k)** | 3.67 V | 4.00 V | **empty — unreachable** ✔ |

So `ST`+`CHRG`, `ST`+`STDBY` and all-three are now all physically impossible, and the five remaining states are unambiguous.

**Two edge cases to handle in firmware:**

1. **A collapsing charger reads as "unplugged".** If `USB_VBUS` sags below ~3.4 V while still physically connected, the mux hands to the battery (ST LOW) and the charger stops (both Hi-Z) → 1.980 V, which your table labels "discharging / unplugged". The *diagnosis* is wrong but the *conclusion* is right — the system genuinely is running on battery. Benign, but don't use `USB_STAT` alone to decide whether a cable is attached.
2. **`STDBY` LOW does not always mean "charged".** With no battery fitted, the TP4056 holds `STDBY` low continuously while `CHRG` blinks at 1–4 s (datasheet LED table: *"Green LED bright, Red LED Coruscate T = 1-4 S"*). `C3` (10 µF) + `C26` (1 µF) sit on the BAT pin, so this board matches that row exactly. **Detect it by the transition**: a reading that alternates 0.595 ↔ 0.450 V on a 1–4 s period is "no battery", not "charged". A static 0.595 V is a real termination.

**⚠ Still worth measuring:** the 1.980 V unplugged state assumes the TP4056's status pins are true Hi-Z while the chip is unpowered. Its datasheet **does not rate those pins at all**. If they clamp through an ESD path to a 0 V VCC, state 1 collapses toward ~0.93 V. One bench check on the first board settles it.

### 2.2 ⬜ **PARTIALLY DONE** — TPS923610 is a 24.5 V part still documented as 30 V

> **Status:** the OVP line now reads `~24V` ✅. Still says **"LED_MONIT … references 0-30V output"** — the part cannot exceed 24.5 V (OVP 24.25–25.5 V), so the firmware scale factor must use the real divider ratio, not "30 V = full scale".

Datasheet (SNVSCN8A): **TPS923610 max V_OUT = 24.5 V recommended, OVP trips at 24.25 / 25 / 25.5 V.** The 30 V parts are the **TPS923611 / TPS923612**.

Your schematic notes say:

* *"LED_MONIT is analog signal that references 0-30V output on LED voltage"* — the divider (`R39 1M / R41 120 k`, ratio 0.1071) is scaled for 30 V → 3.21 V, but the rail can never exceed the ~25 V OVP, so the ADC only ever sees up to **~2.68 V**. Harmless, but firmware must use the real divider ratio, not "30 V = full scale".
* *"Internal OVP should cut out at ~26V"* — **wrong by ~1 V in the unsafe direction.** OVP is 25.5 V max, 24.25 V min. If your software guard is set at 26 V it will never fire; the part's own OVP will fire first (which is fine), but any logic that assumes "26 V is reachable" is wrong.

**Fix:** either (a) correct the notes/firmware to 24.5 V max / 25 V OVP and keep the 610, or (b) fit **TPS923611** if you actually need 30 V. Also re-check the LED string: with 15 V of forward voltage you have plenty of headroom either way, so (a) is probably right.

### 2.3 ⬜ **OUTSTANDING** — ADIM PWM note still below the datasheet minimum

> **Status:** the note still reads **"Recommended to run between 5kHz-25kHz"**. Datasheet §7.3.8 sets the recommended range at **10 kHz–200 kHz**; below 10 kHz you get large FB ripple and audible noise. Use **20–100 kHz**.

Your note: *"Recommended to run between 5kHz-25kHz via PWM_LED for brightness control."*

Datasheet §7.3.8: **recommended PWM frequency is 10 kHz – 200 kHz**; below 10 kHz you get large FB ripple and audible noise (ADIM is *PWM-controlled analog* dimming — it low-pass filters duty into `V_FB = Duty × 200 mV`, so a slow PWM shows up as ripple on the LED current, not as flicker-free chopping).

**Fix:** change the note (and the firmware default) to **20 kHz–100 kHz**. Do not run at 5 kHz.

`R37 = 13.3 Ω → I_LED = 200 mV / 13.3 Ω = 15.0 mA` ✔ exactly matches your annotation. FB regulation voltage confirmed at 200 mV.

### 2.4 ~~e-paper `VGL` (J2 pin 4) has no decoupling capacitor~~ — WITHDRAWN: the `J2` symbol is mislabelled

**This was a false positive caused by a bad symbol, and the circuit is correct.** Recorded here because the underlying symbol defect is real and should still be fixed.

The `FH34SRJ-24S-0.5SH_50_` symbol labels pin 4 `VGL`, which made an unconnected pin 4 look like a missing decoupling cap. Per the panel datasheets, **pin 4 is NC**; the negative gate rail is decoupled at `PREVGL` (pin 23, `C16` 4.7 µF), which is the cap I mistook for a separate rail. The as-drawn topology matches the reference design:

| J2 pin | Actual | Net | |
|---|---|---|---|
| 4 | **NC** | *unconnected* | ✔ correct — symbol says `VGL`, wrong |
| 5 | VGH | C17 4.7 µF | ✔ |
| 21 / 23 | PREVGH / PREVGL | C14 / C16 4.7 µF | ✔ |
| 20 / 22 | VSH / VSL | C13 / C15 4.7 µF | ✔ |
| 18 / 19 / 24 | VDD / VPP / VCOM | C18 / C19 / C20 1 µF | ✔ |

**Real action — fix the symbol, not the circuit:** correct pin 4's name to `NC` in the `FH34SRJ-24S-0.5SH_50_` symbol and audit the remaining 23 pin names against the panel datasheet while you're in there. A mislabelled connector symbol is a latent trap: it already generated one false finding in this review, and the next person (or the next reviewer) will hit the same thing. Note that ERC is independently flagging `lib_symbol_mismatch` on the related `FH34SRJ-6S` symbols (§4), so this symbol family needs a pass anyway.

### 2.5 ~~`COLOR_SEL` floats into a CMOS inverter at boot~~ — ✅ FIXED (`R75` 100 k pull-down)

`COLOR_SEL` connects only to `U4.35 (IO42)`, `Q5.G` (BSS138), and `U12.2` — the **A input of the 74LVC1G04**. There is no pull resistor anywhere on the net.

**The float is confirmed, not assumed.** ESP32-S3 datasheet v2.2 Table 2-1: GPIO42 is **MTMS** (pin 48), and its Pin Settings column shows **`IE` only — no `WPU`, no `WPD`**. So GPIO42 leaves reset as an input with the buffer enabled and **no internal pull resistor at all**. (GPIO40/41 are the same; GPIO39/MTCK is the exception, weak-pulled-up unless `EFUSE_DIS_PAD_JTAG` is burnt.) GPIO42 is not a strapping pin and has no documented power-up glitch. So the node is genuinely undriven from power-on until firmware configures it — and again on every deep-sleep entry unless you latch it with RTC hold.

**What shoot-through is.** A CMOS inverter is a PMOS and an NMOS in series between VCC and GND, gates tied together. At a valid logic level exactly one is on and the static current is leakage (`ICC` ≤ **10 µA**). But between `VIL` = **0.8 V** and `VIH` = **2.0 V** (at 3.3 V) *both* devices conduct simultaneously, creating a direct VCC→GND path — the "crowbar" or shoot-through current. A floating input drifts into exactly that band and parks there, so the crowbar becomes **continuous** rather than a nanosecond transient during a normal edge.

**How much current?** TI does not publish a mid-rail number. The only datasheet anchor is **ΔICC ≤ 500 µA**, specified at `VI = VCC − 0.6 V` (2.7 V at 3.3 V VCC) — i.e. at a level *near the rail*, where crowbar is already past its peak. True mid-rail is worse; for LVC at 3.3 V it is commonly in the **0.5–2 mA** range, but that is empirical, not a spec. TI's own guidance is unambiguous — Recommended Operating Conditions footnote: *"**All unused inputs of the device must be held at VCC or GND** to ensure proper device operation"*, and §8.5.1: *"**inputs should never float** … the undefined voltages at the outside connections result in undefined operational states."*

**Why it matters here — it dwarfs everything else.** Your standby budget:

| Contributor | Current |
|---|---|
| AP2112K quiescent | 55 µA |
| `USB_STAT` ladder (ST asserted, R17=150k) | 13 µA |
| `R57`/`R56` gate divider (**now 1M/10k**) | 4 µA |
| `BAT_MONIT` divider (2 M) | 2 µA |
| DW01A | 3 µA |
| ESP32-S3 deep sleep | 10–20 µA |
| **Total** | **≈ 90–100 µA** |
| **Floating inverter input** | **500–2000 µA** |

A floating input is **5–20× the entire rest of the board**, turning a months-long shelf life into days. Secondary effects: the output can oscillate (an inverter biased in its linear region is a high-gain amplifier, and this node runs next to a switching converter), and `Q5`/`Q6` can be partly on together — though that one is benign here because `R40` holds `Q7` off so `U10` is unpowered at boot.

**Fix: a 100 kΩ pull-down from `COLOR_SEL` to GND.** Because GPIO42 has *no* internal pull, there is nothing to fight — the pull-down wins uncontested and defines a clean 0 V. Cost is 33 µA, and **only while firmware actively drives the pin high**; in sleep, if you leave `COLOR_SEL` low, it draws nothing. (A pull-*up* would work electrically too, but pull-down is better: it costs zero in the idle state, and it agrees with the GPIO's own reset level.)

### 2.5.1 Does a pull resistor slow the inversion? — No.

Short answer: **the pull resistor has no effect on switching speed, because it never does the switching.** Two separate regimes:

**When the GPIO is driving (all normal operation).** The ESP32-S3 sources 20 mA (default drive strength) with an output impedance of tens of ohms. The load is `Ci` = **3.5 pF** (LVC input) + `Q5` gate charge (BSS138 `Ciss` ≈ 50 pF) + trace ≈ **65 pF**. A 100 kΩ resistor in parallel with a ~30 Ω driver is a **0.03 % perturbation** — utterly invisible. Slew rate is `I/C` ≈ 20 mA / 65 pF ≈ **3.2 ns/V**, and the inverter's own `tpd` is **≤3.3 ns** at 3.3 V. Neither number moves when you add the resistor.

**When the GPIO is high-Z (boot, reset, sleep).** Here the resistor is the *only* thing driving the node, and `τ = 100 kΩ × 65 pF ≈ 6.5 µs`. Starting from a mid-rail float, it takes roughly **5 µs** to cross down through the 2.0 V→0.8 V forbidden band. So there *is* a slow transition — but it happens **once, at power-up**, and replaces an **indefinite** float. At ≤500 µA that's about 2.5 nC of charge. Completely negligible.

**One genuine caveat worth knowing.** The SN74LVC1G04 specifies a maximum **input slew rate of 10 ns/V** at 3.3 V ±0.3 V. Driven by the GPIO at ~3.2 ns/V you are comfortably inside it. But this is exactly why you must not try to make the *pull resistor* perform transitions (e.g. by RC-filtering the signal): at 6.5 µs it is ~5000 ns/V, roughly 500× over the limit, which is precisely the condition that provokes oscillation and excess supply current. Keep the resistor as a **DC bias only** and let the GPIO drive the edges.

**Sizing.** 100 kΩ is the sweet spot. Going stronger (10 kΩ) shortens the power-up transition to 0.65 µs but costs 330 µA whenever the pin is driven high; going weaker (1 MΩ) stretches it to 65 µs and starts to compete with the LVC's own input leakage (`Ioff` ±10 µA would swamp a 1 MΩ pull — **do not go above ~200 kΩ**).

**Bonus note on the inverter pair.** `Q6`'s gate gets its signal one `tpd` (≤3.3 ns) after `Q5`'s, so during a colour change both FETs are briefly on. Since both sources tie to the same `FB` node, that momentarily parallels the two LED strings rather than shorting anything, and the FB loop regulates total current — harmless.

---

## 3. 🟡 Margin problems

### 3.1 ~~USB-priority threshold~~ — ✅ FIXED (`R38` = 240 k)

`R38 = 310 k`, `R51 = 100 k`, TPS2116 `V_REF = 1.00 V` (0.92 – 1.08 V):

```
V_switchover = 1.00 V × (310k + 100k) / 100k = 4.10 V typ   (3.77 – 4.43 V over tolerance)
```

**What USB_VBUS actually is.** The mux sees VBUS *minus* the polyfuse *minus* the Schottky:

| Element | Datasheet value | Drop |
|---|---|---|
| `F1` 0805L100WR | Rmin **0.060 Ω**, R1max **0.210 Ω** (post-reflow or post-trip) | 6 – 210 mV |
| `D1` B5819W | **Vf ≤ 0.60 V @ 1 A**, 25 °C (only spec'd point) | ~0.31 – 0.60 V |

The fuse is *not* the problem — **`D1` is**, contributing 0.31–0.60 V. (I over-estimated F1 in my first pass; its real drop is ≤210 mV.)

From a 4.75 V source (the USB 2.0 receptacle minimum for a high-power port), against the **worst-case 4.43 V threshold**:

| Load | USB_VBUS (R1max fuse) | Margin | Result |
|---:|---:|---:|---|
| 100 mA | 4.42 V | −0.01 V | switches to battery |
| 250 mA | 4.32 V | −0.11 V | switches to battery |
| 500 mA | 4.19 V | −0.23 V | switches to battery |
| 1000 mA | 3.94 V | −0.49 V | switches to battery |

Even with a **virgin** fuse and only 250 mA it still fails. So on a worst-case-`V_REF` part this design **never** takes USB priority under any meaningful load — and on a typical part it sits right on the boundary.

**Why that's bad — it oscillates.** When the mux hands over to the battery, the USB load drops to just the 100 mA charger, `USB_VBUS` recovers to ~4.4–4.7 V, `PR1` goes back above threshold, the mux switches back to USB, the load reappears, the rail sags, and it switches away again. **TI does not publish any `PR1` hysteresis** — `V_REF` has a min/typ/max but no hysteresis parameter anywhere in SLVSFG1A — so you must design as though it is **zero**. Each transition is break-before-make (§7.6.1: *"the output is unpowered and will dip depending on the load current and output capacitance"*), `t_SW` = 8 µs, which with `C4` = 22 µF costs a **182 mV dip on LDO_IN at 500 mA** every cycle. Sustained, that's rail noise, EMI, and a conducted-emissions problem.

It also corrupts the status readout — this is the direct cause of the reachable `ST+CHRG` state in §2.1.

**Fix A — one resistor, zero risk: `R38` = 310 k → 240 k.**

```
V_switchover = 3.40 V typ  (3.13 – 3.67 V)
```

Margin becomes **+0.27 V even at 1 A from a 4.75 V source**, and +0.74 V at light load. The threshold still sits above the point where the LDO can no longer hold 3V3 (needs ≥3.72 V input at 600 mA worst case), so genuine brown-out handover still works.

**Fix B — better, if you're revising the board: also replace `D1` with a 0 Ω link, and set `R38` = 270 k** (3.70 V typ, 3.40–4.00 V). This restores 0.31–0.60 V of headroom and gives **+0.54 V margin at 1 A**, while keeping the handover threshold high enough to be useful.

Is `D1` removable? It is largely redundant: the **TPS2116 already blocks reverse current** (V_RCB 42 mV typ), and the TP4056 does not drain BAT when VCC is absent. `D1` protects only against back-driving the connector's VBUS pin from `C2`/`C25` (11 µF — trivial) and against reverse polarity on VBUS (essentially impossible on USB-C). It does **not** protect against hot-plug overshoot — that's `CR1`'s job, and note that hot-plug ringing can reach ~2× VBUS, i.e. above the TPS2116's **6 V** and the TP4056's **8 V** absolute maximums, so confirm `CR1` actually clamps below 6 V. Keep `D1` if you want belt-and-braces; just pay for it with Fix A's lower threshold.

**Do *not* try to fix this with a capacitor on PR1.** It's the intuitive "safety net" but it's wrong here: on a real unplug you need the handover to complete before `C4`+`C6`+`C32` (≈66 µF) collapse, which at 500 mA allows only ~26 µs for a 200 mV droop. Any RC big enough to filter a millisecond-scale Wi-Fi TX sag (≥100 nF against the 76 kΩ divider ≈ 7.6 ms) would leave the rail unpowered far too long. The datasheet neither recommends nor discusses a PR1 capacitor. Keep PR1 fast and set the *threshold* correctly instead — limit any PR1 cap to ≤1 nF for noise only.

**A tempting trick to avoid:** adding a feedback resistor from `ST` back to `PR1` would synthesise hysteresis (ST pulls low on battery, dragging PR1 down and latching the decision). It works electrically, but `ST` is also the 150 k leg of the `USB_STAT` ladder — the feedback resistor would inject the PR1 divider into the status ladder whenever ST is Hi-Z and corrupt every reading in §2.1. If you want hysteresis, use a spare GPIO and the TPS2116's manual mode instead.

### 3.2 ✅ **LARGELY RESOLVED** — AP2112K-3.3 thermal (LED boost moved off the 3V3 rail)

I originally called this "little current headroom". Computing the dissipation makes it much more serious. SOT-23-5 with no thermal pad, `R_θJA` ≈ 250 °C/W on a small copper island (150 °C/W with a generous pour — use both bounds):

| V_IN | I_OUT | P_diss | ΔT @250 °C/W | ΔT @150 °C/W |
|---:|---:|---:|---:|---:|
| 4.6 V (USB) | 300 mA | 390 mW | **+98 °C** | +59 °C |
| 4.6 V (USB) | 500 mA | 650 mW | **+163 °C** | +98 °C |
| 4.2 V (full cell) | 500 mA | 450 mW | **+113 °C** | +68 °C |
| 5.0 V | 600 mA | 1020 mW | **+255 °C** | +153 °C |

The AP2112K has thermal shutdown, so it will not fail — it will **fold back and drop the 3V3 rail** during exactly the events you care about (Wi-Fi TX + LED on + e-paper refresh). At 25 °C ambient you reach shutdown somewhere around 300–400 mA sustained.

The dropout is `(V_IN − 3.3) × I`, so the fix is to reduce either term:

1. **Move the LED boost off `3V3`.** It currently draws **131 mA at 24.5 V out** — through the LDO. Feeding `U10` from `LDO_IN` or `P+` instead removes ~130 mA *and* eliminates a pointless double conversion (battery → LDO → boost). This is the single highest-value change and is already item 15. `Q7`'s gate drive needs re-working since `LDO_IN` > 3.3 V (a small NPN or N-FET level shifter).
2. **Duty-cycle the load in firmware** — never run LED + Wi-Fi TX + refresh concurrently.
3. If you respin, consider a **buck** instead of an LDO, or at minimum a package with a thermal pad.

Note this interacts with §3.1: the *worse* the LDO thermals, the more you want the mux to stay on USB (higher V_IN means more dissipation, but switching to battery at 3.7 V actually **reduces** LDO dissipation). Running from the battery is thermally *better* — another reason not to over-tune the threshold upward.

### 3.3 ~~Boost output capacitor will derate badly~~ — ✅ FIXED (`C9` = 4.7 µF/50 V)

`C9 = 1 µF`, annotated **50 V**, in an **0805** footprint. The TPS923610 reference design calls for `C_OUT = 1 µF`. An 0805 X7R 1 µF/50 V typically loses **60–70 % of its capacitance at 20 V DC bias** → you'll have ~0.3–0.4 µF where the datasheet wants 1 µF, which shows up as extra output ripple and worse LED current regulation.

**Fix:** use **2.2 µF/50 V 0805** (still ~0.7–0.9 µF derated) or a **1 µF/100 V in 1206**. Same applies to `C12` (4.7 µF C_IN) but at 3.3 V bias the derating is mild.

### 3.4 ~~`D3` SMAJ30A~~ — ✅ FIXED (SMAJ26A fitted). Re-evaluation retained below

**Your stated intent:** ESD protection on `LED_SW` where it leaves the board via `J6`, with the rail bounded by the internal OVP (~25 V) and a software OVP at 20 V via `LED_MONIT`. That reframes the question, so here is the evaluation on those terms.

**The intent is right and the part class is right.** `LED_SW` exits on a user-accessible header *and* on `J3` to an off-board LED strip with cable inductance — so you want something that handles both an ESD strike and a modest inductive/hot-plug surge. A 400 W SMA TVS is a sensible choice for that; a tiny low-capacitance ESD-only diode would be worse here, and capacitance is irrelevant on a 15 mA DC rail. Keep the topology.

**Only the voltage grade is wrong.** For a clamp to do anything useful it must start conducting *below* the protected pin's absolute maximum. `U10`'s abs max on `VOUT`/`SW` is **32 V**:

| Part | Standoff | V_br (min) | vs `U10` 32 V abs max |
|---|---:|---:|---|
| **SMAJ30A** (fitted) | 30 V | **33.3 V** | ❌ breakdown is *above* 32 V — U10 is already out of spec before the TVS turns on |
| **SMAJ26A** | 26 V | 28.9 V | ✔ conducts inside the useful window, and 26 V standoff is safely above the 25.5 V max OVP |
| SMAJ24A | 24 V | 26.7 V | ⚠ standoff too close to the OVP ceiling — would sit in the leakage knee if OVP ever trips |

With your software OVP at 20 V the rail normally lives at ≤20 V, so a 26 V standoff part never conducts in operation, and its leakage there is negligible (<1 µA).

**Verdict: keep `D3`, change the grade to SMAJ26A.** It is a one-character BOM change that moves the clamp from "never engages before damage" to "engages 3 V before damage".

**Two honest caveats:**

1. Even SMAJ26A clamps at ~42 V at its full 400 W rating, which is still above 32 V. For a *large* strike `U10` sees an overshoot regardless. That is acceptable — ESD is nanoseconds and abs-max ratings are DC limits — but do not read the TVS as a guarantee. Your real protection against sustained overvoltage is the internal OVP; `D3` is for transients only.
2. **Placement decides whether it works at all.** `LED_SW` reaches *both* `J3` (pins 1, 5) and `J6` (pin 5), and one TVS cannot be adjacent to both. Put `D3` at whichever connector is more exposed — I'd say `J6`, since that is the user-accessible header — and accept longer inductance to the other, or fit a second clamp. The return side is already covered: `D8` (`PESD2IVN-UX`) sits on `C−`/`W−`, which serve both connectors. ✔

### 3.5 ~~`L2 = 4.7 µH`~~ — ✅ **RESOLVED**: `74479325207247`, I_sat 2.7 A confirmed

My first pass flagged this as a current-limit risk. Having computed the operating point, **it isn't** — and the confirmed part has enormous margin:

| V_OUT | D | I_in | ΔI_L pk-pk | I_peak | vs 1.8 A I_LIM | vs 2.7 A I_sat |
|---:|---:|---:|---:|---:|---:|---:|
| 15.0 V | 0.780 | 80 mA | 498 mA | 329 mA | 18 % | 12 % |
| 20.0 V | 0.835 | 107 mA | 533 mA | 374 mA | 21 % | 14 % |
| 24.5 V | 0.865 | 131 mA | 552 mA | 408 mA | **23 %** | **15 %** |

**I_sat 2.7 A against a 408 mA worst-case peak is 6.6× margin.** Closed — no action.

The converter runs deep in **DCM** at this power level (ΔI_L is 4–6× I_in), and would even with the datasheet's 10 µH reference, because that reference is drawn for 60 mA of LEDs and you run 15 mA. Normal and expected.

Residual note (layout, not schematic): 550 mA pk-pk of triangular ripple at 1.1 MHz is a meaningful radiator on a 2-layer board. Keep the `L2`/`C12`/`C9` loop tight.

*(Minor: the MPN as written, `74479325207247`, is 14 digits — Würth WE-series numbers are normally 11. Worth eyeballing the digits before it goes into the BOM.)*

### 3.5.1 ⬜ **OUTSTANDING** — `L1` (22 µH, `NR3015T220MNGH`) is NRND and needs a replacement

**Land pattern that must be matched** (read from the PCB, footprint `srn3010C-100m:IND_SRN3010C-100M`):

| | |
|---|---|
| Pads | 2 × (1.30 mm × 3.50 mm) |
| Pad centres | ±1.10 mm → **2.20 mm pitch** |
| Courtyard | 4.0 × 4.0 mm |
| Body | **3.0 × 3.0 mm** (the SRN3010 / SRN3015 / NR3015 / SWPA3015 family) |
| Height | original is **1.5 mm** — treat that as the mechanical budget |

**Electrical requirements — and the important part is that saturation is *not* the binding constraint.**

`L1` is not in a converter you control. The **panel** drives `Q4`'s gate via `GDR` and senses the current across `R14` (3 Ω) via `RESE`; its internal controller sets the peak current and the on-time. So the peak is fixed by the panel's sense threshold:

| Panel sense threshold | I_peak through `L1` |
|---:|---:|
| 200 mV | 67 mA |
| 300 mV | 100 mA |
| 450 mV | 150 mA |
| 600 mV | 200 mA |

**Peak current is only 65–200 mA**, so even a 0.5 A part gives 2.5× margin and anything ≥1 A is generous. Saturation is easy to satisfy here.

**⚠️ Recommendation: stay at 22 µH — do not increase the inductance.** You asked about going larger, and in a converter you control that would be fine. Here it is a real risk:

* On-time scales with L: reaching 150 mA takes 1.0 µs at 22 µH but **2.1 µs at 47 µH** and 4.5 µs at 100 µH.
* If the panel's controller uses a **fixed on-time or a maximum duty limit** rather than pure peak-current termination, a larger inductor never reaches the intended peak — which *reduces* the energy per cycle and therefore the current available to `PREVGH`/`PREVGL`. The symptom is a panel that works on small updates and browns out its gate rails on a full refresh, which is a miserable thing to debug.
* 33 µH is a modest, probably-safe step if 22 µH is unobtainable; treat 47 µH+ as requiring validation against your panel.

So: **same 22 µH, same 3.0 × 3.0 mm land pattern, shielded, height ≤ 1.5 mm.**

#### Original status: **Obsolete**, not merely NRND

DigiKey's product page for `NR3015T220M` (DK 1008266) shows **Part Status = `Obsolete`** with ~1 pc left; TTI's listing reads *"PLEASE SEE SUGGESTED ALTERNATE LSXBD3030QKT2R2M"* with 0 stock. The exact `-NGH` suffix returns **no results** in any authorised distributor feed. Don't re-buy — the only sources are brokers.

#### Candidates — stock verified 2026-09-09

| Mfr / MPN | L | I_sat | I_rated | DCR max | H | Core | DigiKey | LCSC | JLC |
|---|---:|---:|---:|---:|---:|---|---|---|---|
| *(old)* Taiyo Yuden NR3015T220M | 22 µH | 470 mA | 470 mA | 624 mΩ | 1.5 | ferrite | **Obsolete** | — | — |
| **Bourns SRN3015-220M** | 22 µH | **580 mA** | 600 mA | 622 mΩ | 1.5 | ferrite | **94,719** $0.21–0.38 | ✗ not in JLC lib | — |
| **TDK VLS3015CX-220M-1** | 22 µH | 570 mA | 870 mA | **496 mΩ** | 1.5 | ferrite, shielded | **10,280** $0.14–0.26 | **C3222413** 2,422 $0.141 | Ext |
| **TDK VLS3012HBX-220M** | 22 µH | ~1.09 A | **1.12 A** | 761 mΩ | **1.2** | **metal composite** | **11,312** $0.28–0.50 | **C350879** 5,252 $0.172 | Ext |
| **Sunlord SWPA3015S220MT** | 22 µH | 520 mA | 570 mA | 598 mΩ | 1.5 | shielded | 0 | **C83438 57,198** $0.057 | Ext |
| FNR3015S220MT | 22 µH | 680 mA | 690 mA | 598 mΩ | 1.5 | shielded | — | C167761 29,342 $0.042 | Ext |

#### Recommendation — and note the selection criterion changed

You asked for higher inductance and plentiful stock. Because peak current here is only **65–200 mA**, *every* candidate clears saturation with ≥2.5× margin — even the obsolete original's 470 mA was never the limitation. **I_sat is not a useful tiebreaker for this position.** DCR is not either: at ~100 mA RMS the difference between 496 mΩ and 761 mΩ is 5 mW vs 7.6 mW. So the real criteria are **availability** and **EMC**.

Since you assemble at JLCPCB (`jlcpcb/production_files/`), LCSC stock matters as much as DigiKey:

* 🥇 **TDK VLS3015CX-220M-1** — the cleanest single BOM line. **Active, stocked at both DigiKey (10,280) and LCSC (C3222413)**, properly shielded, and the **lowest DCR of the group (496 mΩ)**. One MPN you can buy for both JLC runs and hand builds.
* 🥈 **TDK VLS3012HBX-220M** — choose this if EMC is the priority. **Metal-composite core** has markedly lower fringing flux than a ferrite drum, which is worth real money on a 2-layer board with an unbroken-reference problem. Also **1.20 mm tall** (0.3 mm *shorter* than the original) and dual-sourced. Costs ~2× and has the highest DCR, neither of which matters here.
* 💰 **Sunlord SWPA3015S220MT (C83438)** — if you just want the cheapest JLC-friendly part with deep stock: **57,198 pcs at $0.057**. Electrically a straight upgrade on the original (520 mA vs 470 mA, 598 mΩ vs 624 mΩ).
* ⚠️ **Bourns SRN3015-220M** has by far the deepest DigiKey stock (94,719) and is the cheapest US option, but it is **not in the JLCPCB assembly library** — so it would force a hand-placed part. Bourns also calls the SRN3015 *semi-shielded* despite DigiKey's "Shielded" field.

**Two caveats before you commit:**

1. **Verify the land pattern against the datasheet drawing.** The compatibility claim above rests on all candidates sharing the industry-standard 3.0 × 3.0 "3015" outline — DigiKey confirms `Size = 3.00 mm × 3.00 mm` for each, but the manufacturers' recommended-land drawings could not be machine-read (Bourns' PDF returns binary; TDK returns HTTP 403). Your pads (1.30 × 3.50 mm, 0.90 mm inner gap) are generous for this family, but eyeball the drawing — especially for **VLS3012HBX**, since metal-composite parts occasionally use narrower terminations.
2. **Avoid `TDK VLS3015ET-220M`** — it is being retired; Mouser's listing already redirects to `VLS3012CX-220M-1`.

*(Price caveat: DigiKey and LCSC render price breaks client-side, so exact qty-10/qty-100 figures could not be read. The ranges above are the true min–max from the distributor feed — qty 10 and 100 fall inside them.)*

### 3.6 ✅ **FIXED** — `PWR_BUTTON` pull-down added (`R76` 100 k)

With `R72` correctly DNP, IO18 connects only to `C29` (2.2 nF) and `R62` (10 k) → `SW10` → 3V3. Open switch = floating input. It will *mostly* sit low through leakage, but it isn't deterministic, and for `esp_sleep_enable_ext0_wakeup(..., 1)` you must have a real pull-down.

**Fix:** add a **100 kΩ–1 MΩ pull-down on PWR_BUTTON**, or explicitly enable the internal pull-down (note the internal one is ~45 kΩ, which with `R62` 10 k still gives 2.7 V when pressed — that reads high, so it works, but make it deliberate).

### 3.7 ~~Strapping pins `IO45`/`IO46` exposed on the dev header~~ — 🚫 DECLINED (software pull-downs)

`UNUSED_GPIO_45 → J6.3` and `UNUSED_GPIO_46 → J6.2`, each via a 33 Ω series resistor, with no pull resistor.

* **GPIO45 = VDD_SPI select.** Must be **LOW at reset** for 3.3 V flash. It relies on the module's weak internal pull-down. Anything a user attaches to J6 pin 3 that sources current at reset will brick the boot.
* **GPIO46 = boot/ROM-message strap**, same weak internal pull-down.
* **GPIO3 = JTAG source select** — floating is the intended default, lower risk.

**Fix:** add **10 kΩ pull-downs on `UNUSED_GPIO_45` and `UNUSED_GPIO_46`** so the strap state is set by the board, not by whatever is plugged into the header. This is cheap insurance against a very confusing failure mode.

### 3.8 ⬜ **OUTSTANDING** — `J6`: `LED_SW` adjacent to `SDA` (see §6.2 for the corrected, dual-row analysis)

J6 pinout: `1 GND, 2 IO46, 3 IO45, 4 SDA, 5 LED_SW, 6 W−, 7 3V3, 8 P+, 9 IO3, 10 SCL, 11 C−, 12 GND`.

**Pin 5 is LED_SW (up to 24.5 V) and pin 4 is I²C SDA.** A single slip, a bridged solder joint, or a misaligned connector puts 24 V onto the I²C bus — which is shared by the ESP32, the DS3231, and the touch panel. The TSD05C/TPD4E1U06 clamps will conduct but will not survive a *sustained* 24 V; they're ESD devices, not fuses.

Related: because Q7's body diode (D→S, toward 3V3) is in series with the TPS923610's high-side body diode (SW→VOUT), an **external source applied to J6 pin 5 back-feeds the 3V3 rail** even with the LED driver disabled.

**Fix:** move `LED_SW` and `W−`/`C−` to one end of the header with a GND pin between them and the logic pins (e.g. `… 3V3, GND, LED_SW, C−, W−`). If the connector is already fixed in the mechanical design, at minimum document it loudly and consider a series PTC on pin 5.

### 3.9 ~~`R57`/`R56` sit directly across the cell, outside the protection~~ — **FIXED**

`R57 = 100 k` (B+ → Q3 gate) and `R56 = 1 k` (Q3 gate → B−) formed a divider **directly across the cell**, bypassing the FS8205A — **~41 µA of continuous drain the protection IC could never cut off**, and it kept draining *after* the DW01A cut off at 2.4 V, so a shelf-stored board would eventually pull the cell into deep-discharge damage.

**Now `R57 = 1 M` / `R56 = 10 k`.** Confirmed no functional impact: the ratio is unchanged, so `Q3`'s gate still sits within ~42 mV of B− and the P-FET is just as fully enhanced (`V_GS(th)` is ~1 V; you have ~4.2 V of drive). Drain drops **41 µA → 4.2 µA**, a 10× improvement, and it is now the *smallest* term in the standby budget rather than the largest. The only theoretical cost is a ~10× slower gate transition on battery insertion (RC against `Q3`'s ~1 nF `Ciss` ≈ 10 µs instead of 1 µs), which is irrelevant for a connector event. ✅

### 3.10 ~~DW01A `R1 = 100 Ω`~~ — 🚫 DECLINED (reasonable). Analysis retained below

You reported that a 100 Ω here dropped so much voltage that your LDO stopped working, and you had to replace it with a 0 Ω jumper. **That symptom means the resistor was in the main current path, not in the VCC branch** — and it is worth understanding, because the correct placement cannot do that.

The DW01A's `VCC` pin is a **sense/supply input drawing `I_CC` = 3 µA typ**. It is not a power path. Through 100 Ω that is:

```
100 Ω × 3 µA = 0.0003 V   (0.3 mV)
```

If your old board dropped enough to kill an LDO, the resistor must have been carrying load current — e.g. inserted into `P+` itself so that everything downstream (mux, LDO, charger) fed through it. At 300 mA a 100 Ω would drop 30 V, i.e. the rail simply collapses. That matches your symptom exactly.

**The correct placement is a dead-end branch.** In this schematic `U5.5 (VCC)` currently connects directly to the `P+` net, which is shared with `U2.VIN2`, `U11.BAT`, `C3`, `C26`, `R12`, `CR3` and `J6.8`. To add `R1` you must create a **new net** that has only three things on it:

```
P+ ──[R1 100 Ω]──┬── U5.5  (DW01A VCC)
                 └── C7 0.1 µF ── B−
```

`C7` **must move** to the IC side of the resistor — otherwise there is no RC filter, just a resistor. Nothing else may connect to that node; if anything that draws real current lands on it, you reproduce the old failure.

**Is it worth it?** Marginal. The DW01A works fine in millions of packs without `R1`; its purpose (§12.2) is to *"suppress the ripple and disturbance from charger"*. `R16` = 1 k on `CS` is already doing the more important latch-up job. Given your history with this component, **it is entirely reasonable to skip it** — just don't skip it because of the voltage drop, because in the correct position there isn't one.

---

## 4. 🟢 ERC, hygiene and verification

`kicad-cli sch erc --severity-all` → **1 error, 45 warnings.**

**The one error — fix it:**
* `[pin_to_pin] Power output ↔ Power output`: `U10.4 (GND)` vs `#FLG03`. The **TPS923610 symbol declares its GND pin as `power_output`**, which is wrong — it should be `power_input`. This is also the source of ~14 of the "Unspecified pin" warnings (U10's VIN/ADIM/FB/SW are all typed `unspecified`). Fix the symbol's pin electrical types; it will clear the error and most of the noise.

**Warnings worth acting on:**
* `[lib_symbol_mismatch]` ×6 — `Conn_01x02`, `D_TVS_Dual_ACA`, `FH34SRJ-6S-0.5SH_50_` (J3 and J4), `TSD05CDYFR` (CR2, CR3). The cached symbols in the schematic differ from the libraries. **Resolve these before fab** so what you review is what you build.
* `[footprint_link_issues]` ×4 — **CR1/CR2/CR3 (TSD05CDYFR) are assigned `D_SOD-323` but the symbol's filter expects `TVS_SOD2_DYF_TEX`.** Verify the real package of the TSD05C you're buying; a SOD-323 vs DFN mix-up is a board-scrapper. `F1` (polyfuse in a plain fuse footprint) is cosmetic.
* `[multiple_net_names]` ×6 — `SD_CMD`/`CMD`, `SD_DAT0..3`/`DAT0..3`, `TP_RST`/`PIN_3`. Harmless (the global label wins) but it makes the netlist confusing; delete the redundant local labels.
* `[single_global_label]` ×2 — `EINK_SW` and `3031_SW` are global labels used once. Make them local labels. (`3031_SW` also looks like a leftover name from a previous LED-driver part number — rename to `LED_L2_SW`.)
* `[four_way_junction]` ×3 — at (256.5, 113.0), (417.8, 184.2), (519.4, 105.4). Style only, but visually check each: four-way junctions are the classic way an unintended connection hides in plain sight.
* `[simulation_model_issue]` ×3 — Q3/Q7/Q8 point at an `irlml6402` SPICE model in a non-existent library. Cosmetic (and note the **symbol is IRLML6402 while the value/BOM is AO3419** — the parts differ in Rds(on) and V_GS(th); make sure your simulations and your BOM agree).

**Stale generated artefacts — regenerate before ordering:**
* `minRead_pcb.net` is from **2026-08-21**; the schematic was edited **2026-09-09**.
* `minRead_pcb.csv` (BOM) is from **2026-08-06** and is **materially wrong**: it lists `C9` as 4.7 µF (it is now 1 µF), `L1,L2` as both 22 µH SRN3010C (L2 is now 4.7 µH/1008), includes a `D7` that no longer exists, and has no entries for TPS923610 or DS3231MZ.
* `production_files/` gerbers, BOM and CPL are from **2026-08-13**.

**Verify before ordering:**
* **ESP32-S3-WROOM-1 variant.** Your note *"ESP32-S3-WROOM-1 does not reserve GPIO for PSRAM"* is only true for the **non-octal** variants. `TP3/TP4/TP5` sit on IO35/36/37, which **are** consumed by octal PSRAM. Pin the BOM to a specific MPN — `N4`/`N8`/`N16` or `N8R2`/`N16R2` (quad PSRAM, uses the flash bus). **Do not order `N8R8`/`N16R8`.**
* **FS8205A vs FS8205 naming.** Fortune's own `FS8205A-DS-17_EN.pdf` describes a **TSSOP-8** part; the SOT-23-6 device is documented as **`FS8205`** (rev 1.7). LCSC C32254 is the SOT-23-6 one. Your footprint is SOT-23-6, so order **C32254** and don't let a distributor substitute a TSSOP-8 "FS8205A". Also: older FS8205 datasheets (rev <1.4) show a *different* pinout — only trust rev ≥1.7.
* **J2 pins 1/6/7** (`HLT_CTL`, `TSCL`, `TSDA`) left NC — correct for a non-touch panel driven without HLT_CTL, but confirm against your exact 4.26" panel.
* **`C19` on `VPP`** (J2 pin 19). VPP is the OTP programming pin; most reference designs leave it NC. A 1 µF to GND is harmless but check the panel datasheet.

**Design notes / minor:**
* **DS3231MZ in VBAT-only mode — three firmware consequences:** (a) the oscillator **does not start until you perform a valid I²C write**, because VCC never rises above V_PF — your init code *must* touch the RTC or the clock never runs; (b) `RST` is permanently held low (you correctly left it NC); (c) temperature compensation runs every 10 s instead of every 1 s, so expect slightly worse drift. Also: with VBAT on the 3V3 rail there is no true backup — the RTC loses time if the cell is removed or fully discharged. `C22 = 1 µF` ✔, but the datasheet asks for a **low-leakage** cap here.
* **`BAT_MONIT` senses `P+`/`GND` rather than `B+`/`B−` — this is the right choice, keep it.** Your stated reason (ESP32 safety) is valid, and there is a second reason that is arguably stronger.

  **Reason 1 — reverse insertion.** On `P+`/`GND`, `Q3` blocks and `P+` simply stays near 0 V; the ADC pin never goes negative. On `B+`/`B−`, a reversed cell puts `B+` 4.2 V *below* ground, so the divider would drive `BAT_MONIT` to about −2.1 V, forward-biasing the ESP32's substrate diode. `R12` limits it to ~4 µA so it would survive, but it is an avoidable abuse of the pin.

  **Reason 2 — leakage, and this is the better argument.** `R12`+`R10` = 2 MΩ draws **2.1 µA at 4.2 V**. On `P+`/`GND` that current flows *through* the protection FETs, so the **DW01A can cut it off** at over-discharge. On `B+`/`B−` it would bypass the protection entirely and become a permanent, unprotectable cell drain — precisely the `R56`/`R57` problem you just fixed. Sensing `P+` keeps the fuel gauge inside the protection boundary.

  **Reason 3 — it measures what actually powers the system.** `LDO_IN` is fed from `P+` via the mux, so `P+` is the voltage that determines whether the board can still run. That is more actionable than the raw cell voltage.

  **The cost is a small load-dependent offset.** The cell sits behind `Q3` (~45 mΩ) + `R27` (0 Ω) + `Q8` (~45 mΩ) on the positive side and the FS8205A pair (~56 mΩ) on the negative side — **~146 mΩ total** — so `P+ − GND` reads low by `I × 146 mΩ`:

  | Load | Error | % of the 3.0–4.2 V span |
  |---:|---:|---:|
  | 10 mA (sleep) | 1.5 mV | 0.1 % |
  | 50 mA (idle) | 7.3 mV | 0.6 % |
  | 150 mA | 21.9 mV | 1.8 % |
  | 300 mA | 43.8 mV | 3.6 % |
  | 500 mA (Wi-Fi TX) | 73.0 mV | 6.1 % |

  **Mitigation is free and you should do it anyway:** sample `BAT_MONIT` only in a known-quiet window (radio idle, LED off, no SD activity). At <50 mA the error is under 8 mV, which is well inside the ADC's own accuracy. You need that quiet window regardless — the 500 kΩ source impedance and `C8`'s settling time mean you should not be sampling during a TX burst. If you ever want better, characterise the offset against load once and correct in firmware; a second sense point is not worth the parts.

  *(Note: when the protection actually trips on over-discharge, `P+` collapses — but so does `LDO_IN`, so the ESP32 is unpowered and cannot read anything either way. Sensing `B+` would not buy you a reading in that state.)*
* **Button ladders are well chosen.** Ladder 1 (10 k pull-up; 100/5.6 k/20 k/56 k) → 0.03 / 1.18 / 2.20 / 2.80 V. Ladder 2 (10 k; 100/12 k/33 k/68 k) → 0.03 / 1.80 / 2.53 / 2.88 V. Good separation for single presses. **But chords are not resolvable** — e.g. ladder 1 SW3+SW9 gives 1.11 V vs SW3 alone at 1.18 V (70 mV apart). Firmware must treat these as single-press-only. The 2.2 nF caps give ~20–120 µs — that's anti-aliasing, not debounce; debounce in software.
* **No hardware power-off.** `U3.EN` is tied to `LDO_IN` permanently, so the 3V3 rail is always live and "off" means ESP32 deep sleep. Since SW10 pulls IO18 to *3V3*, the power button only works while the rail is up — which is consistent, but means total shelf life is set by the ~120–150 µA standby budget (dominated by item 3.9). If you ever want true off, bring `EN` out to a latch.
* **No dedicated VBUS-present GPIO** — plug detection is inferred entirely from the `USB_STAT` ADC ladder, which is why 2.1 matters.
* `D2` power LED: `R59 = 2 k` → `(5 − 2.0)/2000 = 1.5 mA`, but the note says *"2.5mA at 2.0Vf"*. The formula written next to it — `(5 − 2.0)/0.0015` — is the 1.5 mA one. Just fix the label.
* `R45`/`R58` (both DNP) connect **the same two nets** as `R52`/`R46` — the "alternate configuration" only actually swaps VDD/INT between J4 pins 2 and 4; SDA/SCL don't move. Either delete R45/R58 as dead BOM entries, or rewire them to genuinely cross SDA/SCL (`R45: SCL→PIN_5`, `R58: SDA→PIN_6`) if that was the intent.
* `D1` value is `B5819W` but the datasheet field points at `MBR0520~MBR0580`. Both are 40 V SOD-123 Schottkys so it works; just tidy the field.
* `Q4` (e-paper boost FET) has no gate series resistor on `GDR` — `R15 = 10 k` is a pull-down only. A 10–33 Ω series resistor in the gate would soften the edges and help radiated emissions on a 2-layer board, at negligible cost.

---

## 5. Change list — status

*Verified against `minRead_pcb.kicad_sch` as of 2026-09-09 13:27. Values below were read back out of the file, not taken on trust.*

### ✅ Done — confirmed in the schematic

| # | Change | Verified |
|---|---|---|
| 1 | `R17` 100 k → **150 k** | reads `150k` |
| 3 | **`R75` 100 k pull-down on `COLOR_SEL`** | present, `100k` |
| 10 | **`R76` 100 k pull-down on `PWR_BUTTON`** | present, `100k` → pressed reads 3.00 V, 525 mV over VIH |
| 6 | `R38` 310 k → **240 k** | reads `240k` |
| 7 | `R57`/`R56` → **1 M / 10 k** | reads `1M` / `10k` |
| 11 | `C9` 1 µF → **4.7 µF**, annotated 50 V | reads `4.7u` |
| 12 | `D3` SMAJ30A → **SMAJ26A** | reads `SMAJ26A` |
| 24 | `R47`/`R48` 4.7 k → **2.2 k** | both read `2.2k` |
| 25 | `C31` 10 nF → **100 nF** | reads `100n` |
| 15 | **Deleted the LED power-gate; `U10.VIN` → `LDO_IN`** | netlist: `U10.1 -> LDO_IN`, `C12`/`L2` moved with it. `LED_ACTIVATE` net is gone |
| 27a | **microSD gating built on `Q7`** | `Q7`: G=`SD_ACTIVATE`, S=`3V3`, D=`SD_VDD`; `R40` 100k pull-up; `J7.4 -> SD_VDD` |
| 27b | **All five pull-ups + both decaps moved to the switched rail** | `R8/R9/R53/R54/R55` and `C36`/`C37` now read `SD_VDD`, not `3V3` — the phantom-power trap is closed |
| 27c | **`SD_ACTIVATE` driven from `IO41`** | The recommended pin: not a strapping pin, not in the glitch table, no internal pull at reset |
| 27d | **Bleed `R77` = 100 k on `SD_VDD`** | 33 µA idle; 396 ms discharge, or 7.9 ms with SD lines driven low |
| 27e | **Gate `R78` = 1 k from `IO41`, `R40` 100 k on the FET side** | Vgs = −3.27 V on / 0 V off. R40 placement verified correct |
| 27f | **`C37` 4.7 µF → 1 µF** | Inrush 19.1 → 6.93 µC; 3V3 dip **375 → 136 mV**. Replaced the need for a gate cap |
| 5 | **`TPS923610` symbol GND → `power_input`** | **ERC now 0 errors** (was 4) |
| 33 | **`#FLG05` removed from `LDO_IN`; PWR_FLAG added to `SD_VDD`** | Cleared both new ERC errors |
| 21 | **`/50V` appended to `C13`–`C17`, `C20` values** | Now visible in the JLC `Comment` column |
| 14 | **`J6` re-ordered — fuller fix applied** | `4=GND, 8=SDA, 12=P+`. `LED_SW` now bounded only by GND/W−/C−; `P+` cornered by the LED returns |
| 31 | **`L1` → `VLS3012HBX-220M`** | Metal-composite, 1.2 mm tall, dual-sourced |
| 2 | **`J2` symbol pin names corrected** | pin 4 now `NC`; VGH/VGL/VSH/VSL now read correctly |
| 34 | **Stale `Switchover at ~4.1V` note → `~3.4V`** | Matches `R38` = 240 k |
| 4 | **LED note corrected** | now reads `10kHz-25kHz` and `0-24.5V` |
| 31a | **`L1` replacement chosen: `VLS3012HBX-220M`** | annotated on the sheet (see ⚠️ below about where it needs to go) |
| 21a | **50 V annotated on `C13`–`C17`, `C20`, `C9`** | text placed beside each cap (see ⚠️ below) |

**Bonus outcome:** items 1 + 6 together make the previously-ambiguous `ST`+`CHRG` ADC state **physically unreachable** (§2.1) — the mux switchover ceiling, 3.67 V, is now below the TP4056's ~4.0 V operating floor, so the two can no longer both assert.

---

### ⬜ OUTSTANDING — still to do

**Blockers (do before ordering):**

| # | Change | Why it still matters |
|---|---|---|

**Should do:**

| # | Change | Note |
|---|---|---|
| **35** | **Measure ΔV_f between the warm and cool frontlight strings** — then keep `C9` at 4.7 µF (if <0.2 V) or drop it to 1–2.2 µF (if >0.5 V) | §6.17: `C9` re-slew is the binding limit on CCT blend rate, not the boost loop | 🟡 |
| 4 | Finish the LED note: still says **"5kHz-25kHz"** (datasheet min is 10 kHz — use 20–100 kHz) and **"0-30V"** (part is 24.5 V max). The `~24V` OVP edit is done | 2 of 3 sub-items outstanding |
| 14 | `J6`: swap `SDA` (pin 4) ↔ `GND` (pin 12) | Only real HV adjacency; `W−`/`C−` turned out to be low-voltage nets |
| 2 | `J2` symbol: pin 4 `VGL` → **NC**, audit the other 23 names | Circuit is correct; the symbol caused a false finding |
| 18 | Resolve 6 `lib_symbol_mismatch`; **regenerate BOM / netlist / gerbers** | Artefacts are 3–5 weeks stale and materially wrong |
| 17 | If an **R8** S3 ships: keep TP3/4/5 stubs <2 mm or DNP-jumper them + silkscreen the restriction | S31 and S3 N/R2 are unaffected |
| 20 | **Bench-measure `USB_STAT` unplugged** — expect 1.98 V | ~0.93 V means TP4056 ESD backfeed; its datasheet doesn't rate those pins |

**Optional:**

| # | Change | Note |
|---|---|---|
| 28 | `R20` 56 k→39 k, `R36` 68 k→47 k | Doubles top-of-ladder margin vs the 3.1 V ADC ceiling. Ladders already pass worst-case as-is |

---

### 🚫 Declined — your call, noted and reasonable

| # | Item | Rationale accepted |
|---|---|---|
| 8 | 100 Ω on DW01A `VCC` | Prior board failure makes the risk not worth it. (For the record: only 3 µA flows there → 0.3 mV, so the old failure was a placement issue — but skipping is fine) |
| 9 | Hardware pull-downs on `UNUSED_GPIO_45/46` | Software pull-downs; a header pin won't realistically flip a strap |
| 19 | Dedicated VBUS-present GPIO | Sticking with the resistor ladder — makes item 20 more important |
| 29 | Delete `R45`/`R58` | Kept intentionally for the alternate panel config |

### ❌ Withdrawn — my errors, no action needed

| # | Item | Why it was wrong |
|---|---|---|
| 13 | `L2` inductor | Peak current is 23 % of I_LIM, not near it — and `74479325207247` I_sat 2.7 A confirms 6.6× margin ✅ |
| 16 | `CR1–CR3` footprint | TI `DYF` **is** SOD-323; your footprint is correct |
| 22 | `EPD_CS` pull-up | Glitch is a *driven static low* — no SCK edges, so nothing is clocked; a pull-up can't override 20 mA anyway |
| 23 | `SW` net-class clearance | I quoted IPC column B2 (uncoated); under solder mask it's **B4 = 0.13 mm**, so 0.15 mm passes. *Residual:* tent vias on `/PREVGH` + `/PREVGL`, and don't route them adjacent |
| 26 | `R14` power rating | I used peak instead of RMS — actual is 12–28 mW of 125 mW |
| 30 | `R14` written as `3R` | Your bare-number-equals-ohms convention is consistent |
| 2.4 | e-paper `VGL` cap | Pin 4 is NC; the symbol label was wrong |
| 6.13 | `N16R8` incompatibility | The **S31** exists and frees IO35–37; my model was out of date |

---

## 6. Block-by-block detail pass

Exhaustive nit-pick of every block, with the numbers behind each call. Items already covered above are not repeated.

### 6.1 ~~Net classes — the ±22 V e-paper rails are on `Default`~~ — **mostly WITHDRAWN**

I quoted the **wrong IPC-2221 column**. For external conductors under solder mask the applicable condition is **B4 (permanent polymer coating)**, not B2 (uncoated):

| IPC-2221 condition, 31–50 V band | Minimum |
|---|---|
| B1 — internal conductors | 0.10 mm |
| B2 — external, **uncoated** | 0.60 mm ← what I wrongly quoted |
| **B4 — external, permanent polymer coating (solder mask)** | **0.13 mm** |

Your `Default` class at **0.15 mm** is therefore **compliant** for the 44 V `PREVGH`↔`PREVGL` differential, with ~15 % margin. Don't fight your 0.5 mm connector fanout over this.

**Two real caveats remain (🟡):**

1. **B4 only applies where the gap is actually mask-covered.** At exposed copper — component pads, test points, and **untented vias** — the condition reverts to **B2 = 0.60 mm** (or A6 for component terminations). This is the classic trap: the trace run passes easily, but the pad-to-pad gap at a connector or an untented via on `PREVGH`/`PREVGL` does not. **Check the exposed features on those two nets specifically**, and tent the vias.
2. **Simplest mitigation costs nothing:** just don't route `/PREVGH` and `/PREVGL` adjacent to each other. The 44 V case only exists where those two specific traces run in parallel; everything else on the panel is a ≤25 V single-ended net where 0.15 mm has ample margin.

The existing `SW` class (0.3 mm on `LED_SW`/`EINK_SW`/`3031_SW`) is fine as-is.

*(Verified against a published reproduction of IPC-2221B Table 6-1 columns B1/B2/B4; the standard itself is paywalled. Note IPC-2221**C** is current as of Dec 2023, though the ≤500 V values have reportedly been unchanged since 1998 — confirm against your controlling revision if it matters contractually.)*

### 6.2 `J6` is a **2×6 dual-row** header — corrected analysis

**Two corrections to my earlier claim.**

**(a) `J6` is dual-row, not a single row of 12.** Confirmed from the PCB footprint (`CONN_PPPC062LJBN-RC_SUL`), pads on a 2.54 mm grid:

```
row 1 (y = 0.00) :  1 GND   2 IO46  3 IO45  4 SDA   5 LED_SW  6 W-
row 2 (y = -2.54):  7 3V3   8 P+    9 IO3  10 SCL  11 C-     12 GND
```

So neighbours are horizontal (n↔n+1 within a row), vertical (1↔7, 2↔8, … 6↔12), and diagonal at 3.59 mm.

**(b) `W−` and `C−` are NOT high-voltage nets — I was wrong.** `R50` (1 M, W−→GND) and `R49` (1 M, C−→GND) are bleeders that hold both cathode nets near ground. In normal operation one FET is always on (the inverter guarantees it), so the conducting net sits at the FB voltage of **200 mV** and the other is pulled to ~0 V by its 1 M. They only rise if the boost runs with *both* `Q5` and `Q6` off — which is precisely the `COLOR_SEL` float condition from §2.5, and even then they reach only `V_OVP − V_f(string)` ≈ 25 − 15 = **~10 V**, not 24 V.

**So there is exactly one genuinely dangerous adjacency, not three:**

| Pair | Type | Severity |
|---|---|---|
| **pin 4 `SDA` ↔ pin 5 `LED_SW`** | horizontal, 2.54 mm | 🔴 24.5 V one pin from the I²C bus |
| pin 7 `3V3` ↔ pin 8 `P+` | horizontal | 🟡 4.2 V onto a 3.3 V rail (above the ESP32's 3.6 V max) |
| pin 2 `IO46` ↔ pin 8 `P+` | vertical | 🟡 4.2 V onto a GPIO |
| pin 10 `SCL` ↔ pin 11 `C−` | horizontal | 🟢 C− is a ~0 V net — **not** a problem |

**Minimal fix — one swap: `SDA` (pin 4) ↔ `GND` (pin 12).**

```
row 1:  1 GND   2 IO46  3 IO45  4 GND   5 LED_SW  6 W-
row 2:  7 3V3   8 P+    9 IO3  10 SCL  11 C-     12 SDA
```

`LED_SW` (pin 5) is then bounded by `GND` (4), `W−` (6) and `C−` (11) — every neighbour is a LED-domain or ground net, and no logic pin touches it. This is a one-line change that kills the only 🔴.

**Fuller fix, if you can afford to break daughterboard compatibility:**

```
row 1:  1 GND   2 IO46  3 IO45  4 GND   5 LED_SW  6 W-
row 2:  7 3V3   8 SDA   9 IO3  10 SCL  11 C-     12 P+
```

This additionally moves `P+` to the corner where its only neighbours are `C−` and `W−` — both clamped by the FB node and the 1 M bleeders, and both well under the 5.5 V that `U10`'s FB pin tolerates. `3V3` now neighbours only `GND` and `SDA`.

**Honest constraint:** with 12 positions, 3 LED-domain nets, a raw cell and 5 logic nets, you cannot fully isolate everything — there are only two GND pins to spend as guards. If you ever revise the connector, **dropping `P+` from the header** (give a daughterboard `3V3` instead) makes the whole problem disappear and frees a third GND.

### 6.3 ⬜ **OUTSTANDING** — high-voltage capacitor ratings still unspecified

`C13`–`C17` (4.7 µF) and `C20` (1 µF) sit on panel rails of **±15 V to ±22 V**, and every one is an 0805 with **no voltage rating anywhere in the schematic, the BOM, or the value field**. Only `C9` carries a "50V" annotation.

This is the classic e-paper failure: a 4.7 µF 0805 X5R is commonly a **16 V or 25 V** part. At 22 V on a 25 V part you lose 70–80 % of capacitance to DC bias (so 4.7 µF behaves like ~1 µF) *and* you have almost no derating margin; on a 16 V part you are over-rated and it will fail short.

**Fix:** specify **≥50 V** (ideally 100 V for the ±22 V rails) in the value field or a `Voltage` property for `C13, C14, C15, C16, C17, C20`, and re-check the capacitance you actually get after DC-bias derating at the real rail voltage. Add voltage ratings to *every* cap while you're there — the BOM currently has none.

### 6.4 ~~`EPD_CS` has no pull-up~~ — **WITHDRAWN, my reasoning was wrong**

I claimed a power-up glitch on `IO14` could clock garbage into the panel, and that a pull-up would prevent it. Having read the actual table, **both halves of that are wrong.**

ESP32-S3 datasheet v2.2 **Table 2-2 "Power-Up Glitches on Pins"** specifies, for GPIO1–14 and GPIO17: *"Low-level glitch — Typical Time Period 60 µs"*, with footnote 1: *"Low-level glitch: **the pin is at a low level output status** during the time period."*

Two consequences:

1. **It is a driven output, not high-Z.** A 10 k pull-up cannot hold CS high against a 20 mA driver (datasheet: *"The default drive strengths … All other pins: 20 mA"*). The pull-up would simply be overpowered — it does not do the job I claimed.
2. **More importantly, the glitch is a static level, not a pulse train.** `IO12` (MOSI), `IO13` (SCK) and `IO14` (CS) all sit statically low for the same 60 µs. SPI shifts data on **clock edges**; a static low SCK produces none. So the panel latches nothing. The failure mode I described cannot occur.

`EPD_RST`'s pull-up (`R5`) is *not* an inconsistency either — `EPD_RST` is on `IO47`, which is **not** in the glitch table; `R5` holds the panel out of reset during the high-Z window before firmware runs, which is a different and valid job.

**Residual (🟢, optional):** a pull-up on `EPD_CS` would still define the line during the high-Z interval between reset release and firmware configuring the pin. Cheap insurance, but not the risk I originally described. Your call.

*(Worth knowing for other reasons: GPIO18/19/20 glitch **high** as well as low, and GPIO19/20 do so twice over a 3.2 ms / 2 ms window. `IO18` is your `PWR_BUTTON` — so it is driven both high and low for ~60 µs at power-up regardless of any pull resistor. Don't sample the button until well after boot.)*

### 6.5 ⬜ *Optional* — ESP32-S3 ADC ceiling: top of both ladders is closer than it looks

The ESP32-S3 SAR with `ADC_ATTEN_DB_12` has a usable range of **0–3100 mV**; anything above that reads full-scale. Both ladders idle at **3.300 V**, i.e. *saturated*.

That's actually fine for "no button pressed" — saturation is an unambiguous idle indicator. But it compresses the top margin: the real gap is not (idle_min − top_button_max) but (**3.100 V ceiling** − top_button_max):

| Ladder | Top button | worst-case max | gap to 3.1 V ceiling | minus ±30 mV ADC error |
|---|---|---:|---:|---:|
| 1 | SW9 / BACK (56 k) | 2.865 V | 235 mV | 175 mV |
| 2 | SW7 / UP2 (68 k) | 2.942 V | **158 mV** | **98 mV** |

98 mV is workable but it is the tightest margin in the whole button system, and it sits in the region where the S3 SAR is least linear. The same applies to `USB_STAT` state 4 (3.30 V) — your ">3.10 V" window works *only because* it coincides with saturation. Document that; don't let someone "fix" it later by switching attenuation.

**Optional fix:** drop `R20` 56 k → 39 k and `R36` 68 k → 47 k. That moves the top buttons to 2.61 V / 2.72 V, roughly doubling the ceiling margin while keeping all the lower gaps comfortable.

### 6.6 Button ladders — full worst-case tolerance analysis (they pass)

Simulated over 1 % resistors and ±2 % rail, plus ±30 mV ADC error:

**Ladder 1** (`R4` 10 k pull-up): RIGHT 0.033 V · LEFT 1.185 V · CONFIRM 2.200 V · BACK 2.800 V · idle 3.300 V.
Worst-case adjacent gaps: **+1052, +858, +417, +309 mV** — all comfortable. ✔

**Ladder 2** (`R28` 10 k): DOWN1 0.033 V · UP1 1.800 V · DOWN2 2.533 V · UP2 2.877 V · idle 3.300 V.
Worst-case gaps: **+1654, +558, +157, +232 mV**. The **DOWN2→UP2 gap of 157 mV is the tightest in the design** but still passes. ✔

**Chords are not decodable**, confirming the single-press-only constraint:
* The 100 Ω buttons (`SW2`/RIGHT, `SW1`/DOWN1) **mask everything** — any chord including them reads as that button alone (Δ < 1 mV). That's a usable *priority* property if you document it.
* `LEFT + BACK` = 1.113 V vs `LEFT` alone 1.185 V — **only 71 mV apart**, inside worst-case tolerance. Genuinely ambiguous.
* `UP1 + UP2` = 1.666 V vs `UP1` 1.800 V — 134 mV, marginal.

Source impedance is **8.5–10 kΩ**, right at the ESP32's ~10 kΩ guideline; `C27`/`C28` = 2.2 nF hold the SAR sampling cap adequately (≈44× the ~50 pF S/H cap). The RC is 22 µs on release / 0.22 µs on press — that is **anti-aliasing, not debounce**; mechanical bounce is 1–10 ms and must be handled in firmware.

### 6.7 ~~`LED_MONIT` filter is 10× too small~~ — ✅ FIXED (`C31` = 100 nF)

`R39` 1 M / `R41` 120 k → ratio 0.1071, `Z_src` = **107 kΩ**, with only `C31` = **10 nF**.

* Full-scale check: at the 25.5 V OVP max, ADC sees 2.73 V — inside range ✔ (a true 30 V rail would give 3.21 V and **clip**, another reason the 610-vs-611 distinction in §2.2 matters).
* But `τ` = 107 k × 10 nF = **1.07 ms**, and 10 nF is only ~200× the SAR sample cap. After an ADC mux change the node needs several milliseconds to settle.

**Fix:** `C31` 10 nF → **100 nF**. Same reasoning as `BAT_MONIT`, which already sensibly uses 1 µF against its 500 kΩ source.

### 6.8 ~~I²C pull-ups too weak for Fast Mode~~ — ✅ FIXED (`R47`/`R48` = 2.2 k)

`R47`/`R48` = 4.7 k to 3V3. Rise time `t_r ≈ 0.847 × R × C_b`:

| Bus capacitance | t_r | Fast Mode (300 ns) | Std Mode (1000 ns) |
|---|---:|---|---|
| 50 pF (on-board only) | 199 ns | ✔ | ✔ |
| 150 pF (+ touch FFC + header) | **597 ns** | ❌ | ✔ |
| 400 pF (I²C max) | 1593 ns | ❌ | ❌ |

With the touch panel on a flex cable *and* the bus exposed on `J6`, 100–200 pF is realistic. **At 400 kHz you will violate rise time.** Sink current is only 0.70 mA (spec allows 3 mA), so there is lots of room to go stronger.

**Fix:** `R47`/`R48` → **2.2 k** (t_r = 280 ns at 150 pF, 1.5 mA sink) if you want Fast Mode; keep 4.7 k only if you commit to 100 kHz. Note `IO39`/MTCK also has a default internal weak pull-up (~45 k) — harmless, but don't count on it.

### 6.9 SDIO — correct, one nit

33 Ω series (`R21`–`R26`) with ~30–40 Ω ESP32 output impedance gives ~65–73 Ω source against a ~60 Ω trace — **slightly over-damped, which is exactly right** for EMC. 10 k pull-ups on DAT0–3 + CMD ✔; **CLK correctly has no pull-up** ✔. RC edge (33 Ω × ~15 pF ≈ 1 ns) is negligible against a 38 ns bit period at 26 MHz. ✔

**Nit:** `J7.4 (VDD)` is hard-wired to `3V3` with no power switch. An idle-but-powered microSD draws **0.2–1 mA** — that is 2–10× your entire deep-sleep budget. If shelf life matters, put the card's VDD on a load switch (or a GPIO-driven P-FET) so deep sleep can actually remove it. Also confirm `C36`/`C37` (0.1 µF + 4.7 µF on 3V3) are physically at the socket — the card needs local bulk for write bursts.

### 6.10 E-paper charge pump — topology verified correct

Traced completely: `3V3 → L1 22 µH → EINK_SW → Q4(BSS138) → RESE → R14 3 Ω → GND`, with `EINK_SW —D5→ PREVGH` (boost leg) and `EINK_SW —C11 4.7 µF→ node`, `node —D6→ GND`, `node —D4→ PREVGL` (inverting charge pump). The panel's own `GDR` drives `Q4`'s gate and `RESE` is its current-sense return — this is the standard panel-driven arrangement. ✔

* `R14` = 3 Ω sense: at 200 mA peak, 600 mV sense, **120 mW in an 0805 rated 125 mW** — that's **96 % of rating**. Tight. Check your panel's actual peak `GDR` current; if it exceeds ~200 mA, move `R14` to a 1206 or use two 6 Ω in parallel.
* `R15` = 10 k pull-down on `GDR` ✔ correct (holds `Q4` off when the panel is unpowered/high-Z).
* Diode reverse-voltage margins on the B5819W (40 V): D5 ~22 V (1.8×), D6 ~22 V (1.8×), D4 ~20 V (2.0×). Acceptable, but B5819W leakage is specified as **1 mA max at 40 V** — at 22 V and elevated temperature this becomes a real load on rails that the pump can only weakly source. If you see rail droop on long refreshes, a lower-leakage 60 V Schottky is the fix.
* `Q4` is a BSS138 (50 V, ~0.22 Ω): fine for a 22 V swing, but only ~2× voltage margin.

### 6.11 TP4056 thermal — plenty of margin

`(V_BUS − V_BAT) × I_chg` in ESOP-8 with EPAD to GND and thermal vias (`R_θJA` ≈ 40 °C/W):
5.0 V → 3.0 V cell at 100 mA = 200 mW = **+8 °C**; at 4.2 V = 80 mW = **+3 °C**. Trivial. You have room to raise the charge current later (`R6` 12 k → 100 mA; 4.7 k would give ~255 mA and still only ~+20 °C). ✔

### 6.12 Boost output ripple with `C9`

At 24.5 V out, 15 mA, D = 0.865: `V_ripple = I×D/(C×f)`.
Nominal 1 µF → 11.8 mV. **Derated to ~0.35 µF (1 µF 50 V 0805 at 20 V bias) → 34 mV.** With 2.2 µF (≈0.8 µF derated) → 15 mV. Confirms item 11 — not dangerous, but the derated part gives ~3× the intended ripple, which modulates LED current and shows up on `LED_MONIT`.

### 6.13 ESP32-S3 variant vs `TP3`/`TP4`/`TP5` — conditional, not absolute

*(Corrected: the **ESP32-S31-WROOM-1** is a real part, pin-compatible with the S3, whose PSRAM does **not** occupy IO35–IO37. My model predates it. The test points are valid future-proofing for that module.)*

The constraint applies only to what is **actually fitted**. For ESP32-S3 modules, datasheet v1.8 Table 3-1 footnote b is explicit:

> *"For modules with Octal SPI PSRAM … pins **IO35, IO36, and IO37 are connected to the Octal SPI PSRAM and are not available for other uses**."*

| Fitted module | IO35/36/37 | TP3/4/5 status |
|---|---|---|
| S3 **N4/N8/N16** (no PSRAM) | free | OK — fully usable |
| S3 **N8R2/N16R2** (2 MB *quad*) | free | OK — fully usable |
| S3 **N8R8/N16R8** (8 MB *octal*) | PSRAM DQ6/DQ7/DQS | **do not connect or probe** |
| **S31** | free | OK — usable |

So the pads are sound future-proofing. The only live risk is an **R8** build, where they become unterminated stubs on a DDR bus at 80–120 MHz. Manageable rather than fatal, provided you:

1. **Keep the stub microscopic** — TP pad immediately against the module pad, no routed trace. 1–2 mm adds ~1 pF and is harmless; 10 mm across the board is not.
2. **Better: DNP-jumper them.** A `0 Ω` (DNP) between each module pin and its TP pad means an R8 build has *no* stub at all, and an S31 build just gets three jumpers fitted. This gives you both cleanly.
3. **Mark them** — silkscreen/schematic note: *"IO35–37: do not connect on octal-PSRAM (R8) modules."* Exactly the kind of constraint that gets lost between revisions.

Pin the default S3 variant in the BOM. If it is `N16R8`, item 1 or 2 is mandatory; if it is `N16R2`, nothing to do.

### 6.14 ✅ **`Q7` freed and reused — DONE.** LED boost now runs from `LDO_IN`; `Q7` became the SD load switch

TPS923610 datasheet §6.5, verbatim:

> *"I_SD Shutdown current into VIN pin | ADIM = 0 (IC disabled), TJ = 25 °C | typ **0.13** | max **0.25** µA"* (max 0.5 µA to 85 °C), versus *"I_Q Quiescent current into VIN pin | Chip enable. No switching. | typ 260 | max 300 µA"*. Front page: *"130nA ultra-low shutdown current."*

**130 nA typ / 500 nA worst case is less than the leakage of the load-switch FET you'd use to gate it.** Power-gating `U10` with `Q7` buys nothing.

So: **delete `Q7` and `R40`, and connect `U10.VIN` directly to `LDO_IN`.** Control the driver entirely through `ADIM` (hold low ≥ `t_ADIM_SD` = 2.5 ms to enter shutdown; the pin self-parks low via its internal 600 kΩ pull-down, so it is safe at boot with no external resistor).

This resolves four open items simultaneously:

1. **§3.2 LDO thermal** — removes the 131 mA LED load from the AP2112K entirely.
2. **Your 7b concern** — with no `Q7`, nothing pulls a gate to `LDO_IN`, so no ESP32 pin is ever exposed above 3.3 V. `ADIM` is rated to **5.5 V abs max** and needs only **1.2 V** for a logic high, so a 3.3 V GPIO drives it correctly even with `U10` running from a 5 V `LDO_IN`.
3. **Eliminates the double conversion** — battery → LDO → boost becomes battery → boost.
4. **Frees the load switch for the microSD** (your condition on should-fix #10) — a net-zero part count change.

**One caveat to be aware of:** a boost converter always has a DC path from VIN to VOUT through the inductor and the high-side FET's body diode, so with `U10` disabled `LED_SW` will sit at roughly `LDO_IN` (≈4–5 V) rather than 0 V. That is far below the LED string's ~15 V forward voltage, so no current flows and nothing lights — but if you ever need `LED_SW` genuinely dead (e.g. for hot-plugging the LED strip), the load switch is the only way to break that path. Worth knowing before you delete `Q7`.

### 6.15 ✅ **microSD power gating — COMPLETE and verified**

Your proposed circuit (P-FET high-side switch, source = 3V3, gate = `SD_ACTIVATE` with 100 k pull-up, drain = `J7.4 VDD`) is **electrically correct**, and it is the right reuse of the `Q7` pattern:

* `AO3419` at `V_GS` = −3.3 V is fully enhanced; at ~200 mA peak card current and ~50 mΩ that is a **10 mV** drop. ✔
* Body-diode orientation is safe: for a P-channel the diode runs drain→source, so it only conducts if `SD_VDD` > 3V3, which cannot happen. **No back-feed.** ✔
* Gate pull-up to 3V3 means the default state (GPIO high-Z at boot) is **OFF** — the card is unpowered until firmware asks for it. That is the right default. ✔

**But there is a trap that will stop it working — every SD pull-up goes to the *unswitched* rail:**

| Resistor | From | To |
|---|---|---|
| `R8` 10 k | `SD_DAT0` | **`3V3`** |
| `R9` 10 k | `SD_DAT1` | **`3V3`** |
| `R53` 10 k | `SD_DAT2` | **`3V3`** |
| `R54` 10 k | `SD_DAT3` | **`3V3`** |
| `R55` 10 k | `SD_CMD` | **`3V3`** |
| `C36` 0.1 µ / `C37` 4.7 µ | decoupling | **`3V3`** |

With `SD_VDD` gated off, those five 10 k resistors still hold the card's DAT/CMD pins at 3.3 V while its VDD pin is at 0 V. Current flows in through the card's input ESD structures and **phantom-powers it**: the card never fully powers down (so you lose most of the saving you're gating for), it can latch into an undefined state, and sustained reverse injection is a known way to damage cards.

**Required changes to make gating actually work — ✅ ALL DONE**

1. ✅ **`R8`, `R9`, `R53`, `R54`, `R55` moved from `3V3` to the switched `SD_VDD`.** The phantom-power trap is closed.
2. ✅ **`C36`/`C37` moved to `SD_VDD`**, and `C37` reduced 4.7 µF → **1 µF**.
3. ✅ **Bleed: `R77` = 100 kΩ, `SD_VDD` → GND.**
4. ⬜ *(firmware)* **Drive the SD GPIOs low before gating off** — see the discharge note below.
5. ✅ **Gate: `R78` = 1 kΩ in series from `IO41`, with `R40` 100 kΩ pull-up on the FET side of it.** The `R40` placement is correct and worth noting — putting it on the gate side of `R78` means the gate reaches a true 3.30 V when the GPIO releases, so the FET is fully off.

**As-built verification:**

| Item | Value | Result |
|---|---|---|
| `V_GS` when on | 3.3 × 1k/101k = 0.033 V | **−3.27 V — full enhancement** ✔ |
| `V_GS` when off | gate at 3.30 V | 0 V — fully off ✔ |
| Gate RC | 1 k × ~900 pF | 0.9 µs — GPIO edge protection, no soft-start (by choice) |
| `SD_VDD` load | C36 0.1 µ + C37 1 µ + card ~1 µ | **2.1 µF** (was 5.8 µF) |
| Inrush charge | 2.1 µF × 3.3 V | 6.93 µC |
| **3V3 dip on card power-up** | 6.93 µC into 51 µF of bulk | **136 mV** (3.30 → 3.16 V) |
| Bleed idle current | 3.3 V / 100 k | 33 µA while card is on |

**The `C37` reduction did the job the gate capacitor would have.** Dropping 4.7 µF → 1 µF cut the inrush charge from 19.1 µC to 6.93 µC, which takes the worst-case 3V3 dip from **375 mV down to 136 mV** (3.30 → 3.16 V). Against the ESP32-S3's ~2.51 V brown-out that is comfortable, and it is small enough not to disturb the ADC references or the RTC. No `C_gs` needed — correct call.

**Discharge behaviour:**

| Path | τ | to 0.5 V |
|---|---:|---:|
| `R77` 100 k alone | 210 ms | **396 ms** |
| + SD lines driven low (five 10 k ≈ 2 kΩ) | 4.2 ms | **7.9 ms** |

So a firmware power-cycle should either wait **~400 ms** after gating off, or drive the SD lines low first and wait ~10 ms. The smaller `C37` also halved the wait versus the original 4.7 µF.

**One residual trade-off to be aware of:** 1 µF is on the low side of typical microSD local bulk. Cards can pull 50–100 mA bursts during writes, and `Q7` (~45 mΩ) plus the trace sits between `SD_VDD` and the 3V3 bulk. If you see write errors or CRC retries under load, restoring `C37` to 2.2 µF is the first thing to try — the dip would only rise to ~210 mV, still fine.

The `TPD4E1U06` arrays (`U1`, `U9`) are *not* a problem — they are supply-less bidirectional TVS to GND, so they cannot back-feed. ✔

**Which GPIO to use — use `IO41` (the freed `LED_ACTIVATE`).** If you delete `Q7` from the LED boost per §6.14, you can move that entire circuit — `Q7` + `R40` — to the SD card and rename the net. `IO41` is ideal because:

* It is **not** a strapping pin on the S3, and **not** in the power-up-glitch table (that is GPIO1–14, 17–20).
* It comes out of reset as `IE` only — **no internal pull** — so your 100 k pull-up wins uncontested and the FET is cleanly off at boot.

**Do not use `IO45` or `IO46` for this.** Both have an internal **weak pull-down (~45 k) enabled at reset**, which would fight the 100 k gate pull-up: `3.3 × 45/145` = **1.02 V** on the gate → `V_GS` = −2.28 V → the AO3419 would be **partially on at every boot**, powering the card through a half-enhanced FET. `IO3` is also poor here: it is in the glitch table and would drive the gate low for 60 µs, briefly powering the card at every startup.

**Alternative worth pricing:** a dedicated load switch (e.g. TPS22918, SOT-23-6) gives you controlled slew rate *and* an integrated output discharge in one part, replacing the FET + pull-up + gate resistor + bleed resistor. Similar cost, fewer things to get wrong.

### 6.16 🟢 **NEW (from the documentation pass)** — `LED_MONIT` does not read 0 V when the frontlight is off

Not a defect — a firmware-facing behaviour worth writing down, found while documenting §7 of [`docs/HARDWARE.md`](docs/HARDWARE.md).

**Confirmed safe for the LED strip.** With the panel frontlight at V_f ≈ 15 V (~5 white LEDs
in series), an idle `LED_SW` of at most 4.3 V puts **~0.86 V across each junction** — roughly a
third of the ~2.5 V needed to begin conducting. Sub-threshold current is in the **picoamp**
range. This holds on both return paths: the *active* string (13.3 Ω via its FET) and the
*inactive* one (1 MΩ bleed). **No glow, no leakage, no wasted energy.** `C9` simply pre-charges
to that level, which slightly *reduces* boost start-up work.

A boost converter always has a DC path from input to output: `LDO_IN → L2 → SW → HS-FET body diode → VOUT`. With `U10` disabled (`ADIM` low), `LED_SW` therefore settles at roughly `LDO_IN − 0.7 V`, **not** 0 V, and the `R39`/`R41` divider reports it:

| `LDO_IN` | `LED_SW` (off) | `LED_MONIT` reads | divider draw |
|---:|---:|---:|---:|
| 5.0 V (USB) | ~4.30 V | **0.461 V** | 3.8 µA |
| 4.2 V (full cell) | ~3.50 V | **0.375 V** | 3.1 µA |
| 3.7 V (nominal) | ~3.00 V | **0.321 V** | 2.7 µA |
| 3.0 V (empty) | ~2.30 V | **0.246 V** | 2.1 µA |

Three consequences:

1. **Firmware must not treat ~0.25–0.46 V as a fault.** That *is* the healthy off state. A "boost failed to start" check should look for a reading that stays low after `ADIM` has been high for >40 µs, not for a low reading per se. Note the off-state value tracks the battery, so the threshold should be relative to `BAT_MONIT`, not absolute.
2. **The divider draws 2–4 µA continuously**, even with the frontlight off, because `LED_SW` is never actually at 0 V. That is a real (small) contributor to the standby budget, and it cannot be removed without gating the boost input.
3. **No glow.** The LED string sees at most ~4.3 V against a ~15 V forward voltage, so nothing lights. ✔ (This is also why the string can be safely left connected.)

If the standby 2–4 µA ever matters, the fix is the same load switch discussed in §6.14 — but at 2–4 µA against a ~95 µA budget it is not currently worth a part.

### 6.17 🟡 **NEW** — `C9` = 4.7 µF may limit the CCT blend rate

Now that CCT blending is confirmed as an intended feature, `C9` needs a second look — and this
partly **undoes my own earlier advice**.

I recommended raising `C9` (1 µF → 4.7 µF) in §3.3 to counter DC-bias derating and reduce
output ripple. That reasoning assumed a static output. With `COLOR_SEL` time-multiplexing, `C9`
also has to **re-slew between the two strings' forward voltages on every colour transition** —
and the boost can only *discharge* it through the LED current:

```
t_fall = C9 × ΔV_f / I_LED = 4.7 µF × ΔV_f / 15 mA
```

| ΔV_f (warm vs cool) | discharge time | `COLOR_SEL` ceiling (transient < 10 % of half-period) |
|---:|---:|---:|
| 0.1 V | 31 µs | ~1.6 kHz ✔ |
| 0.2 V | 63 µs | ~800 Hz ⚠ |
| 0.5 V | 157 µs | ~320 Hz ❌ flicker range |
| 1.0 V | 313 µs | ~160 Hz ❌ flicker range |

**This is the binding constraint on blend rate** — far more restrictive than the boost's own
7–15 µs loop settling. IEEE 1789-2015 wants > 1.25 kHz for "low risk" flicker at high
modulation depth, which is exactly the regime here (each string goes fully off).

**Mitigating factors:** DC-bias derating works *for* you — a 4.7 µF/50 V 0805 at 15 V bias
delivers perhaps 2–3 µF effective, so real slew times are roughly half the table above. And for
a bonded frontlight, warm and cool are usually the same die with different phosphor, so ΔV_f
should be small.

**Action: measure ΔV_f between the two strings before finalising.**

* **ΔV_f < 0.2 V** → keep `C9` = 4.7 µF, run `COLOR_SEL` at 1.25 kHz. Nothing to do.
* **ΔV_f > 0.5 V** → **reduce `C9` back toward 1–2.2 µF.** Ripple rises to ~30 mV, which is
  invisible on a frontlight; visible flicker is not. Do *not* solve this by slowing
  `COLOR_SEL` below ~1 kHz.

There is a real trade here and it should be resolved with a measurement rather than a default.

### 6.18 🟡 **NEW** — what an ESP32-S31-WROOM-1 swap would actually require

Verified against the **ESP32-S31-WROOM-1 Datasheet, Pre-release v0.1** (Table 3-1 pin
definitions, Table 4-3 boot mode, §5.2.2.10 SDHOST, §5.2.3.3 ADC, Figure 9-1 dimensions).

**Correction to an earlier draft of this review:** I previously stated the S31-WROOM-1 was not
footprint-compatible, based on the published *WROOM-3* datasheet (99 pins) and Espressif's
product table. **That was wrong.** The S31-WROOM-1 is **18.0 × 25.5 × 3.1 mm with 40
castellated pads on a 1.27 mm pitch** — the same main-pin layout as the S3-WROOM-1. Power
(1, 2), `EN` (3), USB (13/14) and UART0 (36/37) all land on identical pins.

#### The blocker: ADC1 pin sets are disjoint

**ADC2 is unusable while Wi-Fi is active on both parts**, so only ADC1 counts. Mapping each
part's ADC1 GPIOs onto physical module pins:

| Part | ADC1 GPIOs | Physical perimeter pins |
|---|---|---|
| ESP32-S3 | GPIO1–GPIO10 | **4, 5, 6, 7, 12, 15, 17, 18, 38, 39** |
| ESP32-S31 | GPIO42–GPIO49 | **28, 29, 30, 31, 32, 33** (+ inner pads 56, 57) |

**The intersection is empty.** There is not one physical pin that offers ADC1 on both parts, so
**no single pin assignment can serve both** — this is a hard result, not a tuning problem.

Good news on the S31 side taken alone: §5.2.3.3 says each SAR ADC measures *"analog signals
from up to eight pins"*, so the `_N`/`_P` suffixes denote optional differential pairing, not a
requirement. Six single-ended ADC1 inputs on pins 28–33 comfortably covers this board's five
analog nets. An **S31-only** board is perfectly feasible.

But pins 28–33 currently carry `TP5`/`TP4`/`TP3`, `I2C_SDA`, `I2C_SCL` and `PWM_LED` — and on
an S3 those pins are `IO35`–`IO40`, none of which are ADC-capable (and `IO35/36/37` are the
octal-PSRAM pins on `R8` parts). So the analog cannot simply be moved there and left alone.

Supporting both parts would mean bringing all five analog nets to **two** pins each with 0 Ω
select jumpers — ten resistors, plus dual routing for the five signals displaced from pins
28–33 — on a board whose stated goal is to be cheap and easy to understand. **Not worth it.**

#### The SD bus would also need moving

§5.2.2.10: *"For the SD/SDIO/MMC host controller, card one can use GPIO20–GPIO25 via IO MUX,
and card two can use GPIO35–GPIO40 via IO MUX."* Note the datasheet mentions **only IO MUX**
here, in contrast to UART (§5.2.2.1: *"They also support mapping to other pins through the GPIO
Matrix"*) and UART1-3 (*"routed to any HP GPIO pins via the GPIO Matrix"*). SDHOST should
therefore be treated as **fixed to those two pin groups**, not freely mux-able.

On the module that means card 1 = pins 15–20 and card 2 = pins 12, 21–25. This board's SD bus
is on pins 5–10, which is neither group. (Ironically, pins 15–20 currently carry
`UNUSED_GPIO_3/46`, `LED_MONIT`, `TP_INT`, `TP_RST` and `SPI_MOSI` — an S31-targeted layout
would put the SD bus exactly there.)

#### Other differences, for completeness

* **Centre pad.** `U4`'s footprint has pad 41 as a single **3.90 × 3.90 mm GND** pad. The S31
  puts **20 signal pads (0.4 × 0.8 mm) + 9 GND pads (0.9 × 0.9 mm)** there, carrying `IO8`–
  `IO19`, `DM`/`DP` and `IO48`–`IO53`. Soldering an S31 to the unmodified footprint risks
  shorting those to ground.
* **Four strapping pins move**: `IO36` (pin 21, `SPI_SCK`, VDD_SPI strap, internally pulled up),
  `IO37` (pin 22, `EPD_CS`, JTAG strap, defaults floating and must be driven), `IO60` (pin 26,
  boot strap, exposed on `J6`), `IO61` (pin 27). The boot button survives by luck — download
  boot needs `GPIO61 = 0, GPIO60 = 1`, and `SW6` plus `IO60`'s internal pull-up gives exactly
  that.

#### Recommendation: drop S31 compatibility as a goal

The pin maps diverge too far. **Keep `TP3`/`TP4`/`TP5`** — they remain genuinely useful on
non-octal S3 variants (`N4`/`N8`/`N16`/`R2`), which is a real and immediate benefit.

**Reword the schematic note**, though. It currently reads *"ESP32-S31-WROOM-1 does not reserve
GPIO for PSRAM. Future proofing :)"*, which implies a drop-in path that does not exist. Better:
*"IO35–37: free GPIO on non-octal-PSRAM S3 variants (N4/N8/N16/R2); do not connect on R8."*

If the S31 ever becomes compelling, it is a **new board revision** — and a clean one, since the
S31's own pin grouping is quite favourable (SD on pins 15–20, analog on 28–33).

### 6.19 Smaller observations

* **`R14` (3 Ω, 0805) — concern WITHDRAWN.** My 120 mW figure assumed 200 mA *continuously*; `R14` only conducts while `Q4` is on, and the current is a rising triangle. `I_rms = I_pk·√(D/3)` gives 63–97 mA over D = 0.3–0.7, so **P = 12–28 mW**, i.e. 10–22 % of an 0805's 125 mW. Comfortable. No action.
* **`C7` (0.1 µF, P+ → B−)** is correctly the DW01A's datasheet `C1` (VCC-to-GND, where the IC's GND *is* B−). ✔ This is right and easy to mistake for an error — worth a schematic note so a future reviewer doesn't "fix" it.
* **`C24` = 100 n** and `C30`/`C33`/`C36` = 0.1 µ are the same value written two ways. Normalise to one notation.
* **`Net-(D2-K)` / power LED:** `R59` = 2 k from `USB_VBUS` → 1.5 mA at V_f = 2.0 V. The note says "2.5 mA" but the formula written beside it, `(5 − 2.0)/0.0015`, is the 1.5 mA one. Fix the label. 1.5 mA is a reasonable choice.
* **`U13` `VCC` → GND is correct** (VBAT-only mode) but there is **no dedicated decoupling on `VBAT`** other than shared 3V3 caps. The datasheet asks for a **0.1–1.0 µF low-leakage** cap when VBAT is the primary supply. `C22` = 1 µF is on 3V3 and appears to be positioned for the RTC — confirm it is physically adjacent, and specify it as a low-leakage dielectric (X7R, not Y5V).
* **`R45`/`R58` — you say these are for an alternate config; here is the exact trace so you can judge.** `R52` (0 Ω, fitted) connects `I2C_SCL ↔ /PIN_6`; `R45` (0 Ω, DNP) connects **the same pair**, `I2C_SCL ↔ /PIN_6`. Likewise `R46` (fitted) and `R58` (DNP) both connect `I2C_SDA ↔ /PIN_5`. So populating the alternate set changes nothing on SDA/SCL. The VDD/INT jumpers *do* work correctly: `R42`(3V3→PIN_2)+`R44`(TP_INT→PIN_4) default, vs `R43`(TP_INT→PIN_2)+`R66`(3V3→PIN_4) alternate — a genuine swap. **If your alternate panel also needs SDA/SCL crossed, the current wiring will not do it** — you would need `R45: SCL→/PIN_5` and `R58: SDA→/PIN_6`. If the alternate panel keeps SDA/SCL in the same positions, then `R45`/`R58` are simply redundant and harmless.
* **Only 5 `PWR_FLAG`s and 0 hierarchical sheets** for a 167-part design on one A2 page. The flat single-sheet approach is why the `4-way junction` and `multiple net names` warnings exist. If you revise, splitting into sheets (Power / MCU / Display / IO) would make the next review far cheaper — 173 global labels on one sheet is a lot to hold in your head.
* **`U12` pin 1 (NC)** is left open ✔ correct for the DBV package.
* **`Q1` pins 2 and 5 (the FS8205A common drain) intentionally unconnected** ✔ correct — they are the internal mid-node. Keep them soldered for thermal reasons.
* **Mounting holes `H1`–`H4` are all `MountingHole_Pad` tied to GND** — fine, but that means four chassis-coupled points. If the enclosure is metal, consider one solid GND and three isolated (or capacitively coupled) to avoid a ground loop through the case.
* **Test points `TP1`–`TP5`** cover RX, TX and the three PSRAM-capable pins. Sensible. Consider adding TPs on `LDO_IN`, `P+` and `USB_STAT` — those are the three nodes this review says to measure.

---

## 7. Scope

This is a **schematic** review — connectivity, part selection, values, margins and datasheet conformance. It does **not** cover layout. The board is 2-layer with native USB, ~26 MHz SDIO and two switching converters (the TPS923610 at 1.1 MHz and the e-paper charge pump); on two layers the return-path integrity under `SW`, `EINK_SW` and the USB pair is the dominant EMC risk and deserves its own review pass against the actual `.kicad_pcb`.
