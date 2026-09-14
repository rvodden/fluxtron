# FluxTron — Project Overview

A MIDI-controllable analogue Eurorack modular synth system. Looks and patches
like a normal analogue modular system, but every parameter is also
MIDI-controllable via CC/NRPN, with panel encoders for direct hands-on
control.

This file holds decisions that apply across the whole module range.
Module-specific decisions live in each module's own folder
(e.g. `MC-1/CLAUDE.md`).

## Product identity

- Range name: **FluxTron**
- Module naming: `FluxTron AB-N` — two/three-letter code + number.
  Same design built more than once (e.g. two envelope generators) reuses one
  code with higher quantity, not a new number.
- Panel aesthetic: aluminium panels, matte black, etched so graphics read as
  silver-on-black. Brief: "clean but psychotic" — jagged hand-drawn divider
  lines and small asymmetric details (e.g. a scratch/crack mark) rather than
  clutter. **No footer wordmark** — an off-centre one was in the original
  brief, tried on VO-1 and rejected; don't re-propose it. The FLUXTRON
  wordmark appears once, at the top. Sections use symbolic icons where
  natural (e.g. a MIDI DIN end-view icon, a pulse-wave icon) rather than text
  labels, except where a plain word is clearer (jack names, "CHANNEL").
- **Indicators are blue, range-wide — FluxTron is cold, not warm.** Every
  indicator LED, and every 7-segment display where a blue part can be
  sourced, is blue. No amber, no warm white, no stock red. This is a
  deliberate palette decision against the matte-black/silver panel, and it
  overrides the earlier "warm glow" suggestion that had been floated for
  MC-1's displays. See the drive consequence under Common components — blue
  is not a drop-in substitution for red.
- Jack labelling: the 1V/octave pitch jack is labelled **V/OCT** on every
  module, input and output alike, so both ends of a patch cable read the
  same. (`1V/O` was tried on VO-1 and dropped as less conventional.)
- Fonts: **Rubik Dirt** for the FLUXTRON wordmark, **Kode Mono** for
  everything else (module code, jack labels, control labels). Use an en-dash
  in module codes, e.g. "MC–1".
- Sizing rule: every module width is a **multiple of 8HP**.

## Scope: Phase 1 (monophonic voice)

Monophonic system. Modules:

| Module | Function | Qty | HP |
|---|---|---|---|
| MC-1 | MIDI-to-CV/Gate interface | 1 | 8 |
| VO-1 | VCO | 1 | 8 |
| VF-1 | VCF | 1 | 8 |
| VA-1 | VCA | 1 | 8 |
| EG-1 | Envelope generator (ADSR) | 2 | 8 each |
| LF-1 | LFO | 1 | 8 |
| MX-1 | Mixer | 1 | 8 |

Total 64HP of 84HP (KOMA Case 3U/84HP) — 20HP spare, not yet allocated
(candidates: extra spacing, blind panels, an output/headphone module).

Flagged for later, not in scope now: **LF-2**, a fuller multi-waveform
crossfade LFO (separate sine/saw/square cores blended via a dual-VCA-style
crossfader), vs. LF-1's single continuous triangle↔saw/square skew control.

## Case

**KOMA Case 3U/84HP** — unpowered (bring your own PSU), 70mm depth. Bought
direct from koma-elektronik.com (not widely stocked via third-party
retailers). PSU/power distribution to be designed separately, not tied to
the case vendor's own bus board.

## Signal architecture: hybrid CV + digital bus

- **All audio and CV is analogue** — pitch, gate, velocity, clock and audio
  all travel as patch-cable voltages, same as traditional Eurorack. Only
  *parameter values* ride the digital bus. Each module's MCU handles its own
  MIDI-CC parameters (encoders), not system-level note/gate data.
