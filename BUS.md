# FluxTron Inter-Module Bus Protocol

The digital backplane that carries parameter values between modules. Referenced
from `CLAUDE.md`; this file is the normative description.

MC-1 is the bus master. Every other module is a slave. Only *parameter values*
travel here — see the invariant below.

---

## 1. The invariant that everything else depends on

> **The bus may carry anything where tens of milliseconds of worst-case
> delivery latency is merely undesirable. It may never carry anything where
> that would be musically wrong.**

Pitch, gate, velocity, clock and audio stay on patch cables. That is what makes
a non-deterministic, clock-stretchable I2C link acceptable as the parameter
transport. Two reasons, and the second is the one people miss:

- **Latency is unbounded in the tail.** A normal transaction is ~400µs at
  100kHz, and worst case behind a burst is ~2ms. But an STM32 slave stretches
  SCL whenever it cannot service the peripheral, and a module writing its
  settings page stalls the CPU for **tens of milliseconds** during flash erase,
  because it is executing from the flash it is erasing. One module saving its
  MIDI channel would halt the entire bus.
- **A wire holds state; a bus carries events.** A gate on a patch cable cannot
  be "lost" — the voltage *is* the state. A note-on/note-off pair is two
  events, and a dropped note-off is a note that sustains forever. Recovering
  that needs retries, acknowledgements and an all-notes-off watchdog, all to
  recreate a property copper has for free.

**Note *number* as information is fine** — telling a module the current note so
it can display it, key-track slowly, or report to a host. It is the
*triggering* role that is forbidden, not the data.

**If a cable-free gate is ever wanted**, the standard Eurorack 16-pin bus
already carries CV and Gate lines, which PS-1's bus board routes. That is
analogue and deterministic. It is a single global pair, so mono only.

---

## 2. Electrical layer

| Property | Value |
|---|---|
| Transport | I2C, multi-drop, MC-1 as sole master per segment |
| DVCC (logic rail) | **5V**, sourced by PS-1's bus board |
| Speed | **100kHz** |
| Pull-ups | One ~2.2kΩ pair, **on the bus board only** |
| Connector | **2×6 (12-pin)** IDC, shrouded and keyed |

### Pinout

| Pin | Signal | Notes |
|---|---|---|
| 1 | DVCC (5V) | Pull-up reference, **not** a module supply |
| 2 | GND | |
| 3 | SDA | |
| 4 | SCL | |
| 5 | nRESET | Master-driven, open-drain. See §8 |
| 6 | ATTN | Slave-driven, open-drain, wired-OR. See §6 |
| 7–10 | A0–A3 | Slot address, driven by the backplane. See §3 |
| 11–12 | *spare* | Reserved |

### ⚠️ Do not use a 10-pin connector

Ten signals tempts a 2×5 header — the same family as the Eurorack power
header, same ribbon, same crimp tooling. **A 10-pin bus header beside a 10-pin
power header is an invitation to plug ±12V into the I2C bus**, which destroys
every MCU on the backplane. 2×6 physically cannot accept the power ribbon, and
the two spare pins cost nothing now and are impossible to add later.

### Pull-ups are singular, and live on the bus board

One 4.7kΩ pair per module across eight modules is ~590Ω in parallel, below the
~1kΩ floor the 3mA sink specification sets — nothing would pull the bus low.
Putting them on the bus board makes the count fixed regardless of how many
modules are installed, and leaves MC-1 electrically just another module.

### Open: 3.3V vs 5V DVCC

**Confirmed: every I2C pin on the G0B1 is marked FT in the datasheet**, so 5V
tolerance holds in normal operation with VDD present. That settles the routine
case.

**The residual concern is a different row.** Absolute-max VIN on an FT pin is
`VDD + 4.0V`, not a flat 5.5V, and ST lists positive injection on FT pins as
**0mA** — it is not a characterised condition. The FT marking and the
absolute-maximum ratings answer different questions.

Our topology puts a module's VDD at 0 while the bus is live, because every
module's 3.3V rail derives *from* the 5V rail:

| Condition | VDD | Status |
|---|---|---|
| Normal operation | 3.3V | In spec — `VDD+4` = 7.3V |
| Power-up window | rising | Out of abs-max only while VDD < 1.0V — tens of µs |
| **PTC trip / LDO failure** | **0V** | **Continuously out of abs-max** |

