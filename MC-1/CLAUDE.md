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
3. Encoder (no LED ring — just the knob; turns to set clock division,
   push to reach channel — see below)
4. Two 2-digit 7-segment displays side by side — **CH** (left, MIDI
   channel 1–16) and **DIV** (right, clock division)
5. Jagged divider
6. USB-C bulkhead, centred on its own row
7. **IN** / **THRU** jacks (the TRS MIDI pair)
8. **VEL** / **CLK** jacks, with the **CLK indicator LED** in the centre
   gap, offset toward CLK
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

### CLK indicator LED

A blue THT LED beside the CLK jack, flashing on each emitted clock pulse.
It answers two questions the panel otherwise cannot: whether MIDI clock is
arriving at all, and what rate is actually coming out — the latter being a
check on the DIV setting that does not require reading the display.

**Placement costs no vertical space**, which matters given the budget
above: it sits in the centre gap on the existing VEL/CLK row rather than
taking a row of its own. **Offset toward CLK, not centred** — an LED
equidistant between VEL and CLK reads as belonging to both.

No icon accompanies it — icons are dropped range-wide. None is needed: the
LED sits beside a jack already labelled CLK, and proximity says the rest.

- **Driven from its own MCU GPIO, not the AS1115.** MC-1 has an AS1115 for
  the displays and it has spare capacity, but hanging the clock LED off it
  would multiplex the indicator at the display refresh rate and bind its
  brightness to the displays' global intensity — which is already being
  set as a compromise between readability and LDO dissipation. A GPIO
  keeps clock timing independent of display refresh and lets the LED be
  dimmed on its own terms. +12V through a series resistor and a small
  NPN/MOSFET, per the range-wide blue-LED rule.
- **The LED needs its own on-time, longer than the jack pulse.** The CLK
  output is `min(10ms, period/4)`; 10ms of light reads as a flicker, not a
  blink. Use **`min(50ms, period/2)`** for the LED. That gives a crisp 50ms
  flash at `04` (10% duty at 120 BPM) and still half-duty rather than a
  solid glow at `32`, where the period is only 62.5ms.
- **⚠️ Dark must not mean "stopped" when it means "slow".** At `8b` the
  output is one pulse every 16 seconds, so a flash-only LED is
  indistinguishable from no clock at all for 15.95 of them. **Hold the LED
  at a low PWM duty whenever MIDI clock is being received**, and drive it
  to full for the flash. Dark then means genuinely no clock; dim means
  clock present but between pulses; bright is the pulse itself. The GPIO
  drive is already there, so this costs nothing but firmware.

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

**This was going to be solved by a MIDI DIN end-view icon, and icons are
now dropped range-wide — so it is open again.** The icon-free fix is a
plain word, which the aesthetic already sanctions: **MIDI** heading that
jack row. Either as a small section label above IN/THRU, or by extending
the labels themselves to `MIDI IN` / `MIDI THRU` if they fit the column
width in Kode Mono.

The section-label form is preferable if it can be fitted into the band
already separating the USB-C row from the jacks, since MC-1's vertical
budget has no room for a new row. Unresolved — it needs deciding against
the real panel layout, not here.

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

### Control: division on the knob, channel behind the push

One encoder serves two parameters, and they are **not** peers:

- **Turning the knob sets clock division.** This is the resting state.
- **Pushing gives access to MIDI channel**, which then **times out back to
  division** after a few seconds without rotation.

Both values stay visible on their own display throughout, so there is no
hidden mode — only a hidden *focus*, which is shown by blinking the active
pair (or by its decimal point, which the GS2022CB-B does have).

**Division is the default because it is the parameter that actually gets
used.** Channel is set once when the rack is patched and then left alone;
division is a performance control, changed whenever the synced LFO should
move at a different rate. Putting the frequent operation on the bare turn
and the rare one behind a deliberate press is the right way round.

It is also the right way round on blast radius, which is the argument that
makes the timeout worth having. **A stray knock on the knob is survivable
if it lands on division and disastrous if it lands on channel** — a wrong
division shifts a modulation rate, a wrong channel silences the voice
entirely, and the second failure looks like broken hardware rather than a
misturned knob. Returning to division automatically means the module is
never left resting on the dangerous parameter.

This uses hardware already specified and otherwise idle — the encoder is
wired with 5 leads including "switch ×2" and nothing previously used the
switch. A second encoder was never an option given the vertical budget.

The rejected alternative was push-and-hold + turn for the secondary
parameter. It needs no focus indicator at all, but is more awkward
one-handed, and the timeout above gives most of the same safety.

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

