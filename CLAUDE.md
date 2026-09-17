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
  wordmark appears once, at the top. **No symbolic icons** — a MIDI DIN
  end-view icon and a pulse-wave icon were in the original brief and are
  rejected; don't re-propose them. Everything is named in words: jack
  names, control labels, and a plain word heading a section where one is
  needed.
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
| PS-1 | Power supply + bus board | 1 | **16** |

Total 80HP of 84HP (KOMA Case 3U/84HP) — 4HP spare, not enough for another
module, so it is spacing or a blind panel.

**PS-1 is a module in its own right, not a section of MC-1**, and is **settled
at 16HP** — 8HP was the earlier hope, and the thermal budget closed it off
once the supply was sized for a full 16-module bus segment rather than for
phase 1's eight. It supplies **±12V and +5V** from an external **15V 3A**
DC brick, and owns the bus board. Everything else about it — rail
generation, regulator choice, current budget, thermals — is in
`PS-1/CLAUDE.md` and does not belong here.

Flagged for later, not in scope now: **LF-2**, a fuller multi-waveform
crossfade LFO (separate sine/saw/square cores blended via a dual-VCA-style
crossfader), vs. LF-1's single continuous triangle↔saw/square skew control.

## Case

**KOMA Case 3U/84HP** — unpowered (bring your own PSU), 70mm depth. Bought
direct from koma-elektronik.com (not widely stocked via third-party
retailers). PSU and power distribution are designed separately and are not
tied to the case vendor's own bus board — they are **PS-1**, which owns both
the supply and the backplane the rack plugs into. See `PS-1/CLAUDE.md`.

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
  - **The protocol is specified in `BUS.md`** — addressing, register map,
    transactions, presets, firmware update and the MIDI mapping. What follows
    here is only what a module author must obey.
  - Physical: a **separate 2×6 (12-pin) connector** (DVCC, GND, SDA, SCL,
    nRESET, ATTN, A0–A3, 2 spare) alongside the standard Eurorack power
    header, its own ribbon, kept away from analogue sections at layout time.
    Populated on every module. **⚠️ Never a 10-pin connector** — a 10-pin
    socket mates with a 16-pin shrouded power header *by design*, that being
    ordinary Eurorack practice, so a 10-pin bus plug would go straight onto
    ±12V and destroy every MCU on the backplane. With power now 2×8
    range-wide, **the range contains no 10-pin connector anywhere**, so this
    rule is satisfied by construction rather than by remembering it.
  - Protocol: **I2C**, reusing the same physical layer as each module's
    local AS1115 (see below).
  - Addressing: **geographic — the backplane holds the address.** Each bus
    board position ties `A0–A3` to a different pattern; the module reads them
    as GPIOs at boot. Two levels, chassis and slot, with only the CX-1 bridge
    needing configuration. **Supersedes the earlier DIP/jumper scheme and the
    daisy-chained auto-addressing enhancement** — geographic needs no
    enumeration protocol at all, and makes duplicate addresses physically
    impossible. See `BUS.md` §3.
  - Multi-case: a **CX-1 bridge module** — slave on the upstream chassis's
    bus, master on its own, over a PCA9615-style differential link. Not a
    transparent buffer; each chassis keeps a private address space. No
    CAN/RS-485 headroom is being designed in. See `BUS.md` §4.
  - **⚠️ The invariant the whole design rests on: the bus carries only what
    can tolerate tens of milliseconds of worst-case latency.** Pitch, gate,
    velocity, clock and audio stay on patch cables — and so do note events,
    which fail the test twice over: a slave stalls the bus for tens of
    milliseconds during a flash erase, and a dropped note-off is a note that
    sustains forever, where a wire simply cannot lose its state. Note *number*
    as information is fine; the triggering role is not. See `BUS.md` §1.
  - **Consequence: every module carries an MCU and a 3.3V rail**, not just
    the ones that wanted one. Accepted cost of the above.
  - **Consequence: the inter-module bus and any local I2C peripheral (e.g.
    an AS1115) must sit on separate MCU I2C ports.** A slave module's MCU is
    a slave on the inter-module bus but master to its own peripherals;
    sharing one port invites multi-master trouble and address clashes.
  - **Analogue modules use a linear LDO for their 3.3V rail, not a buck** —
    switching noise next to precision analogue (expo converters especially)
    is not worth the efficiency. Budget the dissipation into the PTC rating.
  - **3.3V stays per-module, but its LDO is fed from the bus +5V, not +12V** —
    a 1.7V drop instead of 8.7V, which takes VO-1's regulator from 0.4W to
    78mW beside its expo converter. (**⚠️ That 0.4W is an estimate, not a
    measurement**, and it implies a 46mA 3.3V domain against roughly 18mA
    from the datasheets — a 3× spread that decides PS-1's +5V regulator
    rating. See `PS-1/CLAUDE.md`; measure one real module before ordering.)
    A *shared* 3.3V rail stays rejected: no rejection stage between modules,
    and it breaks the per-module PTC rule.
    - **Consequence, now confirmed: every module fits a 16-pin power header.**
      If every module's 3.3V comes from +5V then every module needs +5V —
      there is no such thing as a module that can keep a 10-pin header. Deriving
      3.3V from +12V instead is 0.4W per module against 78mW, roughly 3.2W of
      scattered heat across a phase-1 rack against 0.6W.
    - **⚠️ Do not confuse the two uses of +5V.** *Every* module takes it as its
      3.3V LDO input. Only modules with a **blue display** additionally need it
      as the AS1115's supply (see Common components). "VO-1 needs no 5V rail"
      means no display rail; VO-1 still takes +5V at its header like everything
      else.
    - Keep the portability fallback: a wide-Vin LDO with a **jumper selecting
      bus +5V or bus +12V** as its input, so a module still runs in a case whose
      PSU has no 5V rail — hot, but working.

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

