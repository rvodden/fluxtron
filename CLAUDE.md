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
  - **THT encoders** (Bourns PEC11R, M7 × 0.75 bushing, 12mm body):
    deeper than the 10mm jack standoff, so not something to let dictate the
    whole board. Resolved by **not mounting the encoder on the main PCB at
    all** — it's wired to the main board on flying leads (A, B, common,
    switch ×2 — 5 wires) and gets its mechanical support entirely from its
    own panel nut, same as always; or, on modules with 3+ encoders, onto
    their own sub-board at their own standoff. Either way the main PCB must
    keep clear of the space the encoder body occupies — see the depth note
    below, which is not yet settled.
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
- **⚠️ The PEC11R's behind-panel depth is not yet confirmed.** Bourns and
  every distributor mirror are blocked from the agent environment, and the
  secondhand figures conflict: one listing implies a body ~6.5mm behind the
  panel (21.5mm overall less a 15mm shaft), another reports 12.5mm, which
  looks like the 12mm body *width* misread as depth. If it clears 10mm there
  is nothing to do; if it exceeds 10mm, any main PCB running behind an
  encoder needs a clearance hole. **Resolve against the dimensional drawing
  before laying out a board where an encoder sits over the main PCB.**
  - This does **not** gate modules whose encoders are clustered onto a
    sub-board with the main PCB stopping short of them — the two never share
    that space. VO-1 is clear for this reason. It gates MC-1, whose single
    encoder sits amid a full-height main board.
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
- **Encoders**: **Bourns PEC11R**, THT, 12mm body, **M7 × 0.75** panel-mount
  bushing — mounted off-PCB on flying leads, or on a sub-board where a
  module has 3+, per the depth rule above. (Replaced the 16mm PEC16/M9 after
  MC-1's layout work: tighter footprint and a smaller panel hole, which
  matters on a 40.64mm-wide panel. The 12mm series generally specs lower
  rotational life than the 16mm — the accepted trade for the footprint.)
- **LED/button driver**: **AS1115** (I2C) — drives up to 64 LEDs or 8 digits
  of 7-segment, plus keyscan for up to 64 buttons. One per module is
  generally enough to cover an LED ring (where used), a pushbutton, and/or a
  small digit display. Confirm common-anode/common-cathode polarity against
  the specific display part before committing (AS1115 expects a specific
  drive polarity).
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