- **USB-C**: MC-1's MCU — an **STM32G0B1CBT6**, per the range-wide choice
  in `../CLAUDE.md` — implements a **USB MIDI class-compliant device** —
  plug-and-play on Windows/Mac/Linux, no drivers. Needs the CC1/CC2 5.1kΩ
  pull-down resistors near the connector for proper host enumeration (data
  only, no PD negotiation needed).
  - **No crystal needed for USB.** The G0B1 does crystal-less USB from the
    HSI48 with CRS trimming it against USB start-of-frame, which is well
    inside the ±0.25%% full-speed device tolerance. The HSI is also fine
    for 31250-baud MIDI UART, and MC-1 derives its CLK timing from the
    incoming MIDI clock rather than from its own oscillator, so absolute
    frequency accuracy is not load-bearing anywhere on this module. A
    crystal remains cheap insurance if bring-up suggests otherwise.
  - **USB is why MC-1 specifically needs the G0B1** rather than a smaller
    G0 — the G031/G071 parts have no USB controller at all. Do not be
    misled by the G071/G081 datasheets advertising a *USB Type-C Power
    Delivery controller*: that is UCPD, a PD negotiation block with no USB
    data path, and USB MIDI cannot run on it. Only G0B1/G0C1 define a
    `USB_DRD_FS` peripheral. This is the module that sets the range-wide
    part choice; the others inherit it.
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
  jacks, power header, bus connector, STM32G0B1 MCU, DAC(s). No spacer
  washers needed since jacks reach their native depth directly.
