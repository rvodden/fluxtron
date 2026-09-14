# FluxTron MC-1 — MIDI-to-CV/Gate Interface

Read `../CLAUDE.md` first for range-wide decisions (panel aesthetic, depth
rules, common components, bus architecture). This file only covers
decisions specific to MC-1.

## Role

The system's "front door." Converts MIDI (from USB-C or TRS MIDI IN) into
pitch CV, gate, velocity CV, and a clock/sync pulse for the rest of the
phase-1 analogue system. Also the module carrying the MIDI channel-select
control, since each MC-1 unit filters to one configured channel.

**MC-1 is also the digital bus master.** The phase-1 bus decision (see
`../CLAUDE.md`) makes MC-1 the only module receiving MIDI, so it decodes
CC/NRPN and pushes parameter values to every other module over the 4-pin
I2C bus. That is a firmware role, not extra hardware — the bus connector was
already specified on every module.

## Panel: 8HP

Layout, top to bottom:

1. FLUXTRON wordmark + "MC–1"
2. Jagged divider
3. Encoder (no LED ring — just the knob; sets channel *and* clock
   division, see below)
4. Two 2-digit 7-segment displays side by side — **CH** (left, MIDI
   channel 1–16) and **DIV** (right, clock division)
5. Jagged divider
6. USB-C bulkhead, centred on its own row
7. **IN** / **THRU** jacks (the TRS MIDI pair)
8. **VEL** / **CLK** jacks
9. **V/OCT** / **GATE** jacks
10. Jagged divider

Six jacks, two across, three rows.

4 mounting holes, symmetric, standard oval slots.

### Four-across resolved — the tight axis is now vertical

An earlier draft put PITCH/GATE/VEL/CLK in a single row of four, with
USB-C and both TRS jacks sharing another. Four across at 8HP is 10.16mm
centres, leaving roughly 1mm past the outer nuts and no room for a nut
driver (see the jack-density rule in `../CLAUDE.md`). That draft carried a
warning that MC-1 might not close at 8HP at all, with 16HP as the fallback.

**Resolved by going two across over three rows and giving USB-C its own
row.** Two across is the rule's "comfortable" case rather than its
three-across maximum, so MC-1 closes at 8HP and the 16HP fallback is off
the table.

The constraint moved rather than disappearing. Three jack rows at a
workable ~18mm pitch is around 54mm of the ~110mm usable panel height,
before the encoder's finger clearance (~25mm), the wordmark block, the
display row and four dividers. **Vertical space is now the binding
constraint, with only single-digit millimetres of slack.** Those numbers
are estimates from the mockup, not measured footprints — this is the item
most likely to force another re-layout, and it should be checked against
real footprints before the board is laid out.

### Jack labelling

The pitch output is labelled **V/OCT**. "PITCH" was the earlier draft;
plain "CV" was considered and rejected as ambiguous, since MC-1 emits two
CVs and VEL is the jack immediately beside it.

VO-1's matching pitch *input* carries the same label, so both ends of the
patch cable read alike. Settled range-wide — see `../CLAUDE.md`.

### ⚠️ IN and THRU are not visually distinct from the CV jacks

All six jacks are identical Thonkiconns, and nothing on the current
mockup marks IN/THRU as carrying TRS MIDI rather than CV. Patching a CV
cable into MIDI IN fails silently and is an easy mistake to make.

The range doc already calls for a MIDI DIN end-view icon. The jagged
divider band directly above the IN/THRU row is the natural home for it
and costs no vertical space, which matters given the budget above.

## Outputs: CLK and velocity are both kept

Both were queried against cheaper minimal MIDI-CV interfaces (e.g. Behringer
CM1A) that omit them — decision was to **keep both**:

- **CLK**: MIDI-clock-derived sync pulse, feeds LF-1's sync input (and any
  future clocked module). Not generic — tied directly to LF-1's existing
  sync-in jack. Division is switchable — see the next section.
- **Velocity**: kept for future use (e.g. EG-1 scaling amplitude/envelope
  amount from velocity), even though nothing in phase 1 reads it yet.

## CLK output: switchable clock division

