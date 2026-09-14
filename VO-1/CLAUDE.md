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

VO-1 has no AS1115 — one status LED is all the indication it needs. Note
that it is **not** driven straight off an MCU GPIO: it is blue per the
range palette, and a blue LED's forward voltage leaves no headroom at 3.3V.
It runs from +12V through a series resistor, switched by a small
NPN/MOSFET off the GPIO. See `../CLAUDE.md`.

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
is what VO-1's two full jack rows use. The remaining two (SYNC and PULSE)
share a wider-spaced row of their own — see the layout below.

> **This applies to MC-1 too** — its spec currently calls for a row of four
> jacks on an 8HP panel. Flagged in `../MC-1/CLAUDE.md`; needs re-laying out
> before that board is fabbed.

### Layout, top to bottom

Settled from a panel mockup. All eight jacks fit **and** the four encoders
stay clustered, which is the arrangement that keeps the encoder sub-board
viable (see below). Controls on top, jacks below, with SYNC and PULSE
sharing a row rather than needing a fourth full jack row — which would not
have fitted vertically.

1. FLUXTRON wordmark + "VO–1"
2. Jagged divider
3. Encoder row A: **FINE** (left), **TUNE** (right)
4. Encoder row B: **DEPTH** (left), **WIDTH** (right)
5. Jack row (inputs, 3 across): **V/OCT**, **FM**, **PWM**
6. **SYNC** jack at left, **PULSE** jack at right, gap between them
7. Jack row (outputs, 3 across): **SAW**, **SUB**, **TRI**
8. Jagged divider

**All eight jacks fit, and triangle is kept** — an earlier draft dropped it
to live within seven positions across three clean rows, which the row-6
pairing makes unnecessary.

4 mounting holes, symmetric, standard oval slots.

### Mockup drawing conventions

So these aren't re-flagged on every review of a panel mockup:

- **Knobs are drawn oversize on purpose** — they represent the twiddle
  clearance a finger needs, not the diameter of the cap. Don't read the
  drawn circle as the part.
- **The indicator line is a drawing convention** marking "this is a knob,
  not a random circle." It is not panel art and not a feature of the part.

The related real decision: **the fitted knob caps carry no indicator line.**
The encoders are incremental, so there is no absolute position for a pointer
to show, and one would desynchronise the moment a MIDI CC moved the
parameter. If position feedback is ever wanted it has to be an LED ring,
which was ruled out here on space and cost.

### Jack labelling

The pitch input is labelled **V/OCT** — the conventional spelling, settled
in favour of the earlier `1V/O`. It stays unambiguous against FM and PWM,
which was the original concern, and it matches the label on MC-1's pitch
*output*, so both ends of the patch cable read the same. Now a range-wide
convention (see `../CLAUDE.md`).

## Controls and parameters

| Parameter | Encoder | MIDI CC | How it reaches the analogue |
|---|---|---|---|
| Coarse tune | TUNE | yes | 16-bit DAC → expo summing node |
| Fine tune | FINE | yes | same DAC channel as coarse |
| Pulse width | WIDTH | yes | 12-bit DAC → AS3340 PW input |
| FM depth | DEPTH | yes | 12-bit DAC → OTA in FM CV path |
| PWM CV depth | — | yes | 12-bit DAC → OTA in PWM CV path |
| Sub division (/2 or /4) | — | yes | MCU GPIO → mux |
| Calibrate now | — | yes | firmware action |

Four encoders, clustered as a 2×2 block at the top of the panel with every
jack below them. Per the range rule (3+ encoders), they go on a **dedicated
encoder sub-board** — 4×(A, B) + 4×(switch) + common = 13 lines, so a 14-way
ribbon back to the main PCB.

The clustering is what makes this work. A scattered layout (an earlier
mockup put the four encoders diagonally down the panel, interleaved with
jacks) would have forced 20 loose flying leads instead, because a sub-board
spanning them would have been the main PCB's own footprint.

**Consequence for the main PCB: it ends below the encoder zone rather than
running the full panel height.** That keeps the sub-board clear without
notching anything, but it concentrates the AS3340, MCU, both DACs, the
LM13700, the 74HC74/74HC14 pair, the power section and all eight jacks into
roughly the lower two-thirds — call it 40mm × 80mm. Workable, but expect to
go 4-layer rather than 2.

### Status LED: placed

The auto-tune sweep needs an indicator — without one there is no way to
tell whether a calibration is running, finished, or failed, and the module
would just go quiet mid-sweep with no explanation. An earlier mockup had
dropped it.

**Now sited in the row-6 gap between SYNC and PULSE**, which was already
empty. THT LED per the range-wide preference.

**Blue**, per the range-wide indicator palette — FluxTron is cold, and the
mockup already draws it that way. The one thing this changes electrically:
a blue LED will not run off a 3.3V GPIO, so it is switched from +12V
through a transistor (see the control-architecture section above and
`../CLAUDE.md`). Size the series resistor for 2–3mA; a blue LED at full
tilt on a matte black panel is a distraction, not an indicator.

### ⚠️ Unverified: does a PEC16 actually clash with the main PCB?

The previous draft of this file (and the range-wide rule it fed into)
asserted that a PEC16 body extends 16.1mm behind the panel and therefore
intersects the main PCB's 10mm plane, requiring a cutout. **That is not
confirmed and may well be wrong.**

The range doc's 16.1mm figure is described as "body length behind the
panel", but Bourns' own dimension list pairs it with an **M9 × 0.75 bushing
in 8.3 / 9.3 / 12.5mm lengths** — which reads much more like 16.1mm being an
*overall* length with the bushing included. If so, the behind-panel body is
somewhere near 8mm and **clears the 10mm main PCB entirely**, with no cutout
needed anywhere on any module.

The two outcomes differ a lot:

- **Behind-panel body ≲ 9mm**: no cutouts, no clash. Encoders still can't be
  main-PCB-mounted (they'd fall short of reaching the panel from 10mm), so
  flying leads regardless — but the boards get much simpler.
- **Behind-panel body ≈ 16.1mm**: every PEC16 on every module needs a
  clearance hole (~14mm) through the main PCB, flying leads or not. On VO-1
  that is four holes punched through the middle of the board that has to
  carry the AS3340, MCU, DACs, LM13700 and power section.

**Check the PEC16 dimensional drawing before laying out any module.** The
distributor and Bourns domains are blocked from this environment, so it
could not be resolved here. This also decides whether the range-wide
encoder-depth rule in `../CLAUDE.md` is correct as written.

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
- **Jacks**: Thonkiconn PJ301M-12 ×8 (V/OCT, FM, PWM, SYNC in; SAW,
  PULSE, SUB, TRI out). The three-across rows are at 13.5mm centres; SYNC
  and PULSE sit beside the knobs.
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
