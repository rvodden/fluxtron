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
  guaranteed on the power header; a +5V rail is not" — and MC-1 therefore
  generated 5V locally with an LDO from +12V, dissipating **~560mW**, the
  worst thermal spot in the range, on the module least able to absorb it.
  Owning the PSU removed that premise, and the range doc no longer reasons
  that way. See "The +5V rail" below.
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

**That ~1V is the figure to attack, and Protection now does.** It assumed a
series diode. A PMOS ideal diode recovers most of it, putting the −5% case
nearer **14.0V** and buying back most of a volt of regulator headroom for the
cost of one FET and a controller. The dropout constraint and the dissipation
constraint are the same constraint pulling opposite ways — a −5% brick starves
the regulator while a +5% one cooks it — and 15V sits between them with little
room, so a volt reclaimed at the input is worth more here than it looks.

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
  only, so switcher ripple on it is harmless *on this rail* — but note it is
  the input to every module's 3.3V LDO, and so reaches the pitch DACs at one
  remove. See the open item on it.

Regulator candidates. The 16-module ceilings below decide more of this than
the earlier eight-module ones did:

- **⚠️ LT3045 / LT3094 (500mA) no longer fit the +12V rail** at a 700mA
  design target. Very low noise and ~0.4V dropout, so still attractive —
  but only as a paralleled pair (they are designed for it). On −12V at
  600mA the same problem applies.
- **TPS7A47 / TPS7A33 (1A)** clear both ceilings as single parts.
- LM317/LM337 are ruled out by the dropout note above.

**Rail sequencing matters**: ±12V should come up together. Op-amps across the
rack can latch up if one rail appears well before the other. This is only
two-thirds of the problem — see the three-rail sequencing open item.

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

### Sizing basis: 16 modules, not phase 1's eight

**PS-1 is sized for a full bus segment — 16 slave modules — not for the eight
of phase 1.** The bus gives out 16 slot addresses (`../BUS.md` §3), so 16 is
the number the rack can grow to without a new addressing scheme, and a supply
is the worst thing in the rack to have to redesign later.

Two things follow that are easy to conflate, so keep them apart:

- **The supply rating is 16 modules.** That is this section.
- **The backplane position count is set by the chassis**, and stays at ten for
  the 84HP case. Sixteen 8HP modules is 128HP and does not fit; a 16-slot
  backplane is a later board for a larger chassis. Whether such a chassis is
  served by one 16-slot segment or by two smaller ones with a bridge (which
  is what the bus-board section below recommends) is **still open — and does
  not change the power budget either way**, since the module count is the
  same.

Assumes every module takes 3.3V from the bus 5V rail, which moves the MCU
domain's current off +12V and onto +5V.

#### ⚠️ The per-module +5V figure is unresolved, and it is the one that matters

Two numbers are in circulation and they differ by 3×:

| Basis | Per module | × 16 |
|---|---|---|
| Datasheet-order (G0B1 at 64MHz ~8–12mA, two DACs ~3mA, pull-ups ~2mA) | ~18mA | 288mA |
| Implied by VO-1's "roughly 0.4W off +12V" (0.4W ÷ 8.7V) | 46mA | 736mA |

Neither was measured. At 16 modules this is the difference between a 750mA
regulator and a 1.5A one, so **measure one populated module before ordering
the +5V stage.** It is the highest-value bring-up measurement in the project.

#### Per-rail budget at 16 modules

Display modules assumed to be four rather than MC-1 alone, since a 16-module
rack will not have only one.

| Rail | Load | Low estimate | High estimate |
|---|---|---|---|
| **+5V** | 3.3V LDO inputs ×16 | 288mA | 736mA |
| | AS1115 displays ×4 (dimmed / full) | 120mA | 320mA |
| | **Total** | **~410mA** | **~1050mA** |
| **+12V** | analogue ×16 | 288mA | 400mA |
| | blue indicators ×16 @ ~3mA | 48mA | 48mA |
| | **Total** | **~336mA** | **~448mA** |
| **−12V** | analogue only, no digital load | **~240mA** | **~352mA** |