MC-1 *receives* MIDI clock at the standard **24 PPQN**; it does not
generate it. Tempo belongs to whatever is upstream, so there is no tempo
control here and MC-1 cannot alter the MIDI clock rate in that sense. What
is adjustable is the division applied before the pulse reaches the CLK
jack: **one pulse is emitted every N ticks.**

Some division has to happen regardless — raw 24 PPQN at 120 BPM is 48 Hz,
an audio-rate tone rather than a sync pulse. Making the divisor adjustable
rather than fixed in firmware avoids committing blind to a number that has
to suit LF-1, which is not yet specified.

### Division table

15 values, ordered by rate so the encoder ramps monotonically from fast to
slow:

| Display | Musical value | N (ticks) | @ 120 BPM |
|---|---|---|---|
| `32` | 32nd | 3 | 16 Hz |
| `16` | 16th | 6 | 8 Hz |
| `t8` | 8th triplet | 8 | 6 Hz |
| `08` | 8th | 12 | 4 Hz |
| `t4` | quarter triplet | 16 | 3 Hz |
| `d8` | dotted 8th | 18 | 2.67 Hz |
| `04` | quarter | 24 | 2 Hz — **default** |
| `d4` | dotted quarter | 36 | 1.33 Hz — compound beat |
| `02` | half | 48 | 1 Hz |
| `d2` | dotted half | 72 | 0.67 Hz — 6/8 bar |
| `1b` | 1 bar | 96 | 0.5 Hz |
| `d1` | dotted whole | 144 | 0.33 Hz — 12/8 bar |
| `2b` | 2 bars | 192 | 0.25 Hz |
| `4b` | 4 bars | 384 | 0.125 Hz |
| `8b` | 8 bars | 768 | 0.0625 Hz |

Roughly eight octaves, 16 Hz down to one pulse per 16 seconds. Both ends
earn their place against LF-1: the fast end for rhythmic modulation, the
slow end for section-length sweeps, which is a main reason to sync an LFO
rather than free-run it.

**The `t` and `d` families are not ornament — they are what makes compound
time possible at all.** A binary-only table (1/2/4/8 bars and straight note
values) cannot express a 6/8 or 12/8 beat, whose pulse is a dotted quarter.
`d4` is that beat; `d2` is a 6/8 bar and `d1` a 12/8 bar.

### Display encoding

`t` and `d` are both cleanly renderable on 7-segment, so **the scheme does
not depend on the display having decimal points** — an open question that
would otherwise have been load-bearing. If the part does turn out to have
DPs, lighting one on the dotted values is optional garnish, nothing rests
on it. `t8`/`t4` stay visually distinct from `08`/`04` at 5mm digit height.

**Deliberately excluded: the 16th-note triplet** (N=4, 12 Hz), which would
sit between `16` and `t8`. It can only display as `t6`, where `6` means
"16th" — but the straight family already spells a 16th as `16`, so the code
would be inconsistent about its own family and misread every time. Adding
it would make the table 16 entries and the CC map a clean `cc >> 3`; that
was judged not worth the ambiguity.

### Control: the encoder push switch

One encoder serves two parameters. **Push toggles which one it edits**;
both values stay visible on their own display throughout, so there is no
hidden mode. Focus is shown by blinking the active pair (or by its decimal
point, if the part has one).

This uses hardware already specified and otherwise idle — the encoder is
wired with 5 leads including "switch ×2" and nothing previously used the
switch. A second encoder was never an option given the vertical budget.

The rejected alternative was push-and-hold + turn for division, plain turn
for channel. It needs no focus indicator at all, but is more awkward
one-handed.

### Firmware notes

- **No multiplication, ever.** Multiplying an incoming clock requires
  either predicting the next tick or delaying output by a full period.
  24 PPQN is a hard ceiling and `32` already sits at 16 Hz.
- **Pulse width**: `min(10 ms, period/4)`. A fixed 10 ms is right for most
  of the table, but `32` at 200 BPM has a 37.5 ms period — without the
  clamp the pulse approaches a DC level at the fast end.