- **MCU**: `STM32G0B1CBT6` on every module — see the MCU section below for
  the reasoning and for what a per-module substitution costs.
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
    Size for 2–3mA, not 20 — blue LEDs are bright and the panel is matte
    black. (A FluxTron rack has a guaranteed +5V rail, so this is no longer
    forced; the +12V-and-transistor approach stays the default because it
    costs almost nothing and keeps modules portable to cases without one.)
  - **The AS1115 sources segment current from its own supply**, so its V+
    must exceed the segment's forward voltage plus driver dropout. At 3.3V
    it cannot drive blue segments at all. Any module with a blue display
    needs the **+5V rail** as the AS1115's supply. **Since PS-1 that comes
    from the bus**, not from a local LDO — the local part burns ~560mW on
    MC-1, the range's worst thermal spot, on its most crowded board. Every
    module already has a 16-pin header for its 3.3V LDO input, so a display
    module needs no extra connector, only extra current. Lay out the local
    LDO anyway, unpopulated, with a jumper selecting the source — it is what
    keeps the module working in a case whose PSU has no 5V rail. Budget that
    dissipation into the module's PTC rating only if the LDO is populated.
    **⚠️ Budget the AS1115's rail at the pin, not at the bus.** The
    often-quoted 1.2V of driver headroom (5.0V less a 3.8V worst-case
    segment) assumes a clean 5.0V that no module ever sees: the protection
    standard puts a reverse-polarity element and a PTC in series, and a long
    backplane adds IR drop on top. A 0.7V silicon diode alone takes the
    margin to ~0.4V, which is not a margin. Hence the Schottky/PMOS rule for
    +5V under Circuit protection, a low-resistance PTC, and heavy copper on
    the bus board's 5V pour. Note the margin is genuinely tight — and
    a 5V AS1115 talking to a 3.3V MCU needs the I2C level shift thought
    about. MC-1's spec works this through; any later module with a blue
    display inherits the same three problems.
- **Parameter DAC**: **MCP4728** — 12-bit, 4-channel, I2C, MSOP-10, with
  on-board EEPROM. The standard part for every module's parameter channels
  (pulse width, modulation depths, velocity, and their equivalents
  elsewhere).
  - **It is I2C, not SPI**, so it sits on the module's *local* I2C port
    alongside any AS1115 — never on the inter-module bus, per the two-port
    rule above. Its default address (0x60) does not clash with the
    AS1115's.
  - **⚠️ Never use it for anything pitch-related.** Its internal reference
    is not a precision part. Tune and V/OCT stay on a dedicated 16-bit DAC
    with its own reference; the MCP4728 is for parameters only.
  - **Output range is bounded by its own VDD**, so it does not produce
    Eurorack-level CV directly — every channel goes through an op-amp
    scaling and offset stage. At 3.3V VDD the internal 2.048V reference at
    gain 1 is the usable full scale; gain 2 would clip against the rail.
  - **Use the on-board EEPROM for a sane power-on state** (pulse width at
    50%, depths at zero) so a module is neither silent nor screaming in
    the moments before firmware initialises. It is *not* the store of
    record — the authoritative settings live in MCU flash, and two sources
    of truth is worse than one.
  - **⚠️ Two on one bus needs address programming.** The three address LSBs
    are set by an LDAC-assisted write sequence, not by pins. Fine at one
    per module; plan for it if a module ever needs more than 4 channels.