**+12V is not "analogue only"** despite the old label — the range-wide
blue-LED rule puts every indicator on it through a transistor.

#### Design-for ceilings

| Rail | Design for | Note |
|---|---|---|
| +12V | 700mA | was 600mA |
| −12V | 600mA | was 500mA |
| **+5V** | **1.5A** | **was 750mA — the rail that broke** |

**Everything that scales badly to 16 modules scales badly on +5V**, the rail
that exists only because the 3.3V LDOs moved onto it. ±12V were already sized
generously enough to absorb the change. That trade is still right — it buys
back ~3.2W of scattered module heat — but it concentrates the current into one
centrally regulated rail feeding LDOs that have 1.7V to spend, and that is the
part to design carefully rather than the easy one.

#### What the ceilings cost at the brick

| | Brick draw | Linear dissipation in PS-1 |
|---|---|---|
| Realistic simultaneous draw (+12V 400mA, −12V 300mA, +5V 700mA) | ~1.0A (15W) | ~1.8W |
| Sum of design-for ceilings | **1.89A (28.4W)** | **3.31W** |

The ceilings are conservative-on-conservative — they exist so no individual
regulator saturates, and all three will not peak together. They are still the
right basis while five phase-1 modules and eight hypothetical ones are
unspecified.

### Brick: 15V **3A** (45W)

**Supersedes "15V 2A is ample; 1.5A would do."** That was written against the
eight-module budget and does not survive 16.

**⚠️ Bigger means more current at 15V, never more volts.** The obvious reading
of "a bigger brick" is the one that makes PS-1 worse:

| Brick | +12V regulator dissipation at 700mA |
|---|---|
| **15V** | **2.4W** |
| 18V | 4.6W |

Current headroom at the same voltage adds no dissipation anywhere; voltage
headroom nearly doubles it in the part that already sizes this module's
heatsink — and 4.6W is where 8HP would genuinely stop working, perpendicular
board or not. The 15V choice made on dropout grounds above is unchanged.

Why 3A rather than 2A:

1. **Derating.** 1.89A on a 2A brick is 95% continuous — hot, and short-lived.
   On a 3A brick it is 63%. This alone decides it, independently of whether
   the rack ever reaches the ceilings.
2. It costs nothing thermally, per the table above.
3. It leaves room for the +5V question to resolve upward, which is the open
   direction.

**⚠️ Check the brick starts into the load.** Sixteen modules is roughly 1.6mF
of bulk capacitance charging at once, and plenty of bricks with foldback
current limiting will hiccup indefinitely into that whatever their
steady-state rating. The soft-start below is what makes this work, so the two
are selected together, not separately.

**USB-C PD considered and rejected — do not re-propose.** 15V @ 3A is a
standard PD fixed profile, so 45W PD supplies are commodity, certified and
better-sourced than a 15V barrel brick (15V being a less common brick voltage
than 12/19/24V), and a standalone sink controller (STUSB4500 class) needs no
MCU. It loses on two counts: USB-C is exactly the connector that unplugs when
nudged, against this module's explicit locking-connector requirement, and a
failed negotiation leaves the rack silently on 5V.

### Panel width: 8HP, on a PCB mounted **perpendicular** to the panel

3.31W of linear dissipation plus buck losses is around 5W in the module, with
2.4W of it worst-case in the positive regulator alone. **That does not settle
the width — it settles the orientation**, and an earlier revision here got
that wrong by declaring 8HP dead. A heatsink was always needed, at any width:

| +12V regulator at 2.4W, 40°C case ambient | Sink | Junction |
|---|---|---|
| Bare DPAK on a copper pour (~30°C/W) | 112°C | ~118°C — **fails** |
| Large pour with vias (~20°C/W) | 88°C | ~94°C — marginal |
| Small extruded heatsink (~8°C/W) | 59°C | ~65°C |
| 100mm vertical extrusion (~5°C/W) | 52°C | ~58°C |