**Current is not the problem.** The only path from DVCC to SDA/SCL is through
the bus board's 2.2kΩ pull-up — every other device is open-drain and can only
pull *low*. So injection into an unpowered pin is capped at
`5V / (2200 + 220) ≈ 2.1mA`, inside a typical ±5mA per-pin limit. *(An earlier
revision of this file claimed ~19.5mA and concluded 5V was unviable. That
treated the 5V as a stiff source at the pin, which it is not.)*

It is the **voltage** that is out of specification, and only in the fault case.

**Recommendation: 3.3V DVCC**, from a small LDO on PS-1's bus board feeding
only the pull-ups (a few mA). At 3.3V an unpowered module's pin sees 3.3V
against a 4.0V absolute max **even at VDD = 0** — the problem does not need
mitigating or accepting, it stops existing. 400kHz also comes back (below).

5V remains workable if the out-of-spec fault condition at ~2mA is acceptable.
The case for it was ribbon noise margin alone.

**Not yet decided — do not lay out a bus connector until it is.**

### ⚠️ 5V DVCC costs 400kHz

At 5V the 3mA sink spec (VOL 0.4V) puts a **floor** of 1.53kΩ on the pull-up,
while rise time puts a **ceiling** of `300ns / (0.8473 × Cb)` at 400kHz. Those
cross at about **230pF**, above which no valid passive value exists. A
realistic 84HP segment is 150–250pF. 3.3V DVCC would have a 0.97kΩ floor and
keep 400kHz out to ~370pF.

**It does not bite, because of ATTN.** With slaves signalling, there is no
round-robin polling load to spend bandwidth on, and 2.2kΩ at 100kHz is valid
across the whole capacitance range.

### Module-side requirements

- **Series resistors (~220Ω) on SDA and SCL** at each module's connector.
  Every module's 3.3V rail is derived *from* the 5V rail, so on **every power
  cycle** there is a window where the bus sits at 5V with pull-ups live while
  the MCU's VDD is still climbing from zero. The resistors limit injection into
  the pin ESD structures, and damp ribbon ringing besides.
- **⚠️ The 5V DVCC choice is under review — see the box below.** It may not
  survive the tolerance check.
- **ESD protection on SDA/SCL** per the range-wide protection standard.
- The inter-module bus and any local I2C peripheral must sit on **separate MCU
  I2C ports** (range-wide rule; the G0B1 has three).

---

## 3. Addressing

Two levels: **chassis** and **slot**.

### Slot comes from the backplane, not from the module

**Geographic addressing.** Each connector position on the bus board ties a
different pattern of `A0–A3` to ground; the module reads them as GPIOs at boot.
**Supersedes the DIP/jumper scheme** previously recorded in `CLAUDE.md`.

Better than DIP switches:

- **Duplicates become physically impossible.** Two modules cannot occupy one
  connector. A mis-set DIP gives an address clash that presents as a silently
  half-working bus.
- **Slot number *is* physical position.** MC-1 says "slot 3", you count three
  positions from the left.
- **Moving a module re-addresses it automatically**, including between chassis.

Better than the daisy-chained enable line the range doc previously flagged:
**there is no enumeration protocol at all.** Four GPIO reads at reset and the
module has a unique address. No unassigned-address state, no token passing, no
race if a module resets mid-enumeration, no recovery path to design.

Board area is roughly a wash — a 4-way SMD DIP is ~66mm², the extra connector
pins over a 2×2 are ~63mm² — so this deletes a user-configurable failure mode
for free.

**Implementation notes:**
- Module uses **internal pull-ups**; the backplane pulls down selectively.
- **Detect backplane presence via DVCC**, not via the address pins. A module on
  the bench with nothing plugged in reads all-ones, which is otherwise
  indistinguishable from a real position.

### Chassis costs a module nothing

A module never learns which chassis it is in. MC-1 addresses `(chassis, slot)`;
the bridge for chassis N strips the chassis field and forwards the bare slot
onto its local bus. Modules are genuinely chassis-agnostic.

**Only the CX-1 bridge is configured** — one per chassis, set once, where a
jumper is entirely reasonable.

### I2C addresses

`I2C address = 0x20 + slot`, so **0x20–0x2F**. Clear of the reserved ranges
(0x00–0x07, 0x78–0x7F). Because each chassis has a private address space, only
16 addresses are ever consumed, leaving most of the I2C space free.

### Why 16 slots is enough

