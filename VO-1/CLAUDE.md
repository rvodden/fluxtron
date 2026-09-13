# FluxTron VO-1 — Voltage-Controlled Oscillator

Read `../CLAUDE.md` first for range-wide decisions (panel aesthetic, depth
rules, common components, bus architecture). This file only covers
decisions specific to VO-1.

## Role

The voice's tone source. Takes 1V/oct pitch CV from MC-1 and produces the
raw waveforms the rest of the chain shapes. Every parameter that would
normally be a trimmer or a panel pot is instead encoder- *and*
MIDI-CC-controllable, per the range premise.

## Core: AS3340

**Alfa Rpar AS3340**, the redesigned CEM3340-compatible VCO IC. Chosen over
a discrete matched-pair core and over the original/reissue CEM3340.

One 16-pin IC provides the whole oscillator: exponential and linear
frequency control inputs, sawtooth, triangle and pulse outputs, a pulse-width
control input, and a sync input. It runs on ±12V.

The deciding factor was temperature stability. The AS3340 carries **full
on-chip temperature compensation and needs no external tempco resistor** —
the expensive, hard-to-source +3300ppm part that a discrete core (or the
original CEM3340) would have required. It does still need precision
resistors (Rt, Rz) from the negative supply pin to the compensation pins as
part of that circuit; exact values to be fixed at schematic capture against
Alfa's own tuning note, which gives Rt ≈ 5.6kΩ with the internal Zener
setting that pin to roughly −6V.

## Control architecture: this module is an I2C bus slave

**This is the decision that changed the range, not just VO-1.** The root
doc previously said phase-1 modules don't use the digital bus — but with
MIDI arriving only at MC-1, there was no path for a CC to reach VO-1's tune
or pulse width, and the range's whole premise depends on one existing.

Resolved by **using the 4-pin I2C bus in phase 1**: MC-1 becomes bus master,
decodes MIDI, and pushes parameter values to each module. VO-1 is a slave.
Rejected alternative was a TRS MIDI IN on every module — it would have cost
1–2 of VO-1's very limited jack positions plus an opto-isolator per module,
and made patching the rack a chore.

The analogue claim survives intact: pitch, gate, clock and audio still
travel as patch-cable voltages. Only parameter values ride the bus.

Two consequences that now apply range-wide, not just here (see
`../CLAUDE.md`):

- **Every module gets an MCU and a 3.3V rail**, not just the ones that
  wanted one. That is a real cost of this decision.
- **The inter-module bus and any local I2C peripheral (e.g. an AS1115) must
  sit on separate MCU I2C ports.** VO-1's MCU is a *slave* on the
  inter-module bus but would need to be *master* to a local AS1115.
  Same-port multi-master plus the address-clash risk isn't worth it.

VO-1 has no AS1115 — one status LED off an MCU GPIO is all the indication
it needs.

### Encoders are incremental, so MIDI and panel don't fight

Worth recording because it falls out of the range's encoder choice for free:
the Bourns PEC16 is a relative/incremental encoder, not a pot. When a CC
changes a parameter there is no stale knob position to reconcile and no
value-pickup problem — the encoder just nudges from wherever the parameter
currently is. A pot-based design would have needed pickup/catch logic.

## Panel: 8HP

### Jack density: four across does not fit

An 8HP panel is 40.64mm wide. Four jacks across means 10.16mm centres, which
puts the outer two about 5mm from the panel edge — roughly 1mm of clearance
past the nut, and no room to get a nut driver onto them. The Thonkiconn
takes an M6 × 0.5 nut, and packing jacks tight enough that the socket won't
fit is a well-known DIY failure mode.

**Three across (13.5mm centres) is the practical maximum at 8HP**, and that
is what VO-1 uses. Two across is comfortable but too wasteful here.

> **This applies to MC-1 too** — its spec currently calls for a row of four
> jacks on an 8HP panel. Flagged in `../MC-1/CLAUDE.md`; needs re-laying out
> before that board is fabbed.

### Layout, top to bottom

1. FLUXTRON wordmark + "VO–1"
2. Encoder row A: **TUNE**, **FINE**
3. Encoder row B: **PW**, **FM**
4. Jack row (inputs, 3): 1V/OCT, FM, PWM
5. Jack row (offset): SYNC at left, tune/status LED at right, with the
   jagged divider line running between the input and output sections
6. Jack row (outputs, 3): SAW, PULSE, SUB
7. Footer wordmark, off-centre

Three jack rows and two encoder rows is what fits: ~48mm of jacks + ~44mm of
encoders + header and footer leaves a little over 12mm of slack in 128.5mm.
A fourth jack row does not fit.

**Seven jacks, not eight.** The chosen output set was saw / pulse /
triangle / sub, but the 3-across limit leaves nine positions across three
rows and the layout above uses seven of them. **Triangle is the one
dropped** — saw, pulse and sub-octave cover a subtractive mono voice, and
the triangle remains available on-chip if a later revision wants it. The
spare position in row 5 is deliberate breathing space and is where the LED
and divider live.

4 mounting holes, symmetric, standard oval slots.

## Controls and parameters

| Parameter | Encoder | MIDI CC | How it reaches the analogue |
|---|---|---|---|
| Coarse tune | TUNE | yes | 16-bit DAC → expo summing node |
| Fine tune | FINE | yes | same DAC channel as coarse |
| Pulse width | PW | yes | 12-bit DAC → AS3340 PW input |
| FM depth | FM | yes | 12-bit DAC → OTA in FM CV path |
| PWM CV depth | — | yes | 12-bit DAC → OTA in PWM CV path |
| Sub division (/2 or /4) | — | yes | MCU GPIO → mux |
| Calibrate now | — | yes | firmware action |

