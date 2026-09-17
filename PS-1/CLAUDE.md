# FluxTron PS-1 — Power Supply and Bus Board

Read `../CLAUDE.md` first for range-wide decisions. This file only covers
decisions specific to PS-1.

## Role

Generates the system's rails and distributes them. **Settled: this is its own
module, not a section of MC-1.** MC-1 is already the most crowded board in the
range — USB-C, a display, six jacks, three separate PCBs — and it carries the
precision pitch DAC, which is the last thing that wants a switching converter
and a hot regulator next to it.

PS-1 owns two physically separate things that are easiest to design together:

1. **The supply** — one panel module converting external DC into ±12V (and
   +5V, see below).
2. **The bus board** — the passive backplane the whole rack plugs into,
   carrying both the Eurorack power headers and the 4-pin inter-module I2C
   bus.

## What having our own PSU buys

Three things that were previously worked around, and can now stop being:

- **A guaranteed +5V rail.** The range doc currently reasons "+12V is
  guaranteed on the power header; a +5V rail is not", and MC-1 therefore
  generates 5V locally with an LDO from +12V, dissipating **~560mW** — the
  worst thermal spot in the range, on the module least able to absorb it.
  Owning the PSU removes that premise. See "The +5V rail" below.
- **A fixed home for the I2C pull-ups.** They must be populated exactly once
  (see `../CLAUDE.md`), which previously meant nominating a module. On the
  bus board they are simply always present and always singular, regardless of
  which modules are installed, and MC-1 becomes electrically just another
  module.
- **A definition for the bus connector's VCC pin**, which has never had one.
  It is the pull-up reference, sourced by the bus board. Modules do **not**
  tie it to their local 3.3V — that would parallel every module's LDO output.

## The +5V rail

**PS-1 provides +5V.** It is nearly free once a switcher is in the design, and
it is standard on the 16-pin Eurorack header anyway.

**Every module fits a 16-pin header**, because every module's 3.3V LDO runs
from +5V (see `../CLAUDE.md`). An earlier revision here said this "does not
force a range-wide header change", with only display modules taking 16-pin and
the rest keeping 10-pin. That was written before 3.3V moved onto the 5V rail,
and once it did, there is no module that can keep a 10-pin header.

Two distinct loads share the rail, and conflating them is what caused the
confusion:

| Load | Which modules |
|---|---|
| 3.3V LDO input | **Every** module |
| AS1115 supply for a blue display | Only modules with one — currently MC-1 |

**⚠️ Portability caveat.** A FluxTron module that depends on bus +5V will not
work correctly in someone else's case, since many PSUs omit that rail — which
was main's original reason for keeping MC-1's 5V local. Recommended
resolution: on any module needing 5V, lay out **both** the bus-5V path and a
local LDO from +12V, with a **jumper selecting between them** and the LDO left
unpopulated by default. One jumper and one unpopulated footprint buys the
thermal win at home and portability elsewhere.

## Architecture

### Mains stays outside the case — settled

**External DC brick**, not an internal mains supply. An IEC inlet, fuse,
earth bonding and mains creepage/clearance inside a 70mm-deep Eurorack case is
a meaningful safety and compliance burden, and it is the one part of this
project where getting it wrong is dangerous rather than merely annoying. A
certified external brick moves all of that outside the enclosure and off our
PCB. It is also what most current Eurorack PSUs do.

**Use a locking DC connector**, not a bare barrel jack. A power connector that
can be knocked out mid-patch is a genuine annoyance.

### Brick voltage: 15V — and 12V does not work

**⚠️ A 12V brick cannot supply this rack.** There is no headroom to regulate
+12V from a 12V input, so the +12V rail would be the brick's raw output —
unregulated by us, carrying its ripple and its ±5% tolerance, and sagging
under load. It also leaves nothing to post-regulate the inverted rail with.
For a range that mandates linear LDOs on every module specifically to keep
switching noise away from the expo converter, that would undo the whole
policy at the source.

**15V is the sweet spot**, because the constraint pulls both ways: enough
headroom to post-regulate linearly, not so much that the linear stages cook.

| Brick | +12V rail | Linear loss at ~400mA |
|---|---|---|
| 12V | impossible | — |
| **15V** | **~2.5V headroom** | **~1.25W** |
| 18V | 5.5V headroom | ~2.75W |
| 24V | needs a buck stage first | two stages |

**⚠️ Dropout is tighter than 3V of nominal headroom suggests.** A −5% brick
is 14.25V, and reverse-polarity protection plus inrush limiting can take
another ~1V, leaving roughly **13.2V at the regulator input**. An LM317 needs
about 3V of dropout and would fall out of regulation. This wants a genuinely
low-dropout precision part — see below.