Not arbitrary — **I2C's own 400pF bus capacitance limit caps a segment at
roughly 16 modules**, before the 4-bit slot field does:

| Modules on a segment | Approx. Cb | |
|---|---|---|
| 16 | 210–290pF | comfortable |
| 26 | 330–440pF | at or over the limit |
| 32 | 400–580pF | not a working bus |

(~10–15pF per module of pin, connector and stub, plus ~50pF/m of ribbon.)

Physical positions: 84HP = 10, 104HP = 13 — both fit. A 6U or 2×104HP case is
20–26 positions and wants **two segments with a bridge**, which is what
capacitance demands anyway. The two constraints agree.

Going to 5 bits would cost only one spare connector pin, but would buy address
space that cannot be physically populated.

Total system capacity is 8 chassis × 16 slots = **128**, the full NRPN MSB
space — around 80 modules across 672HP.

---

## 4. Multi-chassis: bridge, not buffer

The chassis extension module (**CX-1**) is an I2C **slave** on the upstream
chassis's bus and a **master** on its own, with a PCA9615-style differential
pair for the physical link.

**Rejected: a transparent buffer** making one logical bus across all chassis.
It leaves a single flat address space — every module in the system needing a
globally unique setting, with a ledger of which case got which range, and
re-jumpering whenever a module moves. Capacitance and fanout also keep
accumulating, and a fault anywhere takes down everything.

The bridge gives each segment private addressing, bounded capacitance, and
fault containment. **It needs no new architecture**: the range-wide rule that
the bus and local peripherals sit on separate I2C ports means the standard
STM32G0B1 is already a two-port device. CX-1 is that part with a different
firmware personality.

**Topology is a daisy chain**, and it falls out for free: the bridge is
addressed like any other module on its parent's bus. Traffic for chassis N+1
goes to the bridge's slot address on chassis N's bus, which re-emits it
downstream. Recursion, no special routing.

- **MC-1 discovers the bridge** by scanning and reading identity registers for
  module type `CX`. An earlier draft reserved slot 15 for it; that does not
  work with geographic addressing, since the bridge occupies whatever physical
  position it is plugged into.
- Each downstream chassis needs its own PS-1 — the inter-chassis link carries
  only differential I2C plus a ground reference, **never power**.
- **The bridge aggregates a summary dirty bitmap** — which slots below it have
  pending events — so MC-1 does one read per *chassis*, not per module.
  Without this, upstream cost grows with system size.
- **⚠️ The bridge needs a transparent pass-through mode** for firmware
  updates. Bootloader traffic uses a fixed address that protocol-aware
  forwarding will not recognise.

---

## 5. Register model and transactions

Standard register-pointer model, so the bus is debuggable from a Pi with
off-the-shelf tools.

| Range | Meaning |
|---|---|
| `0x00–0x7F` | Module parameters (matches the NRPN LSB range exactly) |
| `0x80–0xFF` | System registers |

- **Write:** `[reg, val_hi, val_lo]`, with `reg` auto-incrementing on longer
  payloads — one transaction per module for a preset recall.
- **Read:** write `[reg]`, repeated START, read 2 bytes (auto-incrementing).
- **Values are always uint16, full scale**, whatever the parameter means. The
  module owns the mapping into its own DAC. Booleans use 0 / 0xFFFF; enums use
  small integers.
- **Big-endian on the wire**, matching MIDI's MSB-first convention.

### System registers

| Reg | Name | Access | Notes |
|---|---|---|---|
| `0x80` | `MODULE_TYPE` | R | Two ASCII chars, e.g. `'V','O'` |
| `0x81` | `MODULE_NUMBER` | R | The `-1` in `VO-1` |
| `0x82` | `PROTOCOL_VERSION` | R | This document's revision |
| `0x83` | `FW_VERSION` | R | |
| `0x84` | `PARAM_COUNT` | R | How much of `0x00–0x7F` is real |
| `0x85` | `CAPABILITIES` | R | Bitfield |
| `0x86` | `STATUS` | R | Busy, calibrating, error |
| `0x87` | `DIRTY` | R | 32-bit bitmap, two registers |
| `0x88` | `COMMAND` | W | Identify, calibrate, clear error |
| `0x90` | `STAGE` | W | Preset staging, §7 |
| `0x9F` | `ENTER_BOOTLOADER` | W | Magic value only, §8 |

### Broadcast