- **Reset the divider counter on MIDI Start (0xFA)**, and handle Stop
  (0xFC) / Continue (0xFB). Without the reset the pulse lands off-beat
  after every transport stop. This is the classic bug in this circuit.
- **CC mapping**: `index = (cc * 15) >> 7` gives exactly 0–14 across the
  full CC range with no clamp needed. The naive `cc / 8` overruns the
  table.
- **Encoder end-stops, no wrap** — a stray flick should not take you from
  32nd notes to 8 bars.
- **`1b`/`2b`/`4b`/`8b` assume 4/4.** MIDI clock carries no time
  signature, so `1b` means 96 ticks = four quarter notes, which is a bar
  and a third in 3/4. The `d` entries are defined in ticks and are
  time-signature-agnostic, so the assumption is narrower than it looks.

## MIDI connectivity: USB-C + TRS, daisy-chainable

- **USB-C**: MC-1's MCU implements a **USB MIDI class-compliant device** —
  plug-and-play on Windows/Mac/Linux, no drivers. Needs the CC1/CC2 5.1kΩ
  pull-down resistors near the connector for proper host enumeration (data
  only, no PD negotiation needed).
- **TRS MIDI IN/THRU**: 3.5mm TRS Type A (see range-wide doc).
- **No dual-input/merge logic**: a given unit is fed by *either* USB
  (if enumerated) *or* TRS IN, never both — simpler firmware, no merge
  needed.
- **Daisy chain**: first MC-1 in a chain is fed by USB-C. Its firmware
  **actively regenerates** the incoming MIDI stream out its TRS THRU jack
  (a real retransmit, not a passive electrical loop, since the source is
  USB not a physical DIN/TRS line). A second MC-1's TRS IN patches from the
  first's THRU, filters to its own configured channel, and regenerates THRU
  onward for a third, etc. Lets one USB cable drive several independent
  FluxTron voice chains, each MC-1 tuned to a different channel via its
  encoder.

## Physical construction: three separate boards

This module ended up needing more than the range-wide "main board + flying
leads" default, because USB-C, the display, and the encoder all want
different panel-to-PCB depths and none of them match the 10mm jack-driven
main board:

- **Main PCB** (10mm standoff, set by the 6 Thonkiconn jacks): carries the
  jacks, power header, bus connector, MCU, DAC(s). No spacer washers needed
  since jacks reach their native depth directly.
- **Channel/division encoder**: off-board entirely, on flying leads (A, B,
  common, switch ×2 — 5 wires) back to the main PCB. Its own panel nut
  provides all the mechanical support; main-PCB keep-out zone behind it
  (16.1mm deep, per Bourns PEC16 body length — but see the range doc's
  warning that this figure is unverified). The switch pair is now used, for
  the channel/division focus toggle.
- **USB-C daughterboard**: separate small board, its own shallow standoff
  set by whichever connector is chosen (checked against 219320-0001 as a
  reference point — 8.8mm, i.e. deeper than the display, hence the separate
  board). Needs the CC1/CC2 pull-downs placed on this board near the
  connector.
- **Display daughterboard**: separate small board, very shallow standoff —
  see part choice below (3mm thick). Confirmed too different in depth from
  the USB-C connector to share one carrier, even though both are "small
  digital front-panel components." **Carries both displays** (CH and DIV):
  same part, same 3mm depth, so the depth-grouping rule puts them on one
  board. Its ribbon back to the main PCB grows by two digit-select lines.

