# Hardware — build / buy / skip

DIY-vs-purchase decisions for the `run` project's sensors. Constraints:
- User can **fab PCBs at home** and is comfortable programming **ESP32 / Pi Pico**.
- **Price is king.** Prefer the cheapest option that still gives trustworthy data.
- User originally intended to DIY *all* body sensors.

## Guiding principle

For every sensor, the **PCB + microcontroller is the cheap, easy part**. The real
cost is one of two things:

- **(a) firmware / signal-processing / validation** — DIY *wins*, because that's
  the IP we're building anyway (and it's where the value is).
- **(b) a precision analog front-end** — DIY *loses*, because a mass-produced
  calibrated front-end beats a home build on both price and accuracy.

---

## DEFINITELY DIY — buying is a waste of money

### IMU nodes — foot pod(s) + sacrum/belt sensor
- ~$5–15/node vs $200+ (Stryd) or locked in a $250 watch ecosystem.
- Hardware is trivial (I²C/SPI + a few parts); all difficulty is firmware: step
  detection, stride length, ground-contact time, vertical oscillation, and a
  Madgwick/Mahony fusion filter — exactly the IP we want for F2 + running
  dynamics.
- **Chip picks:** ICM-42688-P, BMI270, or LSM6DSV (modern, low-power, low-noise).
  Avoid the MPU-6050 (old, drifty).
- **Tip:** design ONE generic "body node" (MCU + IMU + LiPo + charger) and reuse
  the same board for both feet and the belt — scale on your own fab run.

### Respiration band
- Resistive/capacitive **stretch sensor** around the chest into the ESP32 ADC →
  respiratory rate + pattern for a few dollars vs. hundreds (Hexoskin).
- True inductance RIP is harder (LC oscillator + frequency counter) and
  unnecessary; stretch is plenty for rate + relative depth.
- The only hard part is tidal-volume *calibration* — a software problem.

### Barometer (grade) — DIY-able, but just use the phone
- BMP390/BMP581 is a ~$3 I²C chip, but the Android phone already has one.
- Only build a dedicated unit later if a body-mounted high-precision barometer is
  wanted. For now: skip the build, use the phone.

---

## BUY (or read commercial data) — DIY not worth it

### SmO₂ / muscle oxygen (NIRS) — the #1 do-not-DIY
- Needs precise multi-wavelength LEDs, calibrated photodiodes, a low-noise
  transimpedance front-end with ambient-light rejection, and a validated
  modified-Beer-Lambert model with pathlength correction. This is what
  Moxy/Train.Red spent years on.
- DIY (MAX30101-style) gives relative trends at best, not validated SmO₂%.
- **Verdict:** buy (Train.Red = cheaper validated option) if you need real SmO₂,
  else skip. Even buying is expensive — weigh against "price is king."

### CGM / glucose — buy-only, no exceptions
- Enzymatic electrochemical filament inserted subcutaneously; FDA-regulated,
  sterile. Cannot be DIY'd.
- BUT the **readout** is DIY: open-source tooling (xDrip+, Juggluco) reads the
  BLE/NFC off commercial sensors (Libre / Supersapiens). Buy disposable, read
  with open software.

### Optical HR — already solved (Scosche), don't build
- Motion-artifact rejection on optical HR during running is genuinely hard
  (why wrist HR is bad for running). Already own a good one + reverse-engineered it.
- *Optional:* a single-lead ECG (**AD8232**, ~$5) on a chest strap is *easier*
  to get clean beat timing/HRV from than optical — DIY-able if better RR is ever
  wanted, but the Scosche already provides RR.

---

## SKIP (for now) — bad price/value either way

### Core body temperature
- Skin temp is trivial DIY (TMP117/MLX90614) but skin ≠ core and is low-value.
- True core needs an ingestible pill or validated heat-flux sensor (CORE,
  ~$280). Skip until it earns its place.

### Gas-exchange / VO₂ analysis
- Calibrated lab instrument; don't DIY. Wearable (VO₂ Master) is expensive.
- Treat as an occasional lab test, not owned hardware.

---

## Cross-cutting: power management
- ESP32 BLE is power-hungry for all-day wear. Fine for prototyping
  (ESP32-C3/S3, cheap, BLE built in).
- For 24/7 wear (F4), **nRF52-class** chips are the gold standard for low-power
  BLE wearables — dramatically better battery life. Not a blocker; just the
  direction to head if a node must run for days.
- Real DIY effort sinks (budget for these, not the electronics): battery/charge
  management, sweat-proof enclosure + body mounting (IP rating, impact), and a
  consistent BLE GATT profile across nodes (prefer standard services like RSC
  `0x1814` where they exist — see `scosche-rhythm24.md` for the reverse-eng
  playbook on custom ones).

---

## Summary table

| Sensor | Decision | Why |
|---|---|---|
| Foot pod IMU(s) | **DIY** | cheap chip, value is in firmware (F2/dynamics) |
| Belt/sacrum IMU | **DIY** | same generic node design |
| Respiration band | **DIY** | stretch sensor + ADC, cheap |
| Barometer | **Use phone** | already on the Android |
| Running power | **DIY (compute)** | derived from IMUs you're already building |
| SmO₂ / NIRS | **Buy or skip** | precision analog FE + validation too hard |
| CGM / glucose | **Buy + DIY readout** | sensor regulated; read via xDrip+/Juggluco |
| Optical HR | **Already bought** | Scosche; motion rejection is hard |
| ECG HR (optional) | DIY if wanted | AD8232, cleaner RR than optical |
| Core temp | **Skip** | skin≠core; true core expensive/low-value |
| Gas/VO₂ | **Skip / lab test** | lab-grade only |