I2C **general call** (address `0x00`), with our commands at second byte
`0x10`+ to stay clear of the spec's own `0x04` and `0x06`. The STM32 slave
supports this via `GCEN`. Used for preset commit and panic.

*(Alternative considered: the G0's second address register with masking, which
would let a module answer a broadcast address in hardware. General call is
simpler and purpose-built; OA2 masking stays available if needed.)*

---

## 6. Upstream events: dirty bitmap, not a FIFO

Panel encoders move parameters locally. MC-1 needs to know, so a host can
record or reconcile — MC-1's USB link is the only path a panel move has to the
outside world.

A module sets a bit in `DIRTY` per changed parameter and asserts **ATTN**.
MC-1 reads the bitmap, then reads the flagged parameters.

**A bitmap cannot overflow — it coalesces**, which is exactly right for a knob
being turned, where a FIFO would drop events or back up. ATTN removes the
round-robin poll entirely; a fallback slow poll (~1Hz) covers a missed edge.

**Two rules that prevent feedback loops:**
- MC-1's shadow updates on read, and it never re-writes a value it just read.
- **A preset commit does not raise dirty flags.** MC-1 knows what it wrote.

MC-1 should also **coalesce downstream**: keep a shadow of every parameter and
push changes on a fixed tick, so a DAW automation sweep cannot storm the bus.

---

## 7. Presets

**Presets live in MC-1, not in the modules.** A preset is a property of the
*rack*, not of a module.

Rejected: per-module preset storage with a broadcast "recall preset 5". It
needs almost no bus traffic, but a module carries its own idea of preset 5
between racks, saving means every module writes flash — the exact clock-stretch
hazard of §1 — and backing a preset up over USB means reading it all back
anyway.

**Presets are not settings.** The existing settings store (module flash, per
`CLAUDE.md`) keeps things tied to the *hardware*: VO-1's calibration constants,
MC-1's MIDI channel and clock division. Those are never part of a preset.

### Recall: stage, then commit

The whole reason this is in the protocol from the start. Writing parameters one
at a time and applying each immediately makes the rack audibly sweep through
intermediate states — a filter opening, a pitch gliding — on what should be an
instant change.

1. MC-1 writes `STAGE = 0x0001` to each populated slot. Parameter writes now
   land in a **shadow copy**; DAC outputs do not move.
2. MC-1 writes the parameters, one auto-incrementing burst per module.
3. MC-1 issues a **general-call COMMIT**. Every module applies its shadow
   simultaneously.

The broadcast is what makes this worth doing: the entire rack switches in **one
I2C transaction**, sub-millisecond skew across every module, instead of tens of
milliseconds of visible sweep.

- `STAGE = 0x0000` **aborts** and discards the shadow.
- **Staging auto-aborts after ~1 second** with no commit, so a master that dies
  mid-recall cannot leave modules staged forever.
- **MC-1 verifies every staged write ACKed before committing.** If any failed,
  abort rather than commit a partially-updated rack.
- Panel encoders keep working during staging; the commit simply wins. Staging
  windows are short enough that freezing the panel would be worse.

### ⚠️ Type-check every slot on recall — this is a safety requirement

A preset records the **module type per slot**. If a preset saved with VO-1 in
slot 2 is recalled into a rack with VF-1 there, MC-1 must **skip that slot**.
Writing VO-1's pulse-width value into whatever a filter keeps at the same
register is not a cosmetic bug; it sets an unrelated parameter to an arbitrary
value. Mismatches are reported, never guessed at.

### Save

MC-1 reads all parameters from each populated slot — one auto-incrementing
burst per module, length from `PARAM_COUNT` — and writes its own flash.

**⚠️ MC-1 must run its flash routines from SRAM**, or defer saves until no note
is sounding. A G0 page erase stalls a CPU executing from flash for tens of
milliseconds, and MC-1 is generating V/OCT and gate.

### Format and capacity

Variable length, storing only populated slots: a per-slot header of chassis,
slot, module type and parameter count, then the values. A typical 8-module rack
at ~8 parameters each is around 160 bytes.

**Program Change recalls a preset**, which is what a DAW will send anyway.
Save has no MIDI equivalent, so it is a system command (§9).

Exact flash budget, preset count and any wear-levelling are MC-1 implementation
details, not protocol. Note that flash endurance is ~10k cycles.

---

## 8. Firmware update over the bus

`CLAUDE.md` records that MC-1 can reflash any module using the STM32's I2C
system bootloader. That pulls three things into the protocol:

