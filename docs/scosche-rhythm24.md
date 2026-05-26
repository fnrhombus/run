# Scosche Rhythm 24 — BLE protocol notes

Live BLE protocol for the Scosche Rhythm 24 optical HR armband, reverse-engineered
from the device + the official `ScoscheSDK24` iOS framework (the SDK is a thin
UInt16 pass-through wrapper — semantics live in the device firmware).

This is what the **`run`** app needs to know to consume HR + per-beat RR data
from the device. For data science / HRV analysis, see the FIT-file section.

---

## Device identification

- Advertising name: `Rhythm 24 <NNNN>` where `NNNN` is a 4-digit device serial.
- Standard BLE services exposed: HR Service `0x180D`, Battery `0x180F`,
  Device Info `0x180A`, Current Time `0x1805`.
- Scosche/Valencell proprietary services:
  - `fce61000-7802-4392-b6b7-96b18deaad22` — **User Data** (config + sport mode control)
  - `fce62000-7802-4392-b6b7-96b18deaad22` — **FIT file** transfer (post-hoc session export)
  - `fce63000-7802-4392-b6b7-96b18deaad22` — Valencell firmware update
  - `fce64000-7802-4392-b6b7-96b18deaad22` — **VDC Expanded Data** (live PPG + per-beat data)

---

## Sport mode — must be set before HRV/RR streaming works

Standard `0x2A37` HR Measurement on this device only emits RR intervals when the
device is in **HRV sport mode**. In default HR mode the RR-present flag bit
(`0x10`) is never set.

Write a single `UInt8` byte to `fce61001` to set the mode:

| value | mode |
|---:|---|
| 0 | HeartRate (default) |
| 1 | Running |
| 2 | Cycling |
| 3 | Swimming |
| 4 | Duathlon |
| 5 | Triathlon |
| **6** | **HRV** ← what unlocks RR data |
| 7 | (multi-mode separator) |

The char is **write-only** despite advertising read+notify — `read_gatt_char`
returns `READ_NOT_PERMITTED`. Confirm the mode took by checking that std_hr
notifications start including the RR-present flag, not by reading the char.

Mode can also be cycled with the physical button on the device. Whatever was
written via BLE persists across BLE disconnects until next button press or next
BLE write.

---

## Channel 1: standard `0x2A37` HR Measurement

Standard BLE HRP encoding. Bit-flags byte + HR + optional RR list.

- Notifications fire at ~2 Hz regardless of mode.
- ~24% of notifications include RR intervals (1–2 per notification).
- RR units are **1/1024 second** ticks → convert to ms with `raw * 1000 / 1024`.
- The other ~76% of notifications carry HR only.

```
flags=0x06 → HR only
flags=0x16 → HR + RR present (bit 0x10)
```

---

## Channel 2: Scosche RRI cluster (proprietary)

The `fce64000` service exposes a per-beat encoding *separate from* `0x2A37`.
The encoding was not documented in the SDK (Swift wrapper only exposes raw
`UInt16` fields) and was reverse-engineered from a live capture by cross-checking
against `0x2A37`'s independently-encoded RR intervals.

| UUID prefix | label | role |
|---|---|---|
| `fce64010` | RRI Status | never observed firing in HRV mode — likely unused |
| `fce64011` | RRI Timestamp | **device-time in seconds**, 1 Hz counter, `UInt16` LE |
| `fce64012` | RRI Reg1 | **ms-offset of 1st beat** in that second window |
| `fce64013` | RRI Reg2 | ms-offset of 2nd beat in the same second |
| `fce64014` | RRI Reg3 | 3rd beat (rare — only used at high HR) |
| `fce64015` | RRI Reg4 | 4th beat (very rare) |
| `fce64016` | RRI Reg5 | 5th beat (extreme HR only) |

### Encoding (verified empirically)

```
beat_device_time_ms = (rri_timestamp_value * 1000) + rri_regN_value
```