- **Channel/division encoder**: off-board entirely, on flying leads (A, B,
  common, switch ×2 — 5 wires) back to the main PCB. Its own panel nut
  provides all the mechanical support. **No main-PCB keep-out is needed**:
  the PEC11R sits 6.5mm behind the panel, wholly in front of the 10mm main
  PCB, which retires the cutout worry the PEC16 carried. The switch pair is
  now used: push reaches the MIDI channel, which the knob otherwise
  does not touch.
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
- **Display**: **Guangcai GS2022CB-B ×2** (CH and DIV) — 0.2" (5.08mm)
  dual-digit SMD 7-segment, blue, common cathode, **black** reference
  surface. Supersedes the Opto Plus OPS-D2010, which was only ever in the
  spec for its 3mm thickness and whose blue availability was never
  confirmed. Datasheet:
  `../datasheets/GS2022_Guangcai_7seg_revA_2022-09-20.pdf`.

  **Black reference surface, not grey.** The series offers both (`-B`
  black, `-G` grey). Black makes unlit segments disappear into the matte
  black panel, so the display reads as blue characters floating on the
  panel rather than as a grey rectangle stuck to it — the right call for
  the cold palette. No electrical difference between the two.

  **Part number decode**, since ordering the wrong variant is easy: the
  datasheet is headed `GS2022A/CX-X`, where `A`/`C` selects common anode
  or common cathode, the next letter is the emitting colour and the `-X`
  suffix is the reference-surface colour. So `GS2022CB-B` = common
  **C**athode, **B**lue, **B**lack surface — note the `B` means different
  things in the two positions. The common-anode `GS2022Ax` is the wrong
  part here; see polarity below.

  Confirmed figures, all four of which were previously open:

  | | |
  |---|---|
  | Body | 12.00 × 10.00 × **3.20mm** |
  | Digit height | 5.08mm (0.2") |
  | Pins | 10, 2.54mm pitch, 10.16mm span |
  | Polarity | common cathode (`GS2022Cx`) |
  | Blue die | InGaN/GaN, λd 460nm |
  | Vf @ 10mA | **typ 3.00V, max 3.80V** |
  | Luminous intensity | 120–180 mcd @ 10mA |
  | Abs max | IF 30mA, Pd 100mW, Ipeak 150mA (0.1ms, 1kHz) |
  | Decimal point | present on both digits |
  | Reflow | 245 ±5°C, 260°C max, ≤2 passes |

  What each one settles:

  - **3.20mm thickness** — near enough the OPS-D2010's 3mm that the
    display daughterboard and its shallow standoff are unaffected. No
    board-stack change.
  - **Common cathode is what the AS1115 wants** (its segment drivers
    source and its digit drivers sink), so picking the `C` variant closes
    the polarity question that was open against the common-anode
    OPS-D2010. Segment pinout: A=3, B=9, C=8, D=6, E=7, F=4, G=1, DP=2;
    DIGIT1=10, DIGIT2=5.
  - **Vf confirms the 5V rail is mandatory, and makes it tighter than
    expected** — see the dedicated note below.
  - **12.00 × 10.00mm** — two side by side is 24mm of a 40.64mm panel,
    comfortable horizontally. 10mm of height for the pair, which the
    vertical budget has to carry.
  - **Decimal points exist**, so the focus indicator can use a DP rather
    than blinking. The division encoding was deliberately designed not to
    need them, so this is a convenience, not a dependency.

### ⚠️ The 5V rail is tighter than "blue needs 5V" suggests

The AS1115 sources segment current from its own V+, so V+ must clear the
segment forward voltage plus the segment-driver and digit-driver drops in
series. With a **worst-case Vf of 3.80V** and the AS1115's 5.5V absolute
maximum, running it at 5V leaves only **1.2V** for both drivers at the
worst-case part. Typical parts (3.00V) leave a comfortable 2.0V.

**Verify the AS1115's driver drops at the intended segment current before
committing the board.** If worst-case parts prove dim, the options are to
run V+ nearer 5.5V, or to accept the typical case and bin.

Two further consequences:

- **I2C level shifting.** With the AS1115 at 5V and the MCU at 3.3V, the
  bus cannot simply be tied together — pull-ups to 5V would over-voltage
  the MCU pins, and the AS1115's input thresholds at V+=5V may sit above
  what a 3.3V driver guarantees. Budget for a MOSFET level-shifter pair on
  the local I2C, or confirm the AS1115's VIH allows 3.3V direct drive.
- **LDO dissipation, and dimming as a thermal lever.** Multiplexed, one
  digit lit at a time, all 8 segments at 10mA is ~80mA from the 5V rail —
  7V × 80mA ≈ **560mW** in a linear LDO from +12V. That wants a SOT-223 or
  DPAK part and a real thermal pad, and it goes on the PTC budget. At
  120–180 mcd these are very bright and will almost certainly be run well
  below full intensity, which cuts the dissipation proportionally — so the
  AS1115's intensity setting is a power decision here, not only a visual
  one. No switching regulator: MC-1 generates precision pitch CV, so the
  range-wide linear-only rule applies to it as much as to VO-1.
  **Still to confirm**: AS1115 digit-drive polarity (common-anode vs.
  common-cathode) is compatible with this part before ordering.
- **Parameter DAC**: **MCP4728** (12-bit, 4-channel, I2C) — velocity CV
  on one channel, three spare. On the local I2C port alongside the AS1115.
  **Sit it on the 3.3V side of the AS1115 level shifter**, not the 5V
  side: the AS1115 needs 5V only because blue segments demand it, and
  there is no reason to drag the DAC up with it.
- **Pitch DAC**: **AD5693R** (16-bit, I2C, 2.5V on-chip reference at
  2ppm/°C) for the V/OCT output. On the local I2C bus alongside the
  MCP4728 and the AS1115 — and, like the MCP4728, on the **3.3V side of
  the level shifter**. **Firmware must write the pitch DAC before raising
  gate**, so a note-on cannot skew against a parameter update sharing the
  bus; see `../CLAUDE.md`.
  **MC-1 is the module where pitch accuracy actually matters** — it has no
  feedback loop, unlike VO-1's auto-tune. It needs a **user calibration
  routine** storing gain and offset in the reserved settings flash page.
- **Encoder**: Bourns **PEC11R-4015F-S0024** — switched, since the push
  reaches the MIDI channel, which the knob otherwise does not touch. THT,
  detentless, 15mm shaft, off-board on flying leads per above. See the
  range doc for the quadrature-counting rule. Both of MC-1's encoder
  parameters are discrete lists, so with no detents the CH and DIV
  displays carry all the step feedback — a reason to keep them bright
  enough to read at a glance while turning, and DIV especially, since
  that is the one being turned.
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
- **Marking the MIDI jacks** — with icons dropped range-wide, IN/THRU
  still need distinguishing from the four CV jacks by some plain-word
  means. See the section above; it costs vertical space MC-1 does not
  obviously have.
- **Bus master firmware** — parameter addressing/encoding over I2C is
  unspecified and is now load-bearing in phase 1.
- Exact USB-C connector part number for the daughterboard.
- **AS1115 segment/digit driver dropout at 5V** against the GS2022CB-B's
  3.80V worst-case Vf — the tightest electrical margin on the module.
- **I2C level shifting** between the 3.3V MCU and the 5V AS1115.
- **A local 5V rail for the AS1115** is now confirmed necessary, not just
  likely — sizing and thermals per the note above.
- **Free-running internal clock** (MC-1 as master when no MIDI clock is
  present) — genuinely useful, genuinely separate: it needs a tempo
  control, a start/stop affordance and probably tap, none of which fit the
  current vertical budget. Deliberately out of scope, not forgotten.
- Exact mounting-hole and connector coordinates (current panel mockups are
  illustrative, not yet tied to real PCB footprints).
