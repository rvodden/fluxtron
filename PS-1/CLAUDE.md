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
  guaranteed on the 10-pin header; a +5V rail is not", and MC-1 therefore
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

**This does not force a range-wide header change.** A bus board carries 16-pin
headers; a module with a 10-pin socket plugs onto the ±12V end of one
perfectly well, which is ordinary Eurorack practice. So:

- Modules that need +5V (currently only **MC-1**, for its AS1115) fit a
  **16-pin** header.
- Modules that do not (**VO-1** explicitly needs no 5V rail) keep **10-pin**.

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
- **Ten 4-pin I2C positions** on the same PCB, with the I2C traces routed
  away from the power traces — the range doc already requires the bus be kept
  clear of analogue sections, and one board makes that a layout task rather
  than a cable-dressing hope.
- **DVCC — the bus logic rail — is 5V**, generated here and carried on the
  bus connector's VCC pin. It is a pull-up reference, not a module supply.
- **The I2C pull-ups live here**, one pair only, never per-module. They have
  to be singular: one 4.7kΩ pair per module across eight modules is about
  590Ω in parallel, below the ~1kΩ floor the 3mA sink specification sets, and
  nothing would pull the bus low. A single **~2.2kΩ** pair suits both 100kHz
  and 400kHz against a realistic bus capacitance (~150–250pF for an 84HP
  ribbon plus stubs), and stays correct however many modules are installed.
  At 5V that pair draws ~2.3mA, comfortably inside the 3mA sink budget.
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
