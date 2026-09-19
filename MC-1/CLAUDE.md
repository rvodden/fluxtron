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
   push to reach channel — see below), with the **CLK indicator LED** to
   its lower right
4. One 2-digit 7-segment display (division at rest, MIDI channel while the
   push-to-focus mode is active — see "Single display" below)
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

#### The budget, measured off the panel mockup

The mockup is drawn to scale (0.1337mm/px horizontal against 0.1327 vertical
— isotropic to 0.4%), so it can be measured rather than estimated. Scaled to
a 40.64 × 128.5mm panel, distances from the panel's top edge:

| Feature | Measured |
|---|---|
| Jack row centres | **83.2 / 96.3 / 109.1mm** |
| **Jack row pitch** | **12.95mm** (13.1 then 12.8) |
| Display, lit segments | 17.0 × 14.3mm, centred y ≈ 54 |
| USB-C cutout as drawn | 12.4 × 4.3mm, centred on the panel |
| CLK LED | ~3.2mm dot at (32.5, 40.6) |

**⚠️ It closes on the jack pitch, not on anything else.** Three rows at
12.95mm is a **~34.7mm block against the ~54mm** the paragraph above budgets
at 18mm pitch — about 19mm recovered, which is what pays for a display twice
the height of the old pair. Totalling the rest with this file's own
allowances (11mm wordmark block, 25mm encoder finger clearance, 19mm display
*body*, ~10mm USB row, three dividers) gives **~107mm against ~110mm
usable**: the single-digit slack, but now resting on one number.

**⚠️ That number is below anything the range has validated.** `../CLAUDE.md`
confirms 13.5mm centres work and that 10.16mm fails on nut-driver access;
12.95mm sits in the untested gap between them. **Prove it with real
Thonkiconns and a nut driver on a test panel before the board is laid out** —
an FR4 blank from a cheap fab with the real footprints is the quick way. If
12.95mm does not hold, the display is what gives, not the jacks.

#### ⚠️ Two things the mockup under-draws

Both make the panel look emptier than the real parts will:

- **The display is drawn as bare segments, not as a package.** 17.0 × 14.3mm
  of lit area against a real 0.56" two-digit body of **25 × 19mm** — 8mm
  wider and 4.6mm taller than what is on the drawing. The height fits; the
  **width** is the one to check, since 25mm in a 40.64mm panel leaves ~3.9mm
  of aluminium each side, immediately below the encoder bushing hole.
- **The USB-C cutout is drawn 12.4 × 4.3mm**; the derived cutout is
  **9.3 × 3.5mm** (see the connector entry). Not wrong if it is deliberately
  drawing overmould clearance, but the slot itself should be the smaller
  figure. *(An earlier mockup had it at 26.2mm wide, nearly 3× — corrected.)*

### CLK indicator LED

A blue THT LED in the control zone, flashing on each emitted clock pulse.
It answers two questions the panel otherwise cannot: whether MIDI clock is
arriving at all, and what rate is actually coming out — the latter being a
check on the DIV setting that does not require reading the display.

**⚠️ Placement supersedes the earlier "centre gap on the VEL/CLK row".**
That position put the LED between the two jack columns at jack height,
which is precisely where patch cables sit: a plug body stands ~10mm proud
of the panel, so the indicator was obscured from any off-axis viewing angle
exactly when the clock mattered. It has moved to **the lower right of the
encoder, above the display** — measured from the panel mockup, a ~3.2mm dot
centred near (32.5, 40.6)mm from the panel's top-left.

Two reasons, and the second is a board decision as much as a panel one:

- **Cables no longer cover it.** The new position is in the control zone,
  which never has anything plugged into it.
- **It lands on the UI daughterboard** with the encoder, display and USB-C
  (see the construction section), so it needs no flying leads and no
  separate mounting. It is a THT LED, which the range's depth rules exempt
  from standoff constraints entirely — leads are bent to reach the panel —
  so it could have gone on any board; being on this one is simply tidier.

**It carries no label, deliberately.** The old justification — that it sat
beside a jack already labelled CLK, so proximity said the rest — does not
survive the move: it is now ~55mm from the CLK jack. The decision stands on
the range-wide rule instead (see `../CLAUDE.md`): **a module's only
indicator needs no label**, because nothing competes with it, its behaviour
identifies it the moment a clock is running, and the manual documents it.
**If MC-1 ever gains a second LED, both get labelled** — at that point
neither is self-identifying.

