# FluxTron MC-1 — MIDI-to-CV/Gate Interface

Read `../CLAUDE.md` first for range-wide decisions (panel aesthetic, depth
rules, common components, bus architecture). This file only covers
decisions specific to MC-1.

## Role

The system's "front door." Converts MIDI (from USB-C or TRS MIDI IN) into
pitch CV, gate, velocity CV, and a clock/sync pulse for the rest of the
phase-1 analogue system. Also the module carrying the MIDI channel-select
control, since each MC-1 unit filters to one configured channel.

## Panel: 8HP

Layout, top to bottom:

1. FLUXTRON wordmark + "MC–1"
2. USB-C bulkhead, MIDI IN, MIDI DIN-icon, MIDI THRU (one row)
3. PITCH, GATE, VEL, CLK jacks (one row of 4, pulse icon centered between
   the middle two)
4. Two-digit 7-segment channel display (range 1–16)
5. Channel-select encoder (no LED ring — just the knob + "CHANNEL" label)

4 mounting holes, symmetric, standard oval slots.

## Outputs: CLK and velocity are both kept

Both were queried against cheaper minimal MIDI-CV interfaces (e.g. Behringer
CM1A) that omit them — decision was to **keep both**:

- **CLK**: MIDI-clock-derived sync pulse, feeds LF-1's sync input (and any
  future clocked module). Not generic — tied directly to LF-1's existing
  sync-in jack.
- **Velocity**: kept for future use (e.g. EG-1 scaling amplitude/envelope
  amount from velocity), even though nothing in phase 1 reads it yet.

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
- **Channel encoder**: off-board entirely, on flying leads (A, B, common,
  switch ×2 — 5 wires) back to the main PCB. Its own panel nut provides all
  the mechanical support; main-PCB keep-out zone behind it (16.1mm deep, per
  Bourns PEC16 body length).
- **USB-C daughterboard**: separate small board, its own shallow standoff
  set by whichever connector is chosen (checked against 219320-0001 as a
  reference point — 8.8mm, i.e. deeper than the display, hence the separate
  board). Needs the CC1/CC2 pull-downs placed on this board near the
  connector.
- **Display daughterboard**: separate small board, very shallow standoff —
  see part choice below (3mm thick). Confirmed too different in depth from
  the USB-C connector to share one carrier, even though both are "small
  digital front-panel components."

Each daughterboard connects to the main PCB via a short header/jumper
(I2C for the display's driver, USB D+/D-/power for the USB board).

## Confirmed parts

- **Jacks**: Thonkiconn PJ301M-12 ×6 (IN handled via TRS above; PITCH,
  GATE, VEL, CLK, plus the TRS IN/THRU pair).
- **Display**: **Opto Plus OPS-D2010 series** — 0.2" (5.08mm) dual-digit
  SMD 7-segment, Diamond seg., **3mm thick**, 10mm × 14.4mm footprint.
  Common anode. Specific colour not yet chosen (8 available — amber/orange
  suggested to suit the panel's warm-glow aesthetic over stock red).
  Example part numbers: OPS-D2010LR (red), OPS-D2010SA (amber).
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

- Exact USB-C connector part number for the daughterboard.
- Display colour choice.
- AS1115 polarity check against OPS-D2010 (common anode).
- Exact mounting-hole and connector coordinates (current panel mockups are
  illustrative, not yet tied to real PCB footprints).