### The negative rail is the actual problem

Worth stating plainly, because the intuitive framing is "derive the lower
voltages", and −12V is not a lower voltage. It cannot be stepped down to from
anything; it has to be **inverted**. That is the whole difficulty of a
Eurorack PSU, and it is what sets the brick voltage above.

Proposed: an **inverting buck-boost** from +15V, set to about **−13.5V**
rather than −15V, so the negative LDO drops only ~1.5V and the negative rail's
linear loss roughly halves. The positive rail cannot get the same treatment
without adding its own buck, which is not worth it at 1.25W.

The inverting stage is a switcher, so linear post-regulation on the −12V rail
is **mandatory**, not optional.

### Rail generation

- **+12V**: 15V → low-dropout linear → +12V.
- **−12V**: 15V → inverting buck-boost → ~−13.5V → low-dropout linear → −12V.
- **+5V**: buck straight off the brick, no linear stage. It feeds digital
  only, so switcher ripple on it is harmless.

Regulator candidates, to verify rather than adopt: **LT3045 / LT3094**
(very low noise, ~0.4V dropout, but **500mA ceiling — check against the +12V
budget below, the margin is thin**), or **TPS7A47 / TPS7A33** (1A, more
current headroom). LM317/LM337 are ruled out by the dropout note above.

**Rail sequencing matters**: ±12V should come up together. Op-amps across the
rack can latch up if one rail appears well before the other.

### Where each voltage is derived

| Rail | Made in | Why |
|---|---|---|
| ±12V | PS-1 | Only place it can be — the inversion lives here |
| +5V | PS-1 | Centralising it deletes MC-1's 560mW local LDO |
| 3.3V | Each module | Keeps per-module isolation and local PSRR |

**3.3V stays on the modules** — a shared 3.3V rail would put every module's
digital noise onto every other module's DAC supply with no rejection stage
anywhere, and would break the per-module PTC rule. But **its input should
change from +12V to the bus +5V**: the same LDO then drops 1.7V instead of
8.7V. See the range doc for the range-wide consequence; it is what turns
VO-1's 0.4W regulator — sitting beside the precision expo converter its own
spec demands thermal separation for — into 78mW.

### Current budget (estimate, not measured)

Assumes every module takes 3.3V from the bus 5V rail, which moves the MCU
domain's current off +12V and onto +5V.