**⚠️ One risk the position carries, for whoever lays out the panel art.**
Sitting beside the encoder and 5mm above the display, an unlabelled blue
dot invites reading as an *encoder* indicator — and MC-1 does have a hidden
encoder state (division vs. channel focus) which is signalled by the
display's decimal point, also blue. Two blue indicators 5mm apart meaning
different things. Judged acceptable because the LED's behaviour is
unmistakable once a clock runs, but if the panel ever feels ambiguous in
the hand, this is why, and moving the LED further from the encoder is the
fix rather than labelling it.

- **Driven from its own MCU GPIO, not the AS1115.** MC-1 has an AS1115 for
  the displays and it has spare capacity, but hanging the clock LED off it
  would multiplex the indicator at the display refresh rate and bind its
  brightness to the displays' global intensity — which is already being
  set as a compromise between readability and LDO dissipation. A GPIO
  keeps clock timing independent of display refresh and lets the LED be
  dimmed on its own terms. +12V through a series resistor and a small
  NPN/MOSFET, per the range-wide blue-LED rule.
- **⚠️ It therefore puts +12V on the UI daughterboard's ribbon.** That rail
  was not in the ribbon's original budget (VBUS/GND/D+/D−, encoder
  A/B/common/switch ×2, SDA/SCL/+5V, 3.3V). One more conductor, either
  feeding a transistor on the UI board or carrying the LED's switched leg
  from a transistor on the main board.
  - **⚠️ Do not "simplify" this by driving the LED from the +5V already on
    that board.** It works electrically, and it is wrong: at 4.68V at the
    pin, a blue LED's 3.0V typical / 3.8V maximum Vf leaves only 0.88–1.68V
    across the series resistor, so brightness varies about **2:1 part to
    part** — the same defect the range doc rejects 3.3V GPIO drive for. From
    +12V the Vf spread is swamped. Spend the conductor.
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

**Still unaddressed as of the latest mockup**, where IN and THRU are drawn
as plain Thonkiconns indistinguishable from VEL, CLK, V/OCT and GATE. It is
the oldest open item on this module and the cheapest to get wrong in use.

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
pair (or by its decimal point, which the GS2022CB-B does have). **⚠️ That
property is under review** — see "Single display" below, which trades it for
larger digits.

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

### Single display: proposed, not settled

**Proposal: drop to one two-digit display**, showing division at rest and the
MIDI channel only while the push-to-focus mode is active. The argument is
that channel is set once when the rack is patched and then left alone, so it
does not earn permanent panel area.

**The reasoning is sound and the part count falls** — one display instead of
two, two digit-select lines instead of four, spare AS1115 capacity. Keep it
on those grounds.

**⚠️ But it is not a source of vertical space, which is what it was reached
for.** Two GS2022CB-Bs sit *side by side in one row*; one larger display is
still one row. The saving is horizontal, and 24mm of a 40.64mm panel was
never the problem. A 0.56" two-digit part is 25 × 19mm against the pair's
24 × 10mm — the same width and **9mm more height**, spent in the one axis
this module has "only single-digit millimetres of slack" in. Whether 8HP can
carry that is the vertical-budget open item; this proposal does not resolve
it.

**⚠️ It creates a genuine hidden mode, and the codes collide.** Divisions
include `16`, `08`, `04` and `02`; channels run `01`–`16`. On one display
`04` is either a quarter-note division or MIDI channel 4, and position no
longer tells them apart. VO-1's rule — *momentary, or audible, or indicated,
never a silent latch* — is still satisfied, but only because the indication
now carries real weight rather than being garnish. **Use both signals: light
the decimal point in channel mode, and blink.** The division encoding was
deliberately designed not to need decimal points, which is exactly what
leaves them free for this.

#### ⚠️ `SLR0562DBA3BD` is rejected — common anode