- **`ENTER_BOOTLOADER` takes a magic value, never a bare flag.** A module that
  jumps to the bootloader by accident goes dark until power-cycled.
- **The bootloader uses a fixed I2C address**, not the slot address, so **only
  one module can be in bootloader mode at a time**. MC-1 sequences reflashing
  strictly one-by-one.
- **Confirmed against AN2606's STM32G0B1xx/0C1x table: either I2C1 or I2C2
  may be used.** The pin set *within* each peripheral is fixed — no alternate
  AF mappings — but there is a choice of peripheral. *(An earlier revision of
  this file said the bootloader was I2C1 on PB6/PB7 only, "fixed, not
  remappable". That came from a community report about the G030 and was
  over-generalised; the G0B1 has more options.)*
- **Consequence: local peripherals go on I2C3.** The G0B1 has three I2C ports
  and the range-wide two-port rule needs the inter-module bus separate from
  the AS1115 / MCP4728 / AD5693R. I2C3 is the port that is *not*
  bootloader-capable, so spending it on locals leaves both qualifying ports
  free for the bus — the bus then takes whichever of I2C1/I2C2 routes better,
  rather than being pinned to one peripheral before layout starts.
- **Bootloader pins, from the same table:**

  | Peripheral | SCL / SDA |
  |---|---|
  | I2C1 | **PB6 / PB7** |
  | I2C2 | **PB10 / PB11** |

  Both are on port B and both are bonded out on LQFP-48, so the choice is
  genuinely free. **Pick one and use it on every module**: uniform firmware
  and layout are worth more than per-module optimisation.
- **Bootloader address: 7-bit `0x5D`** (`0b1011101x` — `0xBA` write, `0xBB`
  read). Clear of the `0x20–0x2F` slot range. **Both I2C1 and I2C2 use the
  same address**, so the one-module-at-a-time constraint holds whichever
  peripheral the bus takes. Bootloader config is target mode, 7-bit
  addressing, analog filter on, up to 1MHz — the bootloader itself is not a
  speed constraint.

### ⚠️ AN2606 bootloader limitations that shape the update flow

Four are documented against this bootloader. Three change what we do:

- **The `Go` command disables the debug access port** — it writes a wrong
  value to `FLASH_ACR`'s `DBG_SWEN` bit when jumping to the application.
  **So never use `Go`. Start the application with nRESET instead.** Cleaner
  regardless, since the module starts from a known peripheral state, and it
  sidesteps the bug entirely. This is the second independent reason nRESET
  earns its pin.
- **Multi-sector erase is broken on Bank2** — a wrong BUSY-bit check raises a
  FLITF error after the first sector. **Workaround: erase one sector at a
  time when targeting Bank2.** Conditional: at 128KB (`CB`) there is probably
  no Bank2, since dual-bank is a feature of the larger G0B1 flash variants.
  **Confirm before any module moves to a bigger part — MC-1 is the likely
  candidate, since it stores the presets.**
- **⚠️ The Empty-check flag is cleared during bootloader startup.** The
  bootloader's own deinitialization writes the default to `FLASH_ACR`, zeroing
  the Empty-check bit. So a module that booted to the bootloader *because* its
  flash was empty will, **on a subsequent reset, try to boot the empty flash
  and crash.**

  This interacts badly with a shared nRESET. A module interrupted mid-reflash
  — erased but not yet programmed — boots to the bootloader via empty check,
  which is what we want; but asserting the backplane's nRESET for any reason
  then crashes it, because nRESET resets *every* module.

  Two rules:
  - **Never assert nRESET while any module is mid-update.** The reset that
    exits the bootloader comes strictly after programming completes.
  - **Do not rely on empty check as the recovery path.** The BOOT0 jumper is
    deterministic regardless of flash state, which makes it more important,
    not less.

  The application note's own wording closes this off: *"Avoid using reset on
  this case. If the system crashes, an option byte change or POR is needed to
  reboot."* **An nRESET pulse is not a POR**, so the backplane's reset line
  cannot recover a module crashed this way — only actually removing power can.
  That is survivable in a rack you can switch off, but it is the reason the
  BOOT0 jumper exists rather than a cleverer bus-driven mechanism.