Four encoders, so per the range rule (3+ encoders) they go on a **dedicated
encoder sub-board** with one ribbon back to the main PCB — 4×(A, B) +
4×(switch) + common = 13 lines, so a 14-way ribbon.

### The encoder sub-board has a depth clash the range rule didn't anticipate

The PEC16 body is 16.1mm behind the panel, so a sub-board carrying the
encoders sits at a **16.1mm standoff — 6.1mm *behind* the main PCB's 10mm
plane**, not in front of it. The two boards overlap in space.

Resolved by **notching the main PCB** in the encoder zone (top of the panel)
so the sub-board drops through, mounted on the encoder nuts themselves plus
one standoff. This is worth feeding back into the range-wide encoder/jack
split rule, which is written as though a sub-board is always shallower than
the main board.

### Tune resolution

The tune DAC must be 16-bit. Over a ±5V offset range that is about 0.18
cent per step; a 12-bit part would give ~3 cents, which is audible on a
sustained note. The modulation-depth and pulse-width channels are fine at
12-bit.

## Auto-tune: closed-loop, no trimpots

The pulse output is squared to logic level and routed to an **MCU timer
capture pin**, so firmware can measure the oscillator's actual frequency.
One extra trace, and it turns the MIDI control into a real advantage.

On command (or at power-on) the module runs a **self-calibration sweep**:
it drives its own tune DAC across the range, measures the resulting
frequency at each point, and builds a scale-and-offset correction curve for
the expo converter. That removes the scale and offset trimpots from the
board entirely and corrects thermal drift without a screwdriver.

During calibration the external 1V/OCT input is **grounded by an analogue
switch** so the sweep is valid regardless of what happens to be patched in.

This corrects the AS3340's own contribution. The external CV path still
depends on the precision of its input resistor, but a 0.1% metal-film part
is stable and doesn't drift. **Flagged as a future enhancement, not in
scope now:** adding a precision ADC on the 1V/OCT input so the MCU knows the
incoming pitch and can close the loop in real time — more powerful, but it
risks zipper artefacts and needs a 16-bit ADC.

## Sub-oscillator

A **74HC74 dual D flip-flop** clocked from the squared pulse output gives
/2 and /4. Selection between them is a MIDI-controllable parameter via a
mux on an MCU GPIO, with the result level-shifted back to a Eurorack-level
output. The same squared pulse feeds the auto-tune capture pin, so the
squaring stage (a 74HC14 Schmitt buffer behind a divider) earns its place
twice.

## Modulation depth: LM13700

FM depth and PWM CV depth are attenuations of a bipolar incoming CV, so a
digital potentiometer won't do it — a single-supply digipot can't pass a
±5V signal. Both are handled by one **LM13700 dual OTA**, one channel each,
under DAC control. A quad VCA (SSI2164/AS2164) was considered and is
overkill at two channels.

## Confirmed parts

- **VCO core**: Alfa Rpar AS3340.
- **Jacks**: Thonkiconn PJ301M-12 ×7 (1V/OCT, FM, PWM, SYNC in; SAW,
  PULSE, SUB out), at 13.5mm centres, three across.
- **Encoders**: Bourns PEC16 ×4, on a dedicated sub-board per above.
- **Sub divider**: 74HC74; pulse squaring 74HC14.
- **Depth VCAs**: LM13700.
- **Tune DAC**: 16-bit required; exact part not yet chosen.
- **Parameter DAC**: 12-bit, 3+ channels; exact part not yet chosen.
- **MCU**: not yet chosen — see open items.

## Circuit protection (beyond the range-wide baseline)

See `../CLAUDE.md` for the standard applying to every module. VO-1
additionally needs:

- **Low-leakage clamp diodes on the 1V/OCT input specifically.** The
  range-wide rule puts clamp diodes to the rails on every CV input, but the
  pitch input feeds a high-impedance summing node where ordinary diode
  leakage current becomes a pitch error. Use a low-leakage part (BAV199 /
  BAS116 class) here. Standard clamps are fine on FM, PWM and SYNC.
- **Series resistors on SAW, PULSE and SUB** per the baseline (~1kΩ).
- **Thermal separation.** The AS3340, its compensation resistors and the
  expo summing node must be kept away from the power header, the 3.3V
  regulator and the MCU. Self-heating and local airflow both show up as
  pitch drift.
- **No switching regulator on this board.** The 3.3V rail comes from a
  linear LDO off +12V despite dissipating roughly 0.4W — a buck converter's
  switching noise next to a precision expo converter is not worth the
  efficiency. The per-module PTC rating must account for the LDO's draw.

## Open items

- **MCU choice — range-wide, not just VO-1.** MC-1's spec says "MCU"
  without naming a part, and the I2C decision now puts one on every module,
  so this should be settled once for the whole range. RP2040 is the
  suggestion: its PIO handles four quadrature encoders and the frequency
  counter without touching the CPU, it has two I2C ports (which the
  bus/local-peripheral split above needs), and MC-1 needs a USB device
  controller anyway. Against it: needs an external QSPI flash, where an
  STM32G0 is single-chip.
- Tune DAC and parameter DAC part numbers.
- The bus protocol itself — parameter addressing/encoding over I2C is now
  load-bearing in phase 1 and isn't specified anywhere yet.
- Whether VO-1 wants any parameter readout at all, or whether the DAW/host
  is the only place a MIDI-set value is visible.
- Exact Rt/Rz values for the AS3340 compensation circuit.
- Exact mounting-hole, jack and encoder coordinates against real footprints.