Each daughterboard connects to the main PCB via a short header/jumper
(I2C for the display's driver, USB D+/D-/power for the USB board).

## Confirmed parts

- **Jacks**: Thonkiconn PJ301M-12 ×6 — V/OCT, GATE, VEL, CLK, plus the
  TRS MIDI IN/THRU pair. Two across, three rows.
- **Display**: **GS2020CB-G ×2** (CH and DIV), blue, per the range-wide
  indicator palette. Supersedes the Opto Plus OPS-D2010, which was only
  ever in the spec for its 3mm thickness and whose blue availability was
  never confirmed.
  Four digits total, well inside the AS1115's 8-digit capability — one
  driver, one I2C address, no extra silicon.
  **⚠️ Datasheet not yet on file — four figures are load-bearing and none
  is recorded here yet.** No public datasheet could be found for this part
  number from this environment, so nothing below is inferred from the part
  number and nothing should be until the datasheet is read:
  - **Thickness.** The display daughterboard exists as a separate board
    purely because the OPS-D2010 was 3mm and could not reach the panel from
    the jacks' 10mm plane. If GS2020CB-G is meaningfully thicker or
    thinner, that standoff moves with it; if it were ever to approach
    ~10mm it could share the main PCB and the daughterboard disappears.
  - **Common cathode or common anode.** Decides whether the standing
    AS1115 polarity question closes or stays open. (If the `C` in `CB`
    does denote common cathode, it closes — but that is a guess about a
    part number, not a fact, and must be read off the datasheet.)
  - **Forward voltage.** Decides whether MC-1 actually needs the local 5V
    LDO that blue segments imply (see `../CLAUDE.md`), and with it the
    extra dissipation on the PTC budget.
  - **Body width and digit height.** Two of these sit side by side on a
    40.64mm panel, so the pair plus their gap and the `CH`/`DIV` labels
    have to fit across 8HP — and vertical height feeds the panel budget
    that is already the tight axis.
  **Still to confirm**: AS1115 digit-drive polarity (common-anode vs.
  common-cathode) is compatible with this part before ordering.
- **Encoder**: Bourns PEC16, THT, off-board per above.
- **USB-C connector**: not yet finalised. 219320-0001 (Molex, 8.8mm) used
  as a reference depth point; still need to pick the actual part for the
  daughterboard.

## Circuit protection (beyond the range-wide baseline)

See `../CLAUDE.md` for the standard applying to every module (reverse power
protection, per-module PTC, output series resistors, input clamp diodes,
ESD on jacks, decoupling, bus ESD). MC-1 additionally needs:

- **MIDI opto-isolation**: keep an opto-isolator on the MIDI receive side
  even though the connector moved from 5-pin DIN to 3.5mm TRS — the point
  of the isolation (protecting MC-1's MCU from ground potential/fault
  conditions on whatever's plugged into MIDI IN) doesn't go away just
  because the connector shrank.
- **USB-C ESD protection**: ESD protection diodes on D+/D-, in line with
  USB-IF expectations for a front-panel-exposed port.
- **USB-C VBUS protection**: a small resettable fuse/current-limit on VBUS
  if MC-1 ever draws bench power from USB (optional fallback power path,
  not yet decided — see main open items list) — protects the host port
  from a fault on MC-1's side.

## Open items

- **Vertical panel budget** — the row-of-4 problem is resolved and MC-1
  closes at 8HP, but height is now the tight axis with only a few
  millimetres of slack, on estimated rather than measured footprints.
- **MIDI DIN icon** for the IN/THRU row, so those two jacks are
  distinguishable from the four CV jacks.
- **Bus master firmware** — parameter addressing/encoding over I2C is
  unspecified and is now load-bearing in phase 1.
- Exact USB-C connector part number for the daughterboard.
- **GS2020CB-G datasheet** — thickness, drive polarity, forward voltage
  and footprint are all unrecorded, and each feeds a decision already made
  elsewhere in this file. Highest-value item on this list.
- **A local 5V rail for the AS1115**, which blue segments force (see the
  range doc). Adds an LDO and its dissipation to MC-1's PTC budget — the
  10-pin power header carries no +5V.
- AS1115 drive-polarity check against the GS2020CB-G (was an open item
  against the OPS-D2010, which was common anode; the new part's polarity
  is not yet known).
- Whether the GS2020CB-G has decimal points — affects only the focus
  indicator and the optional dotted-value marker, not the division
  encoding itself, which was deliberately designed not to need them.
- **Free-running internal clock** (MC-1 as master when no MIDI clock is
  present) — genuinely useful, genuinely separate: it needs a tempo
  control, a start/stop affordance and probably tap, none of which fit the
  current vertical budget. Deliberately out of scope, not forgotten.
- Exact mounting-hole and connector coordinates (current panel mockups are
  illustrative, not yet tied to real PCB footprints).