So the question was never "can 8HP shed 5W", it was "is there room for the
heatsink and the parts". Turned perpendicular, there is:

| 8HP slot, ~105mm usable height, 70mm case depth | Board area |
|---|---|
| Parallel to the panel (the range default) | 43cm² |
| **Perpendicular** | **68cm² (1.6×)** |

Across the 40.64mm width, a 1.6mm board plus clearances plus a 15mm extrusion
comes to 20.6mm — **half the slot, with 20mm spare**. The fins run vertically
through the full 70mm of case depth, which is the orientation natural
convection wants anyway.

**⚠️ This is a deliberate exemption from the range-wide parallel-PCB rule, not
an oversight — do not "correct" it.** That rule exists because pots, encoders
and jacks are built for a parallel board and would otherwise need right-angle
variants of everything. **PS-1 has none of those.** Its panel carries a DC
inlet (a chassis-mount part on flying leads regardless of orientation),
possibly a switch, and THT indicator LEDs — which the range doc already
exempts from depth constraints because their leads are bent to reach. The
rule's justification simply does not reach this module. Recorded in
`../CLAUDE.md` alongside the rule itself.

What perpendicular costs, and it is not nothing:

- **Mechanical support has to be designed.** Every other module is held by its
  jack nuts; PS-1 has no jacks. It needs a front bracket to the panel and
  probably a rear standoff — a part no other module in the range needs.
- **It may resolve the bus-board feed for free.** A card extending back from
  the panel arrives at the backplane edge-on, so a right-angle header or card
  edge could mate directly instead of running a high-current ribbon. Attractive
  at ~2A aggregate, but contingent on the backplane's mounting geometry, which
  is not yet specified. See the open items.
- **16HP stays the fallback** if the mechanical work or the layout does not
  close. Nothing above depends on 8HP; it is the better answer if it fits.

**The +12V regulator ceiling is independent of all this.** At a 700mA design
target an LT3045 (500mA) is out unless two are paralleled. Take the TPS7A47 at
1A, or parallel a pair.

## Bus board

- **Ten 16-pin power positions** (84HP ÷ 8HP), shrouded and keyed. PS-1 at
  8HP consumes one position's worth of panel width, leaving **nine usable
  module positions** in 84HP — phase 1 needs eight. (At the 16HP fallback it
  is two positions and eight remain, which still fits.) The supply is rated
  for 16 modules (see the sizing basis above); the *position count* is set by
  this chassis, and a 16-slot backplane is a later board.
- **⚠️ Heavy copper on the +5V pour — at least 2oz, a pour and not a trace.**
  At 280mA this did not matter; at up to 1.5A it does, because +5V feeds LDOs
  with 1.7V of headroom and, on a display module, the AS1115 directly.

  | 5V conductor, ~600mm | Drop at 1.5A |
  |---|---|
  | 1oz, 2.5mm wide | 177mV |
  | 2oz, 5mm wide | 44mV |
  | 2oz, 10mm wide | 22mV |

  The 1oz case eats an eighth of a display module's entire driver margin in
  backplane resistance alone, and it is worst at the far slot — so a display
  module at the end of a long board is the range's worst-case rail budget.
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
  **CX port**, sent **impedance-balanced** (`../BUS.md` §2.7): a matched
  **0.1%** series resistor pair per signal, and **no driver IC** — the
  receiving end does the rejecting. 0.1% rather than 1% because the impedance
  match sets system CMRR, and 1% lands exactly on the margin needed for the
  worst inter-chassis ground offset. All four Cat5 pairs are then
  differential, which is what lets the cable's **shield** serve as the
  common-mode reference: no signal return current flows in it. Use
  **shielded** RJ45 jacks. Populate alongside the PCA9615, only when the
  chassis chains further. The standard 16-pin header
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
- Bulk decoupling distributed along the rails, not lumped at one end. With
  16 modules' bulk capacitance downstream this is a soft-start question as
  much as a decoupling one; see Protection.