- **The shared digital bus is used from phase 1**, and MC-1 is its master.
  (This reverses the original plan of leaving the bus unpopulated in phase 1
  — see VO-1's spec for the reasoning. MIDI arrives only at MC-1, so without
  the bus there was no path for a CC to reach any other module's parameters,
  and the range's whole premise depends on one existing. The alternative —
  a TRS MIDI IN plus opto-isolator on every module — cost jack positions
  that an 8HP panel does not have.)
  - Physical: a **separate 4-pin connector** (bus VCC, SDA, SCL, GND)
    alongside the standard Eurorack power header, its own ribbon, kept away
    from analogue sections at layout time. Populated on every module.
  - Protocol: **I2C**, reusing the same physical layer as each module's
    local AS1115 (see below).
  - Addressing: **DIP/jumper-selectable** for phase 1. Auto-addressing
    (daisy-chained enable line) is a flagged future enhancement, not
    required now.
  - Multi-case future: **buffered I2C** (e.g. PCA9615-style differential
    extender) when that's needed — no CAN/RS-485 headroom being designed in.
  - **Consequence: every module carries an MCU and a 3.3V rail**, not just
    the ones that wanted one. Accepted cost of the above.
  - **Consequence: the inter-module bus and any local I2C peripheral (e.g.
    an AS1115) must sit on separate MCU I2C ports.** A slave module's MCU is
    a slave on the inter-module bus but master to its own peripherals;
    sharing one port invites multi-master trouble and address clashes.
  - **Analogue modules use a linear LDO for their 3.3V rail, not a buck** —
    switching noise next to precision analogue (expo converters especially)
    is not worth the efficiency. Budget the dissipation into the PTC rating.

## PCB / panel construction

- PCB mounted **parallel to the panel** (standard Eurorack practice) — not
  perpendicular. Panel-mount components (pots, encoders, jacks) are designed
  around this orientation; perpendicular would need right-angle variants of
  everything.
- **Depth budget is set by whichever component needs the most reach from
  panel to PCB** — and different component families need very different
  reach, which is the recurring design problem across every module:
  - **Jacks** (Thonkiconn PJ301M-12, the Eurorack-standard 3.5mm mono jack):
    **10mm** panel-to-PCB. This is now the number that sets the main PCB's
    standoff distance on every module, since jacks are on every module and
    are the shallowest of the "needs real panel-mount depth" components.
  - **THT encoders** (Bourns PEC11R): **6.5mm** behind the panel's rear
    face, switched and unswitched variants alike. That is *shallower* than
    the 10mm jack plane, not deeper, so there is no clash, no cutout and
    no keep-out — the body sits wholly in front of the main PCB with about
    3.5mm to spare, and nothing on the board has to route around it.
    The constraint runs the other way instead: an encoder soldered to a
    PCB puts that PCB at 6.5mm, so it still cannot share the main board at
    10mm. Flying leads or a dedicated sub-board at its own 6.5mm standoff,
    per the split rule below.
  - **Small digit displays / USB-C**: too *shallow* to sit at the 10mm jack
    depth and still reach the panel (see MC-1 for the worked example) — each
    needs its **own small daughterboard** at its own shallower standoff,
    connected back to the main PCB via a short header/jumper. Components
    needing meaningfully different depths do **not** share one daughterboard
    even if both are "digital" — depth compatibility decides board grouping,
    not function.
  - **THT indicator LEDs**: not a depth constraint at all — leads are simply
    trimmed/bent to reach the panel flush, whatever the standoff. Prefer
    THT over SMD for any panel-facing indicator LED for this reason.
- **Encoder/jack split rule for boards with multiple encoders**: 1–2
  encoders on a module → flying leads straight to the main board (simple,
  cheap, matches MC-1). **3+ encoders** → give that module a small dedicated
  encoder sub-board carrying all its encoders, with one multi-pin
  header/ribbon back to the main board, rather than a growing bundle of
  loose flying leads. (Applies to VO-1; likely LF-1 too; check EG-1 once its
  control set — fixed vs. continuous ADSR stages — is settled.)
  - **A sub-board only helps when the encoders are clustered**, so lay the
    panel out that way when a module has 3+. VO-1 went through a mockup with
    its four encoders scattered diagonally and interleaved with jacks, which
    would have forced 20 loose flying leads because a sub-board spanning
    them was the main PCB's own footprint. Clustering them into one block
    fixed it.
  - **A clustered encoder block means the main PCB stops short of it**
    rather than running the full panel height — no notch needed, but the
    remaining board area gets tight on an 8HP module. See VO-1.