- **⚠️ Reported erratum: the I2C bootloader hangs if PA3 stays low**, needing a
  pull-up on PA3. If PA3 is used for anything that idles low, reflash-over-bus
  fails silently. Reported against bootloader v5.2 on a G030, so **check
  whether it applies to the G0B1's bootloader version** before designing the
  pull-up in — it is cheap insurance either way.
- **nRESET is what makes the feature survive a bad flash.** A software
  "enter bootloader" command can only be delivered while the application is
  still running — precisely not the case when reflashing is most needed. Without
  a hardware path, a bricked module is unreachable and you are back to pulling
  modules and hunting for SWD pads, which is the thing the feature exists to
  avoid.
- **BOOT0 is a per-module jumper, not a bus line.** A shared BOOT0 would put
  every module into the bootloader at once, all answering on the same fixed
  address. Since only one module can be in bootloader mode anyway, a jumper
  fits the constraint rather than fighting it: set it on the board being
  recovered, assert the shared nRESET, and that module alone comes up in the
  bootloader while the rest boot normally. nRESET is also the way *out* of the
  bootloader in the normal software-jump flow.

**⚠️ Layout constraint** (also in `CLAUDE.md`): the inter-module bus must land
on a **bootloader-capable I2C peripheral and pin set**, per AN2606, or the
whole feature is lost. Free if designed in, impossible to retrofit.

---

## 9. MIDI mapping

NRPN's structure maps onto the two-level bus address with **zero translation**,
which is the main reason to prefer it over a flat CC map.

| MIDI | Meaning |
|---|---|
| NRPN MSB (CC 99) bits 6:4 | chassis 0–7 |
| NRPN MSB (CC 99) bits 3:0 | slot 0–15 |
| NRPN LSB (CC 98) | parameter register `0x00–0x7F` |
| Data Entry MSB/LSB (CC 6/38) | 14-bit value |
| Data Increment/Decrement (CC 96/97) | nudge by one step |

The NRPN LSB range and the module parameter range are the same 0–127 by
construction, so the LSB *is* the register number.

**CC 96/97 are a real bonus**: they are the standard relative-control messages,
which is exactly the semantic this range already has in hardware, since every
control is an incremental encoder.

- **14→16-bit widening is `(v << 2) | (v >> 12)`** — exact at both endpoints,
  one instruction.
- **`NRPN MSB = 127` is reserved for system commands** addressed to MC-1
  itself: preset save, preset recall, panic, identify, bus rescan, enter
  bootloader for slot N. It costs one slot in a chassis nobody will build.
- **Program Change recalls a preset.**
- **MIDI channel selects the voice chain**, i.e. which MC-1, consistent with
  the existing daisy-chain design.

**MC-1 is a MIDI decoder, not a MIDI repeater.** It runs the NRPN state machine
and pushes clean `(slot, register, value)` triples. Slaves never see MIDI
semantics, so adding a second control surface later touches no module firmware.

### ⚠️ Why coarse and fine tune are separate parameters

14 bits over ±5V is ~0.7 cent per step, which is audible on a sustained note.
VO-1's TUNE and FINE being separate parameters is not only ergonomics — it is
what gets the resolution under one cent. Any future parameter needing better
than 14 bits must split the same way.

---

## 10. Discovery, errors and recovery

- **At power-on MC-1 scans `0x20–0x2F`** and reads `MODULE_TYPE` from each
  responder, building the map dynamically. Its firmware never needs rebuilding
  for a given rack layout.
- **Scanning retries.** Modules do not necessarily boot before the master;
  rediscovery runs on a slow timer rather than once at startup.
- **A missing module NAKs.** MC-1 marks the slot absent and retries on the
  rediscovery timer, not on every transaction.
- **Bus recovery**: if SDA is stuck low, the master issues 9 clock pulses to
  free it. Each module runs an I2C watchdog that resets its own peripheral if
  the bus has been stuck beyond a few hundred milliseconds.
- **Clock stretching is bounded by design rule**, not by hope. No slave may
  stretch beyond ~1ms. Flash writes run from SRAM or with the peripheral
  disabled.

---

## Open items

- Verify the G0B1's I2C pin 5V tolerance, with VDD off, against the datasheet.
- Confirm the STM32G0 I2C bootloader address and pin set against AN2606.
- `CAPABILITIES` and `STATUS` bitfield definitions.
- CX-1 is specified here only as far as the protocol requires; it has no module
  folder yet and is not in phase 1 scope.
- Whether `PROTOCOL_VERSION` mismatch should refuse or degrade.