- **Pitch DAC**: **AD5693R** (nanoDAC+, 16-bit, single-channel, I2C,
  buffered rail-to-rail output, **2.5V on-chip reference at 2ppm/°C**).
  One on VO-1 (tune) and one on MC-1 (V/OCT out).
  - **The on-chip reference is the point.** It beats a discrete REF5025 at
    its 3ppm grade (0.48 cent of warm-up drift against 0.72), deletes a
    part and its capacitors from two modules, and removes a whole class of
    mistake — there is no wrong reference grade to order by accident.
  - **It also removes VO-1's rail risk.** An AD5662 + REF5025 pairing would
    have had the reference generating 2.5V from VO-1's 3.3V rail, since
    VO-1 has no AS1115 and so nothing drawing on +5V beyond its LDO input. Whether the dropout allowed it
    was an open question that could have forced a rail VO-1 does not
    otherwise want. The AD5693R runs from 2.7–5.5V and makes its own
    reference, so the question never arises.
  - **⚠️ Firmware must write pitch before raising gate.** The pitch DAC now
    shares the local I2C bus with the MCP4728 and any AS1115, so a note-on
    can in principle queue behind a parameter update. The magnitude is
    small — about 95µs at 400kHz, against 320µs for a *single MIDI byte* —
    but ordering the writes removes it entirely and makes any residual
    delay apply to the whole note rather than skewing pitch against gate.
    **This is now a firmware requirement rather than an optimisation**,
    which it would not have been on a separate SPI bus. It is the one real
    cost of the choice.
  - Addresses do not clash: the AD5693R sits around 0x4C (A0-selectable),
    the MCP4728 at 0x60, the AS1115 low. On MC-1 it belongs on the **3.3V
    side of the AS1115 level shifter**, alongside the MCP4728.
  - **Supersedes an earlier AD5662 + REF5025 pairing**, which was chosen on
    sourcing grounds when the AD5683R (the SPI sibling of this part) proved
    hard to find. The AD5693R is the same silicon over I2C and is
    available, so the reason for the discrete pairing went away.

### ⚠️ The pitch DAC is the easy part — the output stage is not

One cent at 1V/oct is **833µV**, which over a 10V span is **83ppm**. That
number governs the whole chain, and the DAC contributes almost none of it:

| Source | Drift over 20°C | Cents |
|---|---|---|
| 16-bit LSB over 10V | — | 0.18 |
| AD5693R on-chip reference, 2ppm/°C | 40 ppm | 0.48 |
| **Discrete 1% resistors, 25ppm/°C** | **500 ppm** | **6.00** |
| Matched thin-film array, 1ppm/°C tracking | 20 ppm | 0.24 |
| Precision op-amp, 3µV/°C at gain 4 | — | 0.29 |
| Jellybean op-amp, 10µV/°C at gain 4 | — | 0.96 |
| *INA2134 receiver offset drift* — **cross-chassis only** | *see below* | *see below* |
| *INA2134 gain (resistor TCR) drift* — **cross-chassis only** | *see below* | *see below* |

#### ⚠️ The last two rows apply only to a chassis receiving CV over the link

A locally generated CV does not have them. A chassis taking its CV from an
upstream case over the CX link (`BUS.md` §2.7) puts an `INA2134` differential
receiver in the path, and that adds two terms the rest of this table does not
account for:

| Term | Converts as | Scales with signal? |
|---|---|---|
| **Offset drift** | `(µV/°C × 20) / 833µV` cents | No — fixed offset |
| **Gain drift** (on-chip resistor TCR tracking) | `ppm/°C × 20 / 83` cents | Yes — worst at full scale |