- **Jack density: three across is the maximum on an 8HP panel.** 8HP is
  40.64mm, so four across means 10.16mm centres — about 1mm of clearance
  past the outer nuts to the panel edge, and no room to get a nut driver
  onto them. Three across is 13.5mm centres and works; two across is
  comfortable. Budget panel layouts against three, not four.
- Mounting holes: use proper elongated/oval slots (not round), sized per
  standard Eurorack practice, not plain round corner holes.

## Common components (repeat across modules)

- **Jacks**: Thonkiconn PJ301M-12, 3.5mm mono, panel-mount — 10mm depth,
  sets main PCB standoff on every module.
- **Encoders**: **Bourns PEC11R**, 12mm incremental, THT, M7 × 0.75 metal
  bushing and metal shaft, 6.5mm behind the panel. Replaces the PEC16,
  which was larger (M9 bushing, 16mm body) and whose behind-panel depth the
  range doc could never confirm. Datasheet:
  `datasheets/PEC11R_Bourns_encoder_revD_2026-04.pdf`.
  - `PEC11R-4015F-N0024` — no switch
  - `PEC11R-4015F-S0024` — with push momentary switch
  - Decode: `4` = PC pin horizontal/rear-facing, `0` = **no detents**,
    `15` = 15mm shaft, `F` = metal flatted (D) shaft, `N`/`S` = switch
    option, `0024` = 24 pulses per revolution.
  - **Detentless, deliberately.** Smooth travel suits the continuous
    parameters that dominate the range — tune, fine, pulse width, filter
    cutoff, envelope stages — where a click per step would fight the
    control rather than help it. The trade falls on the discrete
    enumerated controls, MC-1's channel and clock division, which lose
    their tactile step and lean entirely on the display for feedback.
    Note the datasheet's detent-torque figure no longer applies; running
    torque (10–70 gf-cm) is the one that does.
  - **⚠️ Firmware: "24 pulses per revolution" is 24 full quadrature cycles,
    i.e. 96 edge transitions.** Count whole cycles, not edges. At 24 steps
    per revolution MC-1's 15 divisions span about 225° and its 16 channels
    about 240°, both comfortable one-handed sweeps. Counting all four
    edges gives 96 steps per revolution and makes every control on every
    module unusably twitchy. Contact bounce is 2.0ms max at 15 RPM;
    debounce against that.
  - **15mm shaft**, measured from the mounting surface, so roughly 13mm
    proud of a 2mm panel — a good match for a standard Eurorack knob bore.
  - **Bourns states hand soldering is not recommended** (wave solder,
    260°C max for 3 ±1s). Worth knowing for a hand-built module; it is not
    a prohibition so much as a warranty boundary.
  - **⚠️ A press will nudge a detentless shaft.** At 610 ±306gf the switch
    is stiff, and with no detent to hold position the knob will sometimes
    rotate as it is pressed. Suppress rotation for ~50ms after a press
    edge, then resume so hold-and-turn gestures still work. Applies to
    every switched encoder on every module.
  - Switch is SPST momentary, 0.5mm travel, 610 ±306gf. Rotational life
    30,000 cycles, switch life 20,000. Contacts rated 10mA @ 5VDC.
- **LED/button driver**: **AS1115** (I2C) — drives up to 64 LEDs or 8 digits
  of 7-segment, plus keyscan for up to 64 buttons. One per module is
  generally enough to cover an LED ring (where used), a pushbutton, and/or a
  small digit display. Confirm common-anode/common-cathode polarity against
  the specific display part before committing (AS1115 expects a specific
  drive polarity).
