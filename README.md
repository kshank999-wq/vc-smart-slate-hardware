# VC Smart Slate — hardware integration (draft 0.1)

> **Public copy.** This is the VC Smart Slate hardware integration document,
> published on its own for the slate board's developer and their tools. The
> app's source is private; paths below (`SlateCore/…`, `docs/spec/…`) refer
> to it. The master copy is `docs/hardware/HARDWARE-INTEGRATION.md` there,
> and this copy is updated whenever it changes.

For the developer of the VC Smart Slate hardware (the slate with sticks and
its own board). It says what the board must send and receive so that the
VC Smart Slate app on an iPhone or iPad records each clap from the sticks,
frame-accurately, exactly once, and shows the hardware's state.

**Status.** Draft for the hardware developer to work against and correct.
The app's side is built to this draft: the packets, the clock matching, the
Bluetooth connection, a simulated board, and a test screen in the app
(Settings › Hardware slate, section 13). Changes to the protocol should be
agreed before either side builds against them. Open questions are at the
end; answers there change the draft.

Owner: Ken Shank. Repository: `kshank999-wq/vc-smart-slate`. Spec: section
10 of `docs/spec/vc-smart-slate-master-development-spec.md` ("Physical slate
hardware integration (separate SKU): authenticated paired event trigger …
hardware event ID, sequence and acknowledgement so double transmissions do
not produce extra clap records … operate even if hardware is disconnected")
and its test T12.

---

## 1. The arrangement

```
 ┌──────────────────────┐   Bluetooth LE (GATT)   ┌──────────────────────────┐
 │  Slate hardware      │ ──────────────────────▶ │  VC Smart Slate app       │
 │  (peripheral)        │   claps, buttons,       │  (central: iPhone / iPad) │
 │  sticks sensor       │   status, battery       │  journal, timecode,       │
 │  board clock         │ ◀────────────────────── │  Film Studio link         │
 │  optional display    │   acks, time sync,      │                           │
 │                      │   card / timecode       │                           │
 └──────────────────────┘                         └──────────────────────────┘
```

- **The board is a BLE peripheral; the app is the central.** One app
  connects to one board at a time; a board is bonded to the devices that
  paired with it.
- **The app is the record.** It owns the day: scenes, takes, claps,
  timecode, sending to Film Studio. The board reports what happened and
  when, by its own clock; it keeps nothing the app needs to rebuild a day,
  except claps not yet acknowledged.
- **Either side may be missing.** The app works with no hardware (the touch
  sticks on screen). The board must keep capturing claps when the app is
  out of range and deliver them when it reconnects.

## 2. What the app already does

In `SlateCore` (pure Swift, builds and tests on macOS and Linux):

- A clap carries its **trigger**: `ClapTrigger.hardware(deviceID:, eventID:)`
  (`SlateCore/Sources/SlateCore/Domain/Entities.swift`).
- **A retransmission is one clap.** A second clap with the same
  `deviceID` + `eventID` returns the clap already stored; it is never
  recorded twice (`SlateJournal.clap`, `SlateCore/Sources/SlateCore/Domain/SlateJournal.swift`).
  Tested as spec T12.
- **Two different presses closer than 250 ms on one take** are both kept
  and flagged `possibleDoubleTap` for review, never merged.
- Claps are append-only: never edited, renumbered or deleted; a corrected
  timecode is a separate audited record.
- Timecode is integer frame counts and rational rates (24000/1001 and so
  on), never floating-point frames. 23.976, 24, 25, 29.97 (NDF/DF), 30, 50,
  59.94 (NDF/DF), 60.

So the board's job is: a stable **device ID**, a unique **event ID** per
press, an accurate **board timestamp** of the sensor edge, and resending
until acknowledged.

## 3. Bluetooth

- Bluetooth LE 5.0 or later; 2M PHY preferred, 1M accepted.
- **Pairing:** LE Secure Connections, bonded. Every VC characteristic
  requires an encrypted link. The app pairs from its Settings, shows the
  board's name and serial, and keeps the bond; an unpaired central gets
  nothing.
- **MTU:** the board should accept 185 bytes or more (iOS asks for it).
  Event, Event Ack and Status fit the default 20-byte payload; the Pong,
  Card and Timecode packets need the larger MTU, so the board sends them
  only after the MTU exchange.
- **Connection interval:** the board should accept iOS's 15–30 ms. Latency
  of delivery does **not** matter for accuracy (section 5); only the board's
  timestamp does.
- **Advertising:** the VC service UUID in the advertisement, local name
  `VC Slate XXXX` (last four of the serial), every 100–250 ms while
  unconnected and awake.
- iOS cannot stay connected with the app fully closed; the board must not
  depend on that. Claps while the app is away are kept and resent (section 6).

### Standard services

| Service | UUID | Characteristics used |
|---|---|---|
| Device Information | `0x180A` | Manufacturer, Model, **Serial Number** (the device ID), Firmware Revision, Hardware Revision |
| Battery | `0x180F` | Battery Level (notify) |

### VC Slate service

Service UUID **`67de563f-ea8d-4cdb-993b-bdb3fba387a6`**

| Characteristic | UUID | Properties | Direction | Section |
|---|---|---|---|---|
| Event | `9e3926ec-6abb-4974-960e-2fb67fbb97cf` | Indicate | board → app | 4 |
| Event Ack | `46766f85-e5e6-4bff-8ed8-280979cadefd` | Write (with response) | app → board | 6 |
| Time Sync | `eef38df8-601a-4ac6-b891-ee9d02a4f890` | Write, Notify | both | 5 |
| Status | `96550b20-8e21-4dca-9a56-a2956b705f50` | Read, Notify | board → app | 7 |
| Card | `5bd09867-5af1-41c1-b0e4-077c52087beb` | Write (with response) | app → board | 8, optional |
| Timecode | `5ac4de3d-e86b-431e-9b9b-3d22479b7591` | Write (with response) | app → board | 8, optional |
| File | `0bde5362-dbb7-45d4-9a47-34d5cab7138b` | Write (with response) | app → board | 9, SD card |

All multi-byte fields are **little-endian**. Every packet starts with a
protocol version byte, `0x01` for this draft; a side that sees a version it
does not know ignores the packet and reports it in Status.

## 4. Events: the sticks and the buttons

One packet per event, sent by **indication** (so the link layer confirms
delivery as well as the app's own ack).

| Offset | Size | Field | Meaning |
|---|---|---|---|
| 0 | 1 | `version` | `0x01` |
| 1 | 1 | `kind` | see below |
| 2 | 4 | `boot_id` | random per power-up, or a counter kept in flash |
| 6 | 4 | `sequence` | 1, 2, 3 … per event since boot, never reused within a boot |
| 10 | 8 | `tick_us` | board clock at the **sensor edge**, microseconds since boot |
| 18 | 1 | `flags` | bit 0 resend; bit 1 board clock not synced yet; bit 2 debounce extended |
| 19 | 1 | `reserved` | 0 |

20 bytes.

`kind`:

| Value | Event | The app does |
|---|---|---|
| `0x01` | **Sticks closed** | the clap: head slate, second sticks, or a marker on MOS, as the touch sticks do now |
| `0x02` | Sticks opened | shows the sticks open; starts nothing |
| `0x10` | ROLL button | ROLL, if the board has one |
| `0x11` | CUT button | CUT |
| `0x12` | Tail-slate button | the next close is a tail slate |
| `0x13` | Next take (›) | ends the take, next take number |
| `0x20` | Sync button | asks the app for a time sync and timecode now |

The app records a sticks-closed event as
`ClapTrigger.hardware(deviceID: <serial>, eventID: "<boot_id hex>-<sequence>")`.
That pair is the identity of the press: the same pair again is the same clap.

**The edge.** `tick_us` is the instant the sticks **meet**, read in the
sensor's interrupt, before any debounce delay. Debounce decides whether to
report; it must not move the timestamp. Bounce after the first edge within
the debounce window (20 ms suggested; the developer to set it from the
sensor) is not a new event.

## 5. Time sync: frame accuracy without trusting the radio

A BLE packet can arrive 10–60 ms after it was sent, and the delay varies.
At 24 fps a frame is 41.7 ms, so **the app never uses arrival time**. It
converts the board's `tick_us` to its own clock with a measured offset, and
reads the timecode at that instant.

The exchange (like NTP's four timestamps):

1. App writes **Ping** to Time Sync: `version`, `0x01`, `nonce` (u32),
   `t1` = app clock (u64 µs). 14 bytes.
2. Board notifies **Pong** at once: `version`, `0x02`, `nonce`,
   `t2` = board tick on receipt (u64 µs), `t3` = board tick on sending
   (u64 µs). 22 bytes; fits a 185-byte MTU (the board must not send it
   before the MTU exchange).
3. App notes `t4` on receipt. Round trip = (t4 − t1) − (t3 − t2); offset =
   ((t2 − t1) + (t3 − t4)) / 2.

The app pings on connect, every 5 s while connected, and on a Sync button
press, keeps the offsets of the lowest-round-trip exchanges, and fits drift
from them. A clap's board tick maps to app time as `tick_us − offset`,
corrected for drift since the last exchange.

**Requirements on the board clock:** a free-running microsecond counter
(not reset while powered, monotonic, 64-bit or extended to 64), crystal
accuracy of **±20 ppm or better** (2 ms in 100 s between syncs at worst),
timestamp latency from edge to latch under 100 µs.

A clap captured before the first sync (flag bit 1) is kept; the app maps it
from the first sync after, and marks it if the gap is long.

## 6. Acknowledgement and resending

The board keeps every event in a queue until the app acknowledges it.

**Event Ack** (app → board, write with response):

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `version` |
| 1 | 4 | `boot_id` |
| 5 | 4 | `sequence` |
| 9 | 1 | `result`: 1 recorded, 2 duplicate (already recorded), 3 refused |
| 10 | 2 | `take_number` (0 if none) |
| 12 | 4 | `timecode_frames` (frame count at the clap; 0xFFFFFFFF if none) |

16 bytes. `recorded` and `duplicate` both end the board's interest in the
event. `refused` (no scene on the slate, say) also ends it; the board shows
it if it has a display.

**Rules for the board:**

- Keep at least **64 unacknowledged events**, across power cycles if it
  can (flash), oldest dropped first only if full; report a drop in Status.
- On reconnect, resend every unacknowledged event in sequence order with
  the **same `boot_id` and `sequence`** and flag bit 0 set.
- Resend an event not acknowledged within 2 s while connected.
- Never reuse a `boot_id` + `sequence` pair for a different press.

**Rules for the app (to build):** acknowledge only after the clap is saved
on the device (the app's existing rule: nothing is acknowledged before it is
on disk). A resend of a recorded event is answered `duplicate`, and nothing
new is written.

## 7. Status

Read, and notify on change.

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `version` |
| 1 | 1 | `sticks`: 0 open, 1 closed, 2 unknown |
| 2 | 1 | `battery_percent` (also in the Battery service) |
| 3 | 1 | `charging`: 0/1 |
| 4 | 2 | `queued_events` (unacknowledged) |
| 6 | 2 | `dropped_events` since boot |
| 8 | 4 | `boot_id` |
| 12 | 2 | `last_error` code (0 none; codes to be listed by the developer) |
| 14 | 1 | `temperature_c` (signed, if measured) |
| 15 | 1 | `reserved` |

16 bytes. The app shows the board's connection, battery, queue and errors
on its Sync and Settings screens, and the trigger latency it measures.

## 8. Optional: the board's own display and timecode

Only if the hardware has a display or timecode digits of its own (open
question 1). The app sends:

**Card** (app → board), UTF-8, length-prefixed strings, sent whenever the
slate card changes and on connect:

`version`, then fields as `u8 key, u8 length, bytes`: 1 production, 2 scene
("14B"), 3 take, 4 slate number, 5 roll (camera under CAM), 6 camera letter,
7 director, 8 DP, 9 date, 10 flags as text ("INT DAY SYNC"). A packet longer
than the MTU is split: byte 1 is `part / parts`.

**Timecode** (app → board): an anchor for the board to run its own
generator from, so its digits never wait on the radio:

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `version` |
| 1 | 4 | `rate_numerator` (24000, 25, 30000 …) |
| 5 | 4 | `rate_denominator` (1001, 1 …) |
| 9 | 1 | `drop_frame` 0/1 |
| 10 | 4 | `frame_count` at the anchor |
| 14 | 8 | `tick_us` on the board's clock at that frame (already converted by the app) |
| 22 | 1 | `lock_state`: 0 locked, 1 holdover, 2 unsynced, 3 manual |

23 bytes (needs the 185-byte MTU). The board counts frames from the anchor
with integer arithmetic on its microsecond clock and shows HH:MM:SS:FF
(`;` before frames for drop frame). The app re-anchors every 10 s and on
any change of source or rate. The board shows a lock state other than
locked plainly (blinking digits, say), as the app does.

## 9. The SD card: the facing page on board

The board keeps the day's paperwork on a microSD card (FAT32 or exFAT), so
it survives without the phone and can be read in any computer. Everything
lives under a top-level folder `VCSLATE/`, one folder per shooting day:

```
VCSLATE/
  2026-11-07 Day 3/
    facing-page.csv     written by the app (below), replaced after each take
    events.csv          written by the board itself, one line per event
```

**`facing-page.csv`** is the script supervisor's facing page (description
page): one block per setup with its description, lens, camera and sound
rolls and frame rate, then one row per take with PRINT / HOLD / NG, time of
day, timecode in, running time, rolls and comments. The same rows as the
app's Facing page PDF and the supervisor's page in VC Film Studio. The app
writes it over the **File** characteristic after every recorded clap, CUT
and next take, and on connecting.

**File** (app → board, Write with response). Every packet:

| Offset | Size | Field |
|---|---|---|
| 0 | 1 | `version` |
| 1 | 1 | `op`: 1 BEGIN, 2 DATA, 3 COMMIT, 4 ABORT |
| 2 | 2 | `transfer` id, the same on every packet of one file |
| 4 | 4 | BEGIN: file length · DATA: byte offset · COMMIT: CRC-32 · ABORT: 0 |
| 8 | … | BEGIN: the path, UTF-8, relative to `VCSLATE/`, at most 120 bytes · DATA: the bytes |

The app sends BEGIN, then DATA pieces in order that fill each write (MTU
less 3), then COMMIT. The board:

1. On BEGIN, opens `path.part` (creating the day's folder), forgetting any
   transfer still open.
2. On DATA, appends if `offset` is what it has so far; anything else
   abandons the transfer (deletes `path.part`).
3. On COMMIT, checks the length and the CRC-32 (IEEE 802.3, as zlib's
   `crc32`), then renames `path.part` over `path`. Only then is the old copy
   replaced, so a transfer cut off by the radio or a flat battery never
   leaves a half-written facing page.
4. Refuses a path that starts with `/` or contains `..`.

**`events.csv`** is the board's own record, written whatever the app does:
`boot_id,sequence,kind,tick_us,flags,result,take,timecode_frames`, a line
when the event happens and a second line with the ack's result, take
number and timecode once the app answers (section 6). With no app all day
it still lists every clap in order with the board's own time.

The reference for both sides is `SDCard` and `FilePacket` in
`SlateCore/Sources/SlateCore/Hardware/HardwareProtocol.swift`, tested in
`SDCardTests` (the CRC check value, the packets byte for byte, a cut-off
transfer, a bad CRC). The simulated board on the test screen keeps an SD
card in software and lists what the app wrote to it.

## 10. What the board must do with no app

- Capture every close with its timestamp and keep it (section 6), and write
  it to `events.csv` on the SD card.
- If it has digits: keep running timecode from the last anchor, marked
  holdover; never show a time it was not given.
- Sleep when idle; wake on the sticks opening.

## 11. Tests

To pass before the app and board are called working together. Each is run
with the app's test screen (to build: live packets, offsets, round trips,
latencies) and a reference.

| # | Test | Pass |
|---|---|---|
| H1 | Pair, bond, reconnect after power cycle | reconnects without re-pairing; an unbonded phone sees nothing |
| H2 | Clap accuracy | sticks filmed at 120 fps beside a timecode display: the clap's recorded timecode within ±1 frame at 24 fps across 50 claps, ±0.5 frame median |
| H3 | Retransmission (spec T12) | the same event sent 5 times is one clap; the app answers `duplicate` |
| H4 | Out of range | 10 claps with the app away; all 10 recorded once on reconnect, in order, with their own timestamps |
| H5 | Power loss with events queued | after restart, queued events resent (if kept in flash) or the loss reported in Status |
| H6 | Bounce | 200 claps, hard and soft: no extra events, no missed closes |
| H7 | Double tap | two real presses 150 ms apart: two claps, flagged `possibleDoubleTap` |
| H8 | Clock drift | 4 hours connected: offset drift within ±20 ppm; timecode on the board's digits within 1 frame of the app's |
| H9 | Battery | a 12-hour day with the display on, from full |
| H10 | Range | 10 m line of sight with a body between: no lost claps (they may arrive late) |
| H11 | SD card | after 20 takes, `facing-page.csv` on the card matches the app's Facing page CSV; pulling power mid-transfer leaves the previous copy whole; `events.csv` lists every clap |

## 12. Open questions for the hardware developer

1. **Does the board have its own display or timecode digits?** If yes,
   which: the full slate face, LED timecode only, or e-ink? (Section 8.)
2. **The sticks sensor:** contact, Hall effect, accelerometer? Its bounce
   and its edge-to-interrupt latency.
3. **Buttons:** which physical buttons, and which of section 4's kinds
   they are.
4. **The chip and radio:** which BLE SoC, which stack. Is Wi-Fi present?
5. **Clock:** crystal and its ppm; is there a 32 kHz RTC kept in sleep?
6. **Flash:** can the event queue survive power loss? Is there a microSD
   slot (section 9), and which file system?
7. **Timecode in:** does the board read LTC itself (a 3.5 mm or Lemo
   input)? If so, it can stamp claps with timecode directly, and the
   Event packet gains a timecode field.
8. **Firmware updates:** over BLE (needs a DFU service), or by cable?
9. **Power:** battery size, charging, sleep current.

Answers go back into this document; the app's side is then built to it,
starting with the test screen.

## 13. The app's side, and the test screen

Built to this draft (31 October 2026):

- `SlateCore/Sources/SlateCore/Hardware/` — the packets
  (`HardwareProtocol.swift`), the clock matching (`BoardClock.swift`: the
  quicker half of the exchanges, a line through them for drift), and a
  board in software (`BoardSimulator.swift`). Pure Swift, tested on macOS
  and Linux.
- `SlateCore/Sources/SlateKit/HardwareMonitor.swift` — turns events into
  claps on this device's clock at the instant the sticks met, answers each
  with an ack, refuses a clap before the first time sync rather than
  misplace it, and keeps the log the test screen shows.
- `App/Sources/HardwareLink.swift` — Core Bluetooth: scans for the VC Slate
  service, connects, subscribes to Event, Time Sync and Status, reads the
  serial number (the device ID claps carry), writes Event Ack and pings
  every 5 s. The simulated board, with a radio of 10 to 60 ms each way.
- **The test screen**, Settings › Hardware slate, in this order:
  - **Board**: find boards advertising the VC Slate service (name and
    signal strength), connect, disconnect; or *Use the simulated board*.
    Once connected: the board's name and serial number.
  - **Simulated board** (when it is the one connected): the last event at
    the top — what was pressed, how late the radio delivered it, and what
    the slate did with it — then buttons for every event kind (sticks
    closed and opened, tail slate, ROLL, CUT, next take) and *Out of range*,
    which holds every press on the board until it is switched off, when the
    board resends its queue and each press is still one event.
  - **Status**: sticks, battery and charging, events waiting on the board,
    events dropped since boot, boot ID, last error, temperature.
  - **Clock**: how far the board's clock is ahead of the device's, round
    trip last and best, drift in ppm, exchanges answered, the radio delay
    over the last 50 events (average and worst), and *Sync now*. Until the
    first exchange the screen says the board is not synced, and a clap is
    refused rather than placed by arrival time.
  - ***Clap the slate from the board***: off (the default) is a bench
    test — every event logged and acknowledged, so the board stops
    resending, and nothing written to the day. On, the board's sticks clap
    the slate on the instant they met, and ROLL, CUT, tail slate and next
    take act, each once however often it is resent. With it on, the facing
    page is written to the board's SD card after each take (section 9);
    the simulated board lists the files on its card under its buttons.
  - **Events**: every event, newest first: kind, RESENT if flagged, radio
    delay, what the slate did (in red if refused), and the event ID and
    board tick.
- **Which builds have it.** Development and TestFlight builds, for the
  team and the board's developer. The App Store version is built with the
  screen off (the `HARDWARE_TESTING` build setting): *Hardware slate* shows
  greyed out, "coming soon", and Bluetooth never starts, until the
  hardware ships.

**Byte fixtures** for the firmware, from `SlateCore/Tests/SlateKitTests/HardwareTests.swift`
(the app's own tests check these exact bytes):

| Packet | Values | Bytes (hex) |
|---|---|---|
| Event | sticks closed, boot `0x11223344`, sequence 7, tick `0x0102030405060708`, resend | `01 01 44 33 22 11 07 00 00 00 08 07 06 05 04 03 02 01 01 00` |
| Event Ack | same event, duplicate, take 3, frames 1 234 567 | `01 44 33 22 11 07 00 00 00 02 03 00 87 d6 12 00` |
| Ping | nonce 9, t1 1 000 000 µs | `01 01 09 00 00 00 40 42 0f 00 00 00 00 00` |
| Status | closed, 87 %, charging, 2 queued, boot `0x11223344`, −5 °C | `01 01 57 01 02 00 00 00 44 33 22 11 00 00 fb 00` |
| File | `a.csv` containing `hello`, transfer 7: BEGIN, DATA, COMMIT | `01 01 07 00 05 00 00 00 61 2e 63 73 76` · `01 02 07 00 00 00 00 00 68 65 6c 6c 6f` · `01 03 07 00 86 a6 10 36` |

CRC-32 check value: `"123456789"` gives `cbf43926` (`SDCardTests`).

Not built yet: the Card and Timecode packets (section 8), which wait on
open question 1.