Sensitivity, to show which figure matters:

| If offset drift is | Cents | | If gain drift is | Cents |
|---|---|---|---|---|
| 2µV/°C | 0.05 | | 1 ppm/°C | 0.24 |
| 5µV/°C | 0.12 | | 2 ppm/°C | 0.48 |
| 10µV/°C | 0.24 | | 5 ppm/°C | **1.20** |

**Gain drift is the one to check first.** Offset drift costs at most a couple of
tenths of a cent across any plausible value, but gain drift at 5ppm/°C would
exceed every other term in the table combined. TI describe the on-chip
resistors as laser-trimmed with "excellent TCR tracking", which suggests the
1ppm/°C end — but that is a marketing phrase, not a number.

**⚠️ Both figures are owed from the datasheet.** They are not recorded here
because TI's site is unreachable from the environment these notes were written
in, and guessing part specifications has already produced two errors in this
project. Read them before relying on cross-chassis pitch accuracy.

**Separately: this table has never stated how its terms combine.** Summed
linearly it is a worst case; root-sum-square is the realistic figure for
independent drifts and is considerably kinder. Worth deciding which, since
adding terms makes the difference matter more.

Two rules follow, and they matter more than the DAC part number:

1. **Use a matched thin-film resistor network in the scaling stage, never
   two discretes.** It is *tracking* tempco that counts, not absolute —
   which is why an array specified at 25ppm absolute can still track to
   1ppm. Discretes throw away six cents over a warm-up and would waste
   every penny spent on the reference.
2. **Use a precision op-amp, not a TL072.** Offset voltage can be
   calibrated out; offset *drift* cannot. Note the 2.5V reference means a
   gain of about 4 to reach a 10V span, which multiplies the op-amp's
   input-referred drift by 4 rather than 2 — so this matters more here
   than it would with a 5V reference.

### Precision lands on MC-1, not VO-1

Counterintuitive, and worth stating plainly because the instinct is the
other way round:

- **VO-1 auto-tunes.** Its closed loop measures real oscillator frequency
  and corrects, absorbing DAC gain error, reference tolerance and slow
  drift. It needs monotonicity and short-term stability, little else.
- **MC-1 has no feedback at all.** Its V/OCT goes into someone else's VCO
  and whatever comes out is what you hear. The precision requirement lands
  on the interface module.

MC-1 should therefore carry a **user calibration routine** — output a known
code, measure with a meter, store gain and offset in the flash page already
reserved for settings. That removes the absolute-accuracy burden entirely
and leaves only drift, which calibration cannot fix. It is the reason to
spend the BOM on tempco rather than on initial accuracy.
- **MIDI I/O**: **3.5mm TRS, Type A** (MIDI Association-ratified standard,
  2018) — not 5-pin DIN (too big), not 2.5mm (non-standard minority format).
- **Bus connector**: **2×6 (12-pin)**, separate from the power header,
  populated on every module regardless of phase. Pinout and the module-side
  requirements (~220Ω series resistors on SDA/SCL, ESD, the 5V-tolerance
  check) are in `BUS.md` §2.
  - **VCC is DVCC, the bus's logic rail: 5V**, sourced by PS-1's bus board.
    It is a pull-up reference, not a module power rail — modules must **not**
    tie it to their local 3.3V, which would parallel every module's LDO
    output against every other's.
  - **The pull-ups live on the bus board, populated exactly once**, never
    per-module. Sizing and the reasoning are in `PS-1/CLAUDE.md`.
  - **⚠️ The module 3.3V LDO must track its input** — no long enable delay or
    soft-start. Every G0B1 I2C pin is FT, so 5V tolerance holds in operation,
    but absolute-max VIN is `VDD + 4.0V`; a delayed-start regulator is the
    only thing that would hold VDD near zero while DVCC is already at 5V.
    See `BUS.md` §2.

## MCU: STM32G0, range-wide

**`STM32G0B1CBT6`** (LQFP-48, 128K flash, USB FS device) is the default on
every module. Settled once for the range, per the bus decision that put an
MCU on every board.

### Why ST rather than RP2040

RP2040 was the standing suggestion, on the strength of its PIO and the fact
that MC-1 needs a USB device controller. It lost once the constraint was
relaxed to *one vendor, not necessarily one part* — which removes the thing
that was forcing a single USB-capable MCU onto all seven modules.