Evaluated as the 0.56" candidate and it cannot be used.
`../datasheets/C225942.pdf` states 共阳 (common anode), and its wiring
diagram confirms it: pins 8 and 7 are the DIG.1/DIG.2 commons feeding the
anodes, with segments returning on 10, 9, 1, 4, 3, 6, 5, 2. **The AS1115
sources segment current and sinks digit current, so it requires common
cathode** — which is why the GS2022**C**B-B was picked over the GS2022A.
Not fixable in firmware. Source the common-cathode equivalent of the same
family; the file is an LCSC part sheet (`C225942`), so the search starts
there.

Everything else about the part was good, and carries over as the screening
criteria for its replacement:

| | SLR0562DBA3BD |
|---|---|
| Digits | 0.56" (14.2mm), two, 8° slant, black face |
| Body | 25 × 19 × **8.0mm**, leads 6.2 ±0.5mm, 10 pins |
| Vf | typ 3.0 / **max 3.4V — specified only at IF = 20mA** |
| Iv | 85 / 125 / 185 mcd at 20mA |
| λd | 457 / 462 / 467nm — blue |

- **⚠️ Screen any taller display on Vf at 10mA before digit height.** Many
  parts above ~0.4" put **two dice in series per segment**, which takes Vf to
  6–7V and cannot be driven by an AS1115 on a 5V rail at any current. This
  part is single-die and at 3.4V max is *better* than the GS2022's 3.80V —
  but its datasheet characterises Vf only at 20mA, so the 10mA figure is an
  extrapolation and belongs on the bench list.
- **The margin improves only if it still runs dim.** Scaling the AS1115
  driver drops as the 5V-rail section does gives ~3.61V of chain at 10mA
  (margin ~+1.07V) but ~4.21V at 20mA — no better than today. And this part
  is *less* efficient than the GS2022 (85–185 mcd at 20mA against 120–180 at
  10mA), so brightness pressure pushes toward the current that spends the
  gain. The "do not fix a dim display by raising the intensity register"
  rule holds whichever part is chosen.
- **A thick THT display helps the board stack.** At 8.0mm deep with 6.2mm
  leads it is nowhere near the 3.2mm SMD tier, and it reaches the panel from
  the 6.50mm UI board — see the construction section. **Confirm the
  replacement's body depth before that standoff is committed**, since the
  6.50mm figure depends on it.
- **A 25 × 19mm part needs a 25 × 19mm window** in a 40.64mm panel, leaving
  ~3.8mm of aluminium each side and close to the encoder bushing hole. Check
  that before cutting.

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
- **⚠️ One MC-1 per segment — a second MC-1 means a second *system*, not a
  second voice in the same case.** Two MC-1s sharing one bus board was
  considered and rejected: they would contend for the bus CV line, the bus
  Gate line and the master role, and the CV contention is silent — two drivers
  average to a plausible wrong pitch with no error raised anywhere. The
  reasoning and the alternatives weighed are in `../BUS.md` §2.8.
- **So a second MC-1 takes its own chassis**, with its own PS-1 and bus board,
  where it is that segment's master and its chassis is a chassis 0 in its own
  right. It gets a private address space, its own presets and its own Panic
  domain; the two racks share no conductor. Note the desk consequence: a DAW
  panic must go out on every channel that has an MC-1 on it, since one CC 120
  reaches one rack.
- **MIDI THRU is unaffected and stays.** The daisy chain — one USB cable into
  the first MC-1, TRS THRU onward to the next, each filtering to its own
  channel — is how two systems stay fed from one host, and it was never the
  thing in conflict. What it does not do is make them one system.
- **⚠️ A *bridged* chassis is mastered by its CX-1**, so an MC-1 there would
  put two masters on that segment too. CV and Gate reach a bridged chassis over
  the differential link on the CX cable instead (`../BUS.md` §2.7) — cheaper
  than an MC-1 and without the conflict. Bus expansion (CX-1) and voice
  expansion (another MC-1) are separate axes: a chassis is either bridged into
  an existing system or the head of a new one, never both.

## Physical construction: boards by depth tier

This module needs more than the range-wide "main board + flying leads"
default, because USB-C, the display and the encoder all want panel-to-PCB
depths that do not match the 10mm jack-driven main board.

