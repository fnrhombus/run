# Feature wishlist — `run`

Running / training app. Captured from design discussion. Status legend:
`idea` (just noted) · `scoped` (design agreed) · `building` · `done`.

Context driving the project:
- Existing apps either paywall features or compute pace poorly.
- Hardware on hand: Bluetooth optical HR monitor (Scosche Rhythm 24, see
  `scosche-rhythm24.md`), Android phone.
- Candidate sensors discussed: foot pod (BLE RSC `0x1814`), sacrum/belt IMU,
  respiration band (RIP), barometer/grade, SmO2 (NIRS), core temp, CGM.

---

## Features

<!-- New features appended below as the user describes them. -->

### F1 — Haptic HR-zone coaching · `idea`

Phone vibrates to tell the runner to speed up or slow down so they hold a
target heart-rate zone **without looking at the screen** (phone strapped to
upper arm).

- **Input:** live HR from the BLE monitor (Scosche `0x2A37`), compared against
  a target zone (lo/hi bpm bounds, or % of HRmax / HRR).
- **Output:** distinct vibration patterns — e.g. one buzz pattern = "speed up"
  (HR below zone), another = "slow down" (HR above zone), silence = in zone.
  Patterns must be distinguishable by feel alone through a sleeve.
- **Design notes / open questions:**
  - HR lags effort by 10–30 s; cue off a smoothed HR + rate-of-change so it
    doesn't nag during normal zone wobble. Add hysteresis / a dead-band and a
    minimum re-alert interval so it isn't buzzing constantly near a boundary.
  - Distinguish "drifting out" (gentle reminder) from "way out of zone"
    (stronger/urgent pattern)?
  - Android haptics via `expo-haptics` are limited to preset styles; rich
    custom vibration patterns need the native `Vibration` API
    (`Vibration.vibrate([pattern])`) — confirm against v56 docs.
  - Screen will be off / pocketed-equivalent on the arm — vibration must fire
    reliably with the app backgrounded/asleep (foreground service / keep-alive
    during an active run).
  - Future: same haptic channel could cue target *pace* zones once foot-pod
    fusion exists (see sensor notes), not just HR.