The deciding factors, strongest first:

- **I2C slave is the one peripheral every module depends on**, and ST's is
  much the better of the two. Hardware address matching, proper clock
  stretching, DMA. RP2040's is a Synopsys DesignWare block whose slave mode
  is its roughest edge — a risk to manage on every board in the rack rather
  than a feature.
- **STM32G0's system bootloader speaks I2C.** There is already an I2C bus
  with MC-1 as master and every module as a slave, so **MC-1 can reflash any
  module in the rack over the existing bus** — no per-module SWD header, no
  pulling modules to update them. The bus's nRESET line is what makes this
  survive a bad flash; see `BUS.md` §8. RP2040's bootrom is USB
  mass-storage and cannot do this. A real architectural feature falling out
  of a decision made for other reasons.
- **Single-chip: no external QSPI flash.** Seven fewer parts and footprints
  across the range, which matters most on VO-1, whose main board is already
  down to roughly 40 × 80mm.
- **LQFP at 0.5mm pitch** rather than QFN-56 at 0.4mm with a thermal pad —
  relevant if any of this is hand-built.

**⚠️ Layout constraint that comes with the I2C bootloader:** the
inter-module bus must land on a **bootloader-capable I2C peripheral and pin
set**, or the reflash-over-bus feature is lost. Per AN2606's STM32G0B1xx/0C1x
table that is **I2C1 or I2C2** — the pins within each are fixed, but there is
a choice of peripheral. **So put local peripherals (AS1115, MCP4728,
AD5693R) on I2C3**, which is not bootloader-capable and does not need to be;
that satisfies the two-port rule above and leaves both qualifying ports free
for the bus. Free if designed in, impossible to retrofit.

### What this gives up

- **No PIO.** An earlier draft of this decision leaned hard on it for
  VO-1's four quadrature encoders, and that was an overestimate: four
  encoders at human speed is on the order of 2,000 interrupts per second
  in total, which a 64MHz M0+ does not notice. Software quadrature decode
  is adequate and ordinary. VO-1's auto-tune frequency counter needs one
  timer in counter mode and one gating it, available on any G0. Where PIO
  would genuinely have been elegant is bit-banged waveform generation, if
  a later module (LF-1) wants it; DMA plus timers covers most of that.
- **Cost**: roughly £2 against RP2040's £1, so about £8 across the range.

### Per-module substitution is deliberately a late decision

One part number is the default because six of the modules are not yet
specced, and committing to a cheaper part before knowing their peripheral
needs is premature. The unused USB on those six costs well under £1 each.

Dropping a specific module to a smaller part (e.g. `STM32G031` in LQFP-32)
stays open, and is low-stakes precisely because it is the same family: same
HAL, same registers, same debugger, same toolchain. **Decide it per module
when that module's schematic is real, not now.**

**Two limits on that substitution**, verified against ST's CMSIS device
headers (ST and distributor domains are blocked from this environment; the
headers are definitive and reachable):

- **MC-1 can never be substituted.** No STM32G0 below G0B1 has a USB data
  peripheral at all. The trap is that the G071/G081 datasheets advertise a
  "USB Type-C Power Delivery controller" — that is UCPD, a PD *negotiation*
  block with no USB data path, and it cannot carry USB MIDI. Only G0B1/G0C1
  define a `USB_DRD_FS` peripheral.
- **A substitution gives up hardware encoder mode**, should a module ever
  want it instead of the software decode chosen above. Encoder interface mode
  exists only on TIM1/2/3/4, and TIM4 is present only on G0B1/G0C1 — a G031
  offers three, a G030 two. Not a blocker while decode stays in software, but
  it is the option that quietly disappears with the smaller part.

The dual-I2C requirement does **not** constrain the substitution — every G0
including the G030 has two I2C peripherals, and the G0B1 has three.

### The G0B1's internal DAC is not needed

An earlier draft of this decision flagged the G0B1's 12-bit 2-channel
on-chip DAC as a possible way to delete an external part on VO-1. The
MCP4728 settles it the other way: at four channels it covers any module's
parameter needs in a single part, where internal-two-plus-external-one
would still have been one external part *and* two different code paths for
what is conceptually one thing. The internal DAC stays unused.

## Non-volatile settings storage