**⚠️ Supersedes "three separate boards"** — which put the encoder on flying
leads and gave USB-C and the display a daughterboard each, on the grounds
that their depths were incompatible. With the real connector drawing in hand
(`../datasheets/2193200001_sd.pdf`) that grouping is wrong. Measured to the
panel's **rear** face, as the jacks' 10mm is, the non-jack parts fall into
**one tier at ~6.5mm**, not three separate ones. The old 8.8mm figure for
USB-C was its height above its own PCB, which includes the 2mm panel.

- **UI daughterboard, 6.50mm standoff** — carries the encoder, the display,
  the USB-C receptacle, the AS1115 and the **CLK indicator LED** (a THT part,
  which the range's depth rules exempt from the standoff entirely; its leads
  are bent to reach the panel). Taking the panel's rear face as zero and its
  front as −2.0mm:

  | Part | Geometry | At a 6.50mm standoff |
  |---|---|---|
  | Encoder | body 6.5mm behind mounting surface | **exact** |
  | USB-C | 8.80mm above PCB → face at −2.30mm | 0.30mm proud of panel front |
  | Display (THT, 8.0mm body) | face at −1.50mm | 0.5mm recessed in its window |

  The encoder is the rigid datum — bushing and nut clamp it to the panel, so
  the board comes to it. The display pokes 1.5mm into the 2mm panel window
  and stops half a millimetre shy of the front, which reads as intentional
  rather than as a part behind a hole.

  **⚠️ The display row of that table is provisional.** The 8.0mm body depth
  comes from `SLR0562DBA3BD`, which is rejected (common anode — see the
  display section). The 6.50mm standoff holds for the encoder and the USB-C
  regardless, but re-check it against the replacement part's body depth
  before committing the stack.

  **1.6mm board.** It carries a large THT display, a THT encoder and a USB-C
  receptacle, which out-votes Molex's 1.0mm footprint recommendation — see
  the connector entry for what that costs.

  **The AS1115 belongs here, not on the main board**: it keeps the segment
  traces short, and the ribbon back to the main PCB then carries only
  VBUS/GND/D+/D− (4), encoder A/B/common/switch ×2 (5), SDA/SCL/+5V (3),
  3.3V and **+12V for the CLK LED** — roughly a 15-way, replacing two
  daughterboard connections and a five-wire flying-lead bundle. See the LED
  section for why that +12V conductor is not optional.

- **Main PCB, 10mm standoff** (set by the six Thonkiconn jacks): jacks,
  power header, bus connector, MCU, both DACs, protection.

  **Only its *front* face is constrained** by the UI board sitting 3.5mm in
  front of it. Its rear face is unobstructed, with ~58mm of case depth
  behind it, so the dense work goes there — and the power and bus headers
  want to face rearward to meet their ribbons anyway. That is what lets MC-1
  stay at two boards despite the upper section growing.

### ⚠️ Board count is deferred to schematic capture

Splitting the jacks onto a third board, leaving a full-size "guts" board
behind everything, was considered and is **not decided either way**. It
needs a real circuit to lay out before the area question is answerable.

Depth is not the obstacle — the stack comes to roughly 27mm of the KOMA's
70mm. Two other things decide it, and both should be settled alongside the
schematic:

- **⚠️ Ground is the reason to be cautious, and it lands on MC-1 hardest.**
  One cent is 833µV. With jacks on their own board every sleeve return
  reaches the output stages' ground reference through a ribbon, and ~10mA of
  gate current through ~50mΩ of ribbon ground is 0.5mV — about **0.6 cents**,
  moving whenever the gate does. That is larger than every static term in the
  range's pitch error budget except the discrete-resistor row, and MC-1 is
  the module that cannot absorb it, having no auto-tune to hide behind. If
  the split happens: alternate signal and ground through the ribbon rather
  than running one ground wire, and give V/OCT its own return to the output
  stage's ground reference rather than sharing with GATE.
- **Mechanical support.** Every module in the range is held by its jack nuts.
  Move the jacks to a daughterboard and the *jack* board is held while the
  guts board has nothing holding it — the same problem `../PS-1/CLAUDE.md`
  records as needing a panel bracket and a rear standoff, now invented for a
  second module.

## Confirmed parts