| Rail | Estimated draw | Design for |
|---|---|---|
| +12V (analogue only) | ~240mA | 600mA |
| −12V | ~200mA | 500mA |
| +5V (all 3.3V domains + MC-1's AS1115) | ~280mA | 750mA |

About **10W actual**. A **15V 2A** brick (30W) is ample; 1.5A would do, and
the headroom is for phase 2 rather than for its own sake. **Revise once
modules are measured, not estimated.**

Total linear dissipation in PS-1 is roughly 2W, which an 8HP board can shed
with decent copper and possibly a small heatsink on the positive regulator.

## Bus board

- **Ten 16-pin power positions** (84HP ÷ 8HP), shrouded and keyed.
  - **A bus segment tops out around 16 modules**, set by I2C's 400pF limit
    rather than by the 4-bit slot field: ~16 modules is 210–290pF, ~26 is at
    or over the limit. A larger case (6U, or 2×104HP) therefore wants **two
    segments with a bridge between them**, which is what the addressing
    scheme would have forced anyway. The two constraints agree.
- **Ten 2×6 I2C positions** on the same PCB, with the I2C traces routed away
  from the power traces — the range doc already requires the bus be kept clear
  of analogue sections, and one board makes that a layout task rather than a
  cable-dressing hope. See `../BUS.md` for the pinout.
- **⚠️ The bus board holds the slot addresses.** Each of the ten positions
  ties `A0–A3` to a different pattern of grounds — that is the whole of the
  geographic addressing scheme, and it is this board's job. Number them from
  the left, so a slot number and a physical position are the same thing.
  Modules pull these up internally, so the board only ever pulls down.
- **A dedicated master port at the left-hand end** — a **2×4 (8-pin)** header,
  narrower than the 2×6 slots, so neither can be plugged into the other.
  MC-1 uses it in chassis 0; CX-1 uses it in every chassis below. Because the
  master is not a numbered slot, **all 16 slot addresses stay available to
  slaves** — see `../BUS.md` §3.
- **A dedicated CX port** — PCA9615 plus an RJ45 — for chaining *down* to
  another chassis. Footprint on every board; populate only when this chassis
  actually chains further. A buffered link needs a transceiver at both ends,
  so this hardware is unavoidable; putting it on the bus board means the link
  costs no backplane position. The far end of the cable is a CX-1 module in
  the downstream chassis, reached at `0x30` on this segment.
  - **The PCA9615 suits 5V DVCC directly.** Its differential-side supply
    `VDD(B)` runs 3.0–5.5V with best operation at 5V, and its separate
    single-ended supply `VDD(A)` makes it an inherent level translator — so no
    translation is needed at this port whatever DVCC is set to.
  - This board carries no *upstream* port: a chassis is brought onto the bus
    by its own CX-1, not by its bus board.
- **Route the Eurorack bus CV and Gate lines.** Within a chassis they are the
  standard global pair. To reach a downstream chassis they leave through the
  **CX port**, which therefore also carries a **differential line driver for
  CV and a slew-limited buffer for Gate** (`../BUS.md` §2.7) — CV must be
  differential because inter-chassis ground offset lands straight on it at
  12 cents per 10mV. Populate alongside the PCA9615, only when the chassis
  chains further. The standard 16-pin header
  carries them and we are going 16-pin for the +5V anyway, so this is two
  traces on a board already being fabbed. If a cable-free global gate is ever
  wanted, the mechanism exists and is analogue and deterministic — which
  putting note events on I2C would not be (see `../BUS.md` §1). Module-side
  connection stays unpopulated or jumpered. A single global pair, so mono
  only. **Free now, a respin later.**
- **DVCC — the bus logic rail — is 5V**, generated here and carried on the
  bus connector's VCC pin. It is a pull-up reference, not a module supply.
- **The I2C pull-ups live here**, one pair only, never per-module. They have
  to be singular: one 4.7kΩ pair per module across eight modules is about
  590Ω in parallel, below the ~1kΩ floor the 3mA sink specification sets, and
  nothing would pull the bus low. A single **~2.2kΩ** pair, and the bus runs
  at **100kHz**.
  - **⚠️ At 5V DVCC, 400kHz is not available on a full segment.** The 3mA
    sink spec (VOL 0.4V) puts a *floor* of 1.53kΩ on the pull-up at 5V, while
    rise time puts a ceiling of `300ns / (0.8473 × Cb)`. Those cross at about
    **230pF** — above that there is no valid passive pull-up value at all. A
    realistic 84HP segment is ~150–250pF (10–15pF per module of pin,
    connector and stub, plus ~50pF/m of ribbon), so it sits right on that
    boundary. 3.3V DVCC would have a 0.97kΩ floor and keep 400kHz out to
    ~370pF; this is a real cost of the 5V choice.
  - **It does not bite, because of the bus ATTN line.** MC-1 reads a module
    only when it signals, so there is no round-robin polling load to spend
    bandwidth on. At 100kHz a parameter write is ~400µs, far below anything
    perceptible, and 2.2kΩ is valid across the whole capacitance range
    (floor 1.53kΩ, ceiling 4.7kΩ at 250pF).
  - 400kHz stays available only on a segment kept under ~160pF, which is
    roughly ten modules on a short ribbon. Worth knowing, not worth
    designing around.
  - **This corrects an earlier note here** claiming 2.2kΩ suited both speeds
    against 150–250pF. It does not; at 250pF and 400kHz the rise time is
    ~466ns against a 300ns limit.
- Bulk decoupling distributed along the rails, not lumped at one end.

## Protection

The range-wide protection standard in `../CLAUDE.md` is written for modules
*consuming* power. PS-1 is the source, so it needs a different list:

- **Input reverse polarity and input overvoltage** — the wrong brick will be
  plugged in eventually.
- **Soft start / inrush limiting.** Every module's bulk capacitance charges
  at once at power-on.
- **Per-rail current limiting** with sensible fault behaviour.
- **Output short-circuit protection** on each rail.
- Downstream faults are already covered by the per-module PTC rule.

## Open items

- **Verify the +12V regulator's current ceiling.** An LT3045 stops at 500mA
  against a ~600mA design target; either parallel two (they are designed for
  it) or take the TPS7A47 at 1A.
- **Panel width: 8HP or 16HP.** 8HP leaves 12HP spare and is probably enough
  given a brick does the AC-DC conversion; 16HP leaves 4HP and is comfortable
  for heatsinking. Decide once the thermal design is real.
- **Does PS-1 get an MCU?** The range rule puts one on every module, but PS-1
  has no MIDI-controllable parameters, so it is the one legitimate candidate
  for exemption. Against exemption: a cheap G0 reporting per-rail voltage and
  current over the bus would make brown-outs and shorts diagnosable instead
  of mysterious. Worth deciding deliberately rather than by default.
- **Panel layout** — DC inlet, power switch (or not), per-rail indicator LEDs.
- Whether the bus board is one 84HP PCB or two smaller ones linked, which is
  a fabrication-cost question.
- Exact part numbers for the buck stages and linear post-regulators.