Not previously recorded anywhere, and several modules need it:

| Module | Must survive power-off |
|---|---|
| MC-1 | MIDI channel, clock division |
| VO-1 | auto-tune calibration constants |

**Internal flash on the STM32G0**, not a separate EEPROM. This was an open
question while RP2040 was the candidate — writing settings there means
suspending XIP on the external flash the code is executing from, which is
workable but fiddly, and a small I2C EEPROM would have been the cleaner
answer. Single-chip removes the problem: in-application flash writes on a
G0 are ordinary.

Reserve a flash page for settings in the linker script from the start.
Retrofitting a settings area after the code has grown into it is a much
worse job than reserving it now.

## Circuit protection standard

Applies to every module's schematic. Cheap to design in now, painful to
retrofit after boards are fabbed — treat this as a checklist each module
must satisfy, not a per-module decision to re-derive.

- **Reverse power protection**: shrouded, keyed **2×8 (16-pin)** power header
  on **every** module (prevents backwards insertion mechanically). 2×8 is
  mandatory module-side, not just on the bus board: a 2×5 header physically
  omits the +5V pins, so a module cannot have one and still take the rail its
  3.3V LDO runs from. Budget ~8mm more board width than a 2×5 would have
  needed — worth checking on VO-1, whose main board is already down to roughly
  40 × 80mm. The CV and Gate pins come along with it, unconnected by default,
  which costs nothing and leaves a module able to tap the bus CV/Gate later
  without a connector change. The header is backed by a
  reverse-polarity protection circuit on each rail as a backstop for the
  "offset by one pin" case a keyed shroud doesn't catch.
  - **On ±12V**: simple series diodes (~0.7V drop, acceptable given the
    headroom) unless a specific module's circuit is voltage-sensitive enough
    to need an ideal-diode/PMOS approach instead.
  - **⚠️ On +5V, a 0.7V series diode is not acceptable** — use a Schottky or
    a PMOS ideal diode. The ±12V justification is a headroom argument and it
    does not carry over: +5V feeds a 3.3V LDO with only 1.7V to spend, and on
    a display module it is also the AS1115's supply, where the drop comes
    straight off the tightest electrical margin in the range. See the AS1115
    rail-budget warning under Common components.
- **Per-module overcurrent protection**: a resettable PTC polyfuse on each
  power rail, per module, so a fault on one module can't pull down the
  shared bus and affect its neighbours. Exact current rating TBD per
  module's actual draw. **Note +5V is now one of those rails on every
  module**, not only on ones with a display — and conversely each module's
  **+12V draw has fallen**, since its 3.3V domain moved off that rail.
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

## Every module defines its own safe state

The bus carries a **Panic** broadcast (`BUS.md` §6.7) and **every module must
implement it**. What "safe" means is the module's to define — the rack is too
heterogeneous for a central answer. MC-1's is a safe CV and a dropped gate; a
mixer's is levels at zero; a VCO's is something else again.

**So every module's `CLAUDE.md` must state its safe state**, as part of the
same checklist as the protection baseline above. A module that receives Panic
and does nothing is worse than one that does not implement it, because the rack
looks like it responded.

For most modules this is the same as the power-on state the MCP4728's EEPROM
already holds (pulse width at 50%, depths at zero). Where they coincide, say
so rather than define the same thing twice.

## Open items / not yet decided

- Spare 4HP allocation — spacing or a blind panel; too narrow for a module.
- EG-1's control set (fully continuous ADSR via 4 encoders vs. some fixed
  stages) — affects whether it needs a dedicated encoder sub-board.
- PS-1's own open items — whether it carries an MCU, and part numbers. See
  `PS-1/CLAUDE.md`. (The external brick, the 15V 3A input, the 16HP panel and
  the 16-module sizing basis are settled; the per-module +5V draw is owed a
  measurement.)
- `BUS.md`'s own open items — four register definitions owed (`COMMAND`
  opcodes, the bootloader magic value, the `INVENTORY` format, and the
  `CAPABILITIES`/`STATUS` bitfields), plus which of I2C1/I2C2 the bus takes.
  (DVCC, I2C pin tolerance, the bootloader address and pin sets, and Panic
  are all settled.)
- **Each module's safe state for Panic**, per the section above. MC-1 and VO-1
  both still owe theirs.