- **Jacks**: **four** Thonkiconn PJ301M-12 (V/OCT, GATE, VEL, CLK) plus
  **two stereo jacks** for the TRS MIDI IN/THRU pair. Two across, three rows.
  - **⚠️ Corrects "PJ301M-12 ×6".** The PJ301M-12 is a *mono* jack with no
    ring contact and cannot carry 3.5mm TRS Type A, so the earlier list
    specified a part that physically cannot do the job on two of its six
    positions. A PJ320-class stereo jack is needed there: a second BOM line,
    a different footprint, and an extra conductor per jack on whatever
    ribbon serves that row. Exact part at schematic capture. See
    `../CLAUDE.md`.
- **Display**: **⚠️ the quantity and the part are both open** — the panel
  now carries **one** display, not two (see "Single display" above), and the
  0.56" candidate evaluated for it was rejected as common anode. What follows
  describes the two-display arrangement it replaces; it stands only as the
  fallback if the single-display scheme is abandoned, and as the reference
  for what a replacement part has to match electrically.

  **Guangcai GS2022CB-B ×2** (CH and DIV) — 0.2" (5.08mm)
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

### The 5V rail closes — and it closes because the display runs dim

**Resolved against DS000206** (`../datasheets/`), which was the tightest
open electrical question on this module. The AS1115 sources segment current
from its own V+, so V+ must clear the segment Vf plus both driver drops in
series. The datasheet gives the drops as test conditions rather than as
dropout figures:

| Driver | Datasheet condition | MC-1 runs at |
|---|---|---|
| Segment source | 1.00V drop at 41mA (`VDD=5.0V, VOUT=VDD−1V`) | **10mA** |
| Digit sink | 0.65V drop at 320mA | **80mA** (8 segments) |

MC-1 sits a factor of four below both, so scaled down the chain needs
**4.21V at V+** with a worst-case 3.80V segment — leaving 0.79V spare on a
clean 5.0V rail.

**⚠️ The margin exists only because the display is run dim.** At the
datasheet's own test currents the same sum is 5.45V and **would not work at
5V at all**. MC-1 already intended to run well below full intensity for
thermal reasons; that is now a *functional* requirement, not an aesthetic or
thermal preference. Do not let anyone "fix" a dim display by raising the
intensity register.

**⚠️ And it only closes on a low-drop element.** Budgeted at the pin rather
than at the bus, per the range-wide warning:

| PS-1 5.00V, less… | At the pin | Margin |
|---|---|---|
| **P-FET 0.006** + PTC 0.20 + pour 0.05 + feed 0.06 | 4.68V | **+0.48V** |
| the same at connector end-of-life (feed 0.13) | 4.62V | **+0.41V** |
| Schottky 0.30 + PTC 0.20 + pour 0.05 + feed 0.06 | 4.39V | +0.18V |
| Silicon diode 0.70 + PTC 0.20 + pour 0.05 + feed 0.06 | 3.99V | **−0.22V — fails** |

The **feed** term is PS-1's Micro-Fit cable losing V+ and GND together
(`../PS-1/CLAUDE.md`); it was missing from an earlier version of this table,
which therefore read ~0.06V optimistic.

**Fit the MOSFET.** At 126mA a ~50mΩ logic-level P-channel part drops 6mV
where a Schottky drops 300mV, which more than doubles the margin for the
cost of a gate resistor. It must be a *logic-level* part — Vgs is only −5V
here. See the reverse-polarity rules in `../CLAUDE.md`, including the
orientation gotcha.