- **⚠️ Blue LEDs do not run off the 3.3V rail.** Red and amber AlGaInP dice
  drop about 2.0V typical / 2.5V max; blue InGaN/GaN drops **3.0V typical
  and 3.8V maximum** (figures from the Guangcai GS2022 datasheet, MC-1's
  display, and representative of blue dice generally). Two consequences of
  the blue-indicator decision, both of which have to be designed in rather
  than discovered at bring-up:
  - **A blue LED cannot be driven directly from a 3.3V MCU GPIO.** Allowing
    for the GPIO's own drop there is around 2.9–3.1V available, which for a
    3.2V part leaves nothing across the series resistor — dim, and wildly
    variable part to part. Drive indicator LEDs from **+12V through a series
    resistor, switched by a small NPN or MOSFET off the GPIO** instead.
    +12V is guaranteed on the 10-pin header; a +5V rail is not. Size for
    2–3mA, not 20 — blue LEDs are bright and the panel is matte black.
  - **The AS1115 sources segment current from its own supply**, so its V+
    must exceed the segment's forward voltage plus driver dropout. At 3.3V
    it cannot drive blue segments at all. Any module with a blue display
    needs a **local 5V rail** (another LDO from +12V) for the AS1115.
    Budget that dissipation into the module's PTC rating, same as the 3.3V
    LDO. Note the margin is genuinely tight — a 3.8V worst-case segment
    against the AS1115's 5.5V maximum leaves little for the drivers — and
    a 5V AS1115 talking to a 3.3V MCU needs the I2C level shift thought
    about. MC-1's spec works this through; any later module with a blue
    display inherits the same three problems.
- **MIDI I/O**: **3.5mm TRS, Type A** (MIDI Association-ratified standard,
  2018) — not 5-pin DIN (too big), not 2.5mm (non-standard minority format).
- **Bus connector**: 4-pin (VCC, SDA, SCL, GND), separate from power header,
  populated on every module regardless of phase.

## Circuit protection standard

Applies to every module's schematic. Cheap to design in now, painful to
retrofit after boards are fabbed — treat this as a checklist each module
must satisfy, not a per-module decision to re-derive.

- **Reverse power protection**: shrouded, keyed 10-pin power header on every
  module (prevents backwards insertion mechanically) *plus* a
  reverse-polarity protection circuit on each rail as a backstop for the
  "offset by one pin" case a keyed shroud doesn't catch. Default to simple
  series diodes (~0.7V drop, acceptable given ±12V headroom) unless a
  specific module's circuit is voltage-sensitive enough to need an
  ideal-diode/PMOS approach instead.
- **Per-module overcurrent protection**: a resettable PTC polyfuse on each
  power rail, per module, so a fault on one module can't pull down the
  shared bus and affect its neighbours. Exact current rating TBD per
  module's actual draw.
- **Output short-circuit protection**: series resistor (~1kΩ typical) on
  every CV/audio/gate output, so a patching mistake (output shorted to
  ground or to another output) is current-limited rather than damaging.
- **Input overvoltage protection**: clamp diodes to the rails on every
  CV/audio input, since any input can have another module's output patched
  into it and Eurorack signal levels aren't universally well-behaved.
- **ESD protection**: TVS diode arrays on jack signal lines (and on any
  other user-facing exposed conductor, e.g. encoder shafts) on every
  module — patch cables and panel controls are genuine static discharge
  entry points.
- **Decoupling**: adequate local decoupling (bulk electrolytic + local
  ceramic near ICs) on every module, so a module's own current transients
  don't drag down the shared rail for its neighbours — particularly
  relevant given the shared backplane/daisy-chain approach.
- **Bus connector (I2C) ESD**: basic ESD protection on SDA/SCL at each
  module's bus connector, even in phase 1 while the bus is unused — cheap
  now, saves a retrofit once the bus is load-bearing across more modules.

Module-specific protection beyond this baseline (e.g. MC-1's USB-C ESD/VBUS
protection and MIDI opto-isolation) is noted in that module's own
`CLAUDE.md`, not here.

## Open items / not yet decided

- Spare 20HP allocation (extra spacing vs. blind panel vs. new module).
- **Bus parameter protocol** — how CC/NRPN values are addressed and encoded
  over I2C. Load-bearing in phase 1 now, and not yet specified.
- **MCU choice, range-wide** — every module needs one under the bus
  decision, so pick once rather than per module. RP2040 vs. STM32G0 is the
  live question; see VO-1's open items for the trade-off.
- EG-1's control set (fully continuous ADSR via 4 encoders vs. some fixed
  stages) — affects whether it needs a dedicated encoder sub-board.
- PSU design for the (unpowered) KOMA case.
- Auto-addressing scheme for the digital bus (deferred, not blocking).