- **⚠️ How PS-1 itself lands on the board is not yet specified.** It is the
  source, so it cannot plug into a 16-pin slot as a sink — it needs its own
  feed connector sized for the full rail currents (~2A aggregate), or the
  supply and backplane share one PCB. Open; see below.

## Protection

The range-wide protection standard in `../CLAUDE.md` is written for modules
*consuming* power. PS-1 is the source, so it needs a different list:

- **Input reverse polarity — a PMOS ideal diode, not a series diode.** This
  is forced by the dropout budget this file computes above: a −5% brick plus
  protection and inrush losses already lands near 13.2V at the regulator
  input, and a 0.7V series diode is most of what makes that tight. An ideal
  diode recovers nearly all of it, and is the difference between needing a
  sub-0.5V-dropout regulator and not. At ~2A it also saves over a watt of
  heat in a module that has no watts spare.
- **Input overvoltage** — the wrong brick will be plugged in eventually.
- **Soft start / inrush limiting.** Every module's bulk capacitance charges
  at once at power-on.
- **Per-rail current limiting** with sensible fault behaviour.
- **Output short-circuit protection** on each rail.
- Downstream faults are already covered by the per-module PTC rule.

## Open items

- **⚠️ Measure one populated module's 3.3V domain.** The 18mA-vs-46mA spread
  above sets the +5V regulator rating and nothing else resolves it. Highest-
  value measurement in the project; VO-1 is the obvious candidate.
- **One 16-slot segment or two bridged segments** for a larger chassis. The
  bus-board section recommends two; sizing the supply for 16 modules assumes
  the rack reaches that count either way, so this is a bus decision rather
  than a power one — but the two documents should agree before a backplane
  is laid out.
- **How PS-1 feeds its own bus board** — a dedicated source connector, one
  shared PCB, or a direct edge-on mate now that the supply card is
  perpendicular. The third is the most attractive at ~2A and needs the
  backplane's mounting geometry pinned down first. See the bus-board section.
- **PS-1's mechanical mounting.** A perpendicular card has no jack nuts
  holding it; it needs a panel bracket and probably a rear standoff. This is
  what 8HP is contingent on, and it is the item that would send the module
  back to 16HP.
- **Rail sequencing across three rails, not two.** The existing note only
  covers ±12V rising together for op-amp latch-up. But +5V is a buck straight
  off the brick and will come up first, while `../CLAUDE.md` requires each
  module's 3.3V LDO to track its input with no soft-start (for DVCC
  5V-tolerance). Every MCU therefore boots and starts driving DAC outputs
  into op-amps whose rails do not exist yet. Probably benign; currently
  nobody's stated problem.
- **+5V switching noise reaches the precision DACs.** The range-wide "no buck
  near precision analogue" rule keeps a switcher off each module, but PS-1's
  +5V buck now feeds every 3.3V LDO, and that 3.3V rail supplies the AD5693R
  on VO-1 and MC-1. Realistically HF noise rather than pitch error — a DC
  pitch CV does not shift from 1MHz ripple — but it wants LC filtering at
  each module's 5V input, and it makes the LDO's *high-frequency* PSRR a
  selection criterion. Confirm rather than assume.
- **Per-module PTC ratings are now derivable** from the budget above: ~100mA
  hold on ±12V, ~100mA on +5V, and MC-1 separately at ~150mA for its display.
  Choose low-resistance parts on +5V — the PTC's series resistance comes off
  the same margin as the reverse-polarity element.
- **Does PS-1 get an MCU?** The range rule puts one on every module, but PS-1
  has no MIDI-controllable parameters, so it is the one legitimate candidate
  for exemption. Against exemption: a cheap G0 reporting per-rail voltage and
  current over the bus would make brown-outs and shorts diagnosable instead
  of mysterious. Worth deciding deliberately rather than by default.
- **Panel layout** — DC inlet, power switch (or not), per-rail indicator LEDs.
- Whether the bus board is one 84HP PCB or two smaller ones linked, which is
  a fabrication-cost question.
- Exact part numbers for the buck stages and linear post-regulators.