Two further levers if it is ever wanted: **raise the bus to 5.25V** (+0.49V
on top, well inside the AS1115's 2.7–5.5V operating range), or drop the
segment current below 10mA.

**Correction: the AS1115's absolute maximum is 7V, not 5.5V.** 5.5V is the
top of the *operating* range (2.7–5.5V). An earlier revision of this file
had the two confused, which made running V+ near 5.5V look like flirting
with the abs-max rating. It is not — it is an ordinary operating point and a
legitimate margin lever.

Two further consequences:

- **⚠️ I2C level shifting is mandatory — settled, not a budget item.**
  `VIH = 0.7 × VDD` for SDA and SCL (DS000206), so a 3.3V MCU cannot drive
  the AS1115 directly at any V+ this module can use:

  | V+ | VIH | 3.3V MCU |
  |---|---|---|
  | 5.50V | 3.85V | fails |
  | 5.25V | 3.67V | fails |
  | 5.00V | 3.50V | **fails** |
  | 4.50V | 3.15V | would work — but steals segment headroom |

  Dropping V+ to 4.5V is the only way to avoid the shifter, and the section
  above shows the segment chain cannot spare the 0.5V. **Fit the MOSFET
  level-shifter pair.** Pull-ups to 5V would also over-voltage the MCU pins,
  which was always the other half of the reason.
- **Where the 5V comes from — PS-1 changes this.** The rail should now be
  taken **from the bus**, not made locally. PS-1 provides a guaranteed +5V,
  and MC-1 carries a 16-pin power header like every module — that is now
  range-wide, since every module's 3.3V LDO runs from +5V, so it costs MC-1
  nothing beyond the extra segment current. **Lay out the local LDO anyway, unpopulated, with a jumper selecting
  the source** — that is what keeps MC-1 working in a case whose PSU has no
  5V rail. The dissipation figures below apply only if the LDO is populated.
- **LDO dissipation, and dimming as a thermal lever** (local-LDO path only).
  Multiplexed, one digit lit at a time, all 8 segments at 10mA is ~80mA from
  the 5V rail — 7V × 80mA ≈ **560mW** in a linear LDO from +12V. That was the
  worst thermal spot in the range, on its most crowded board, and taking the
  rail from the bus deletes it. If populated, it wants a SOT-223 or
  DPAK part and a real thermal pad, and it goes on the PTC budget. At
  120–180 mcd these are very bright and will almost certainly be run well
  below full intensity, which cuts the dissipation proportionally — so the
  AS1115's intensity setting is a power decision here, not only a visual
  one. No switching regulator: MC-1 generates precision pitch CV, so the
  range-wide linear-only rule applies to it as much as to VO-1.
  **Digit-drive polarity confirmed**: DS000206 describes the DIG0:DIG7 lines
  as sinking current from the display common cathode, and the segment lines
  as sourcing into it — which is the common-cathode GS2022C**x** already
  specced. The two match; nothing left to check here.
- **Parameter DAC**: **MCP4728** (12-bit, 4-channel, I2C) — velocity CV
  on one channel, three spare. On the local I2C port alongside the AS1115.
  **Sit it on the 3.3V side of the AS1115 level shifter**, not the 5V
  side: the AS1115 needs 5V only because blue segments demand it, and
  there is no reason to drag the DAC up with it.
- **Pitch DAC**: **AD5693R — specifically `AD5693RBRMZ`, the B grade**
  (16-bit, I2C, 2.5V on-chip reference at 2ppm/°C typ) for the V/OCT output.
  **⚠️ `AD5693RARMZ` is the A grade** — 20ppm/°C max, 4.82 cents, and one
  letter away. MC-1 has no auto-tune to hide it behind; see
  `../CLAUDE.md`. On the local I2C bus alongside the
  MCP4728 and the AS1115 — and, like the MCP4728, on the **3.3V side of
  the level shifter**. **Firmware must write the pitch DAC before raising
  gate**, so a note-on cannot skew against a parameter update sharing the
  bus; see `../CLAUDE.md`.
  **MC-1 is the module where pitch accuracy actually matters** — it has no
  feedback loop, unlike VO-1's auto-tune. It needs a **user calibration
  routine** storing gain and offset in the reserved settings flash page.
  - **⚠️ The routine must require a warm module, not merely recommend one.**
    Calibration fixes gain and offset *at the temperature it runs at*, so the
    excursion the pitch error budget is spent against is measured from that
    temperature — not from 25°C nominal. Calibrating cold at switch-on makes
    the module eat the entire warm-up; calibrating after ~30 minutes of
    running cuts the real excursion to nearer ±5°C, which **scales every
    drift term in `../CLAUDE.md`'s budget by about 0.25**. That is worth
    roughly a cent — more than the DAC grade and the linear-vs-RSS question
    combined — for the cost of a timer.
  - **So the firmware gates it on uptime**: refuse to enter calibration until
    the module has been powered for a set warm-up period, and say why on the
    display rather than failing silently. The exact period is a bench
    measurement (watch the V/OCT output settle from cold), not a guess; 30
    minutes is the working assumption until someone measures it.
  - Calibration removes *absolute* error, not drift, which is the reason the
    BOM spends on tempco rather than initial accuracy. It also cannot
    separate the reference tempco from the DAC's gain tempco — they are one
    die at one temperature — so both land in the residual together.
- **Encoder**: Bourns **PEC11R-4015F-S0024** — switched, since the push
  reaches the MIDI channel, which the knob otherwise does not touch. THT,
  detentless, 15mm shaft, **on the UI daughterboard** per above —
  superseding the flying leads, since a board at its tier now exists. See the
  range doc for the quadrature-counting rule. Both of MC-1's encoder
  parameters are discrete lists, so with no detents **the display carries
  all the step feedback** — a reason to keep it bright enough to read at a
  glance while turning. That argument gets stronger, not weaker, with a
  single display: division and channel now share one readout, so it is the
  only feedback either parameter has.
- **USB-C connector**: **Molex `2193200001`** (219320-0001) — 16-pin USB 2.0
  Type-C receptacle, vertical/top-mount, 8.80mm above PCB. **Settled**; it
  was previously carried only as a reference depth point. Drawing:
  `../datasheets/2193200001_sd.pdf`.

  | | |
  |---|---|
  | Mating interface | 8.34 +0.06/−0.02 × 2.56 ±0.04 (USB-IF; both flagged FC) |
  | **Body envelope** | **8.94 (W) × 3.16 (D)**, constant over the full height |
  | Height above PCB | 8.80 |
  | Contact coplanarity | 0.10 |
  | Molex rec. PCB thickness | 1.0 ±0.10mm |

  - **Panel cutout: a 9.3 × 3.5mm full-radius slot (R1.75).** Cut to clear
    the **body**, not the 8.34mm mating oval — the shell is full-width right
    down to the PCB, so the mating dimension is irrelevant once it passes
    through the panel. That leaves ~0.18mm per side on width and ~0.17mm on
    height, and a full-radius slot suits both a 3.5mm end mill and the
    connector's oval-ended body. Use 9.6 × 3.8 (R1.9) if more assembly slack
    is wanted on a hand-built panel. **Molex publish no panel cutout for
    this part** — it is a board-mount connector with a land pattern, not a
    panel-mount one with a bezel — so this figure is derived here, not
    quoted from the drawing.
  - **Mount it protruding, not recessed.** Flush with the panel *front*,
    which puts the board 6.80mm behind the panel's rear face on its own, or
    6.50mm on the shared UI board with the connector 0.30mm proud. Sitting
    the connector wholly behind a 2mm panel would make the plug reach into a
    recess, and a typical overmould fouls the panel before it seats.
  - **⚠️ D+/D− are not internally bridged.** Both pairs come out separately
    (Dp1/Dn1 on A6/A7, Dp2/Dn2 on B6/B7), so **tie A6↔B6 and A7↔B7 on the
    board** — that is what makes the cable work either way up. On a 16-pin
    part this is the designer's job; some 6-pin parts do it internally.
  - **SBU1/SBU2 (A8/B8) go nowhere** for a MIDI device. Leave unconnected.
  - CC1 (A5) and CC2 (B5) each take their own 5.1kΩ pull-down, placed on
    this board near the connector.
  - **Blind shell pegs are accepted, and should be confirmed at bring-up.**
    The 1.00/1.10mm pegs do not break through the 1.6mm board the UI
    daughterboard needs, so they get no back-side solder. Judged acceptable
    because the close-fitting panel slot takes the cable insertion and
    withdrawal load, which is the failure mode the pegs exist to prevent —
    but that is reasoning, not a tested claim. The conservative alternative
    is a 1.0mm board plus a mid-board support standoff. 1.2mm is not a
    compromise; it is 0.1mm of peg and no retention.
  - **⚠️ `2171780001` is rejected — do not re-propose it on depth grounds.**
    It is a 6-pin part at 6.50mm, and its 2.3mm depth saving is real. It is
    also **power-only**: its six contacts are CC1, CC2, VBUS ×2 and GND ×2,
    with **no D+/D− at all** (`../datasheets/2171780001_sd.pdf`), so it
    cannot carry USB MIDI. Distributor listings describe it only as "6
    circuits", which reads as though it were VBUS/GND/D+/D−/CC1/CC2 — the
    drawing is the thing that settles it, and this is the second USB-C part
    number to fail on a property no listing stated.

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

## Safe state on Panic

MC-1 must define what it does on a bus Panic (`../BUS.md` §6.7), and it is the
module where the answer matters most, since it drives pitch and gate directly.

**Gate low is the unambiguous part** — that is what stops sound. The rest is
not yet decided: whether V/OCT holds its last value or goes to 0V, and whether
velocity CV drops to zero. Holding pitch avoids a click into whatever the VCO
feeds; zeroing it is more predictable. **Decide before firmware, not at
bring-up.**

MC-1 also **maps CC 120 (All Sound Off) to a bus Panic**, since that is what a
DAW's panic button sends. **CC 123 (All Notes Off) stays local** — it is about
notes, so MC-1 drops its gate and leaves the bus alone.

## Open items

- **Vertical panel budget** — the row-of-4 problem is resolved and MC-1
  closes at 8HP, but height is now the tight axis with only a few
  millimetres of slack, on estimated rather than measured footprints.
  **⚠️ Now measured rather than estimated, and it closes at ~107mm of ~110mm
  — on a 12.95mm jack pitch the range has never validated.** See "The budget,
  measured off the panel mockup" above. 0.56" digits are reachable *if* that
  pitch is buildable, which is a bench test with real Thonkiconns and a nut
  driver, not a calculation. That test is now the gate on the display part.
- **Marking the MIDI jacks** — with icons dropped range-wide, IN/THRU
  still need distinguishing from the four CV jacks by some plain-word
  means. See the section above; it costs vertical space MC-1 does not
  obviously have.
- **Bus master firmware** — the protocol itself is specified in `../BUS.md`;
  what remains is MC-1's own side of it: the NRPN state machine, the parameter
  shadow and downstream coalescing, discovery and rediscovery, preset
  stage/commit sequencing, and driving firmware updates.
- ~~Exact USB-C connector part number for the daughterboard~~ — **settled:
  Molex `2193200001`.** See the connector entry for the derived cutout, the
  flush-mount depth and the D+/D− bridging.
- **The display part is open again.** The single-display proposal stands, but
  `SLR0562DBA3BD` is common anode and unusable; a common-cathode two-digit
  blue equivalent is needed. Its body depth sets the UI board's 6.50mm
  standoff, and its digit height waits on the vertical budget above.
- **PCB count** — two boards (UI at 6.50mm, main at 10mm) or three (jacks
  split out, full-size guts board behind). **Deliberately deferred to
  schematic capture**, when the area question becomes answerable. See the
  construction section for the ground and mounting consequences of the
  three-board option.
- **Stereo jack part** for the TRS MIDI pair, per the correction in the parts
  list.
- **The warm-up period the calibration routine gates on.** The requirement is
  settled — calibration must refuse to run on a cold module, per the pitch
  DAC entry — but 30 minutes is a working assumption. Owed a bench
  measurement: log the V/OCT output from switch-on and see when it settles.
- ~~AS1115 segment/digit driver dropout at 5V~~ — **settled** against
  DS000206: it closes with +0.48V at the pin behind a MOSFET (+0.41V at
  connector end-of-life), and only at 10mA/segment. A silicon diode fails it
  outright. See the 5V rail section.
- ~~I2C level shifting~~ — **settled: mandatory.** `VIH = 0.7 × VDD` puts
  the threshold at 3.50V with V+ at 5V, so 3.3V direct drive is out.
- **The 5V rail for the AS1115** is confirmed necessary. Source is settled
  in principle — from PS-1 over a 16-pin header, with an unpopulated local
  LDO and a jumper as the portability fallback — but the connector change and
  the jumper arrangement are not yet drawn.
- **Free-running internal clock** (MC-1 as master when no MIDI clock is
  present) — genuinely useful, genuinely separate: it needs a tempo
  control, a start/stop affordance and probably tap, none of which fit the
  current vertical budget. Deliberately out of scope, not forgotten.
- Exact mounting-hole and connector coordinates (current panel mockups are
  illustrative, not yet tied to real PCB footprints).
