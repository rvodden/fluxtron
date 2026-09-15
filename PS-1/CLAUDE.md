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

## Proposed architecture — NOT yet settled

Recorded as a recommendation so the shape is on paper. Every item here needs
confirming before anything is laid out.

### Mains stays outside the case

**Recommend an external DC brick**, not an internal mains supply. An IEC
inlet, fuse, earth bonding and mains creepage/clearance inside a 70mm-deep
Eurorack case is a meaningful safety and compliance burden, and it is the one
part of this project where getting it wrong is dangerous rather than merely
annoying. A certified external brick moves all of that outside the enclosure
and off our PCB. This is also what most current Eurorack PSUs do.

Suggested input: **24V DC**, 2A or better (~48W, against an estimated ~10W
actual draw — see below).

**Use a locking DC connector**, not a bare barrel jack. A power connector that
can be knocked out mid-patch is a genuine annoyance.

### Rail generation: switch, then linearly post-regulate

Consistent with the range's own rule that precision analogue does not sit
downstream of a switcher:

- 24V in → buck to roughly **±13.5V** → **linear post-regulators** to ±12V.
  The linears reject the switcher's ripple; candidates are LT3045/LT3094
  (very low noise, ~£4 each) or TPS7A47/TPS7A33, with LM317/LM337 as the
  cheap fallback.
- **+5V** comes straight off a buck with no linear stage — it feeds digital
  only (AS1115 segment current), so switcher ripple on it is harmless.

**Rail sequencing matters**: ±12V should come up together. Op-amps across the
rack can latch up if one rail appears well before the other.

### Current budget (estimate, not measured)

Eight modules at 64HP. Rough per-module figures: ~25mA on +12V for the MCU and
its 3.3V LDO, plus 20–40mA/+12V and 15–30mA/−12V of analogue, plus MC-1's
~80mA on 5V.

| Rail | Estimated draw | Design for |
|---|---|---|
| +12V | ~500mA | 1.5A |
| −12V | ~250mA | 1.0A |
| +5V | ~100mA | 0.5A |

About 10W actual against ~25W of capability, which leaves real headroom for
phase 2 rather than for its own sake. **Revise once modules are measured, not
estimated.**

## Bus board

- **Ten 16-pin power positions** (84HP ÷ 8HP), shrouded and keyed.
- **Ten 4-pin I2C positions** on the same PCB, with the I2C traces routed
  away from the power traces — the range doc already requires the bus be kept
  clear of analogue sections, and one board makes that a layout task rather
  than a cable-dressing hope.
- **The I2C pull-ups live here**, one pair only. ~2.2kΩ suits both 100kHz and
  400kHz against a realistic bus capacitance.
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

- **Confirm external brick vs. internal mains.** Everything above assumes the
  brick. This is the decision that shapes the rest of the module.
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