Each register fires as a separate notification. Within a second window the
registers fire in beat order: Reg1 typically <512ms, Reg2 typically >512ms.
The host can convert device-ms to wall-clock by anchoring once: at the first
`rri_timestamp` notification arrival, compute
`offset = host_time - (timestamp_value * 1000)` and apply that offset to all
subsequent beat times.

### Validation

In a 60s capture, beats derived from this encoding agreed with `0x2A37`'s
independently-encoded RR intervals to within ~5 ms, and the derived mean HR
(96.9 bpm by beat count) matched the device's reported mean HR (96.7 bpm).

### Coverage

- Indoor at <1m line-of-sight: ~88% of beats captured from the RRI cluster alone
- At range / through body / during heavy movement: drops climb to 30%+
- Even combined with `0x2A37`, BLE drops mean realtime never reaches 100%

### Other VDC characteristics (in `fce64000`)

Not useful for per-beat timing — most are 1Hz sampled summary metrics that
correlate with HR because they all rise together during exertion (step rate,
stride rate, etc.), not because they encode beat-level data:

- `fce64001` VDCSignalData — raw PPG (never observed firing; rate-control not in SDK)
- `fce64002` VDCOpticalData — DC optical, 1Hz
- `fce64003` VDCActivityData — never observed firing
- `fce64004` VDCHeartRateData — duplicates standard HR
- `fce64005` VDCStepRateData
- `fce64006` VDCStrideRateData
- `fce64007` VDCDistanceData
- `fce64008` VDCSpeedData
- `fce64009` VDCStepsData
- `fce6400a` VDCCalorieRateData
- `fce6400b` VDCTotalCaloriesData
- `fce6400c` VDCAmbientLightData
- `fce6400d` VDCACSignalData — AC pulsatile, 1Hz (per-second amplitude, not waveform)

---

## Recommended architecture for `run`

Two-channel:

### Live (during the run)

Subscribe only to **standard `0x2A37`**. Display HR from each notification. Don't
bother with the RRI cluster for live — adds complexity, and 5–10% BLE drops are
invisible to a 1Hz HR display.

### Post-hoc (for HRV / data science)

After the run, download the session as a **FIT file** via the `fce62000` service.
The device records to onboard storage continuously, so the FIT file has every
beat the sensor detected — no BLE drops.

- `fce62001` directory/control — opcodes `0=readInfo, 1=erase, 2=download`
- `fce62002` file data channel
- Response codes: `fail / ok / recordingInProgress / downloadInProgress / downloadFinished`

FIT is an open Garmin format. Parse with the Garmin FIT SDK or
[`fitparse`](https://github.com/dtcooper/python-fitparse) for offline analysis.

---

## Open questions (deferred until implementation)

1. Does the device auto-record in HRV mode, or does it need an explicit
   start-recording command? (SDK exposes FIT *read* opcodes; recording trigger
   unconfirmed.)
2. How long does a FIT download take over BLE? A 1-hour HR-only file is ~30–50 KB
   so probably under a minute, but worth measuring.
3. Onboard storage limit and behavior when full — Scosche specs say "hundreds of
   hours" but full-storage behavior (overwrite vs stop) is undocumented.
4. Does the BLE write to `fce61001` (sport mode) persist across power-cycles, or
   does the device boot back to mode 0?

---

## Reference: working capture pipeline

While building, you can reproduce the protocol behavior from a Linux box using
the Python pipeline at `/tmp/rhythm24-capture/` (in the noop session — not
preserved long-term). Bleak + a 60s capture is enough to verify any change to
the encoding model.

Sources:
- [`scosche/ScoscheSDK24`](https://github.com/scosche/ScoscheSDK24) — UUID names
  and characteristic ownership (semantics not documented; SDK is a `UInt16`
  pass-through)
- [HRV4Training: Scosche's Rhythm24 & HRV](https://www.hrv4training.com/blog2/scosches-rhythm-24-heart-rate-variability) —
  confirms HRV mode required for RR
- [adafruit/Adafruit_CircuitPython_BLE_Heart_Rate#17](https://github.com/adafruit/Adafruit_CircuitPython_BLE_Heart_Rate/issues/17) —
  community confirmation that `0x2A37` returns empty RR in default HR mode
