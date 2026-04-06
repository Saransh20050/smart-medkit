# Smart Medkit

> Arduino-based medicine reminder with physical lid confirmation, on-device scheduling, and serial log analyser — no smartphone, no app, no internet required.

---

## The problem it solves

Most pill reminders just beep. They have no way to know if you actually took your medicine. The Smart Medkit closes that loop: the alarm only stops when you physically open the lid, detected by a magnetic reed switch. Every event is logged to serial and can be reviewed through a companion web dashboard.

---

## Features

- **Reed switch confirmation** — buzzer + LED alarm continues until the lid is physically opened; it doesn't just time out
- **DS3231 RTC** (±2 ppm) with CR2032 battery backup — survives power cuts without losing alarm times
- **2 independent medicine slots**, each with its own configurable alarm time
- **On-device configuration** — set alarm times using just 2 buttons and the 16×2 LCD; no PC, no app, no serial terminal needed
- **Anti-flicker LCD rendering** — differential draw engine; `lcd.clear()` is never called in the main loop
- **Auto daily reset** — `medTaken` and `alarmFired` flags clear automatically at midnight
- **Full serial event log** — every boot, config change, alarm fire, and completion is timestamped and logged
- **MedKit Log Analyser** — paste a serial dump into the companion web dashboard and get a structured session report

---

## Hardware

| Component | Spec | Pin(s) |
|---|---|---|
| Arduino Uno R3 | ATmega328P, 16 MHz, 5V | — |
| DS3231 RTC module | ±2 ppm, I2C, CR2032 backup | SDA→A4, SCL→A5 |
| 16×2 LCD | HD44780 compatible, 5V | RS=12, EN=11, D4–D7=5,4,3,2 |
| Reed switch ×2 | Normally-open, 5V logic | A1, A2 (INPUT_PULLUP) |
| Piezo buzzer | Passive, 2 kHz @ 5V | D7 |
| Red LED ×2 | 3 mm, 2V Vf, 20 mA | D8, D9 (via 220 Ω) |
| Push button ×2 | 6×6 mm tactile, pull-up | D13 (NEXT), D10 (CHANGE) |
| Neodymium magnet ×2 | 5×2 mm disc | Glued to lid |

**Total BOM cost: ~₹575 (~$7)**

---

## Wiring

```
Arduino Uno
│
├── D2  ← Push button (BTN_NEXT)      [INPUT_PULLUP, active LOW]
├── D7  → Piezo buzzer                [tone() @ 2000 Hz]
├── D8  → LED slot 1                  [via 220 Ω]
├── D9  → LED slot 2                  [via 220 Ω]
├── D10 ← Push button (BTN_CHANGE)    [INPUT_PULLUP, active LOW]
├── D11 → LCD EN
├── D12 → LCD RS
├── D2–D5 → LCD D4–D7
├── A1  ← Reed switch slot 1          [INPUT_PULLUP, NO type]
├── A2  ← Reed switch slot 2          [INPUT_PULLUP, NO type]
├── A4  ↔ DS3231 SDA + LCD SDA        [I2C shared bus]
└── A5  ↔ DS3231 SCL + LCD SCL        [I2C shared bus]
```

---

## How it works

### Normal operation
The LCD shows a live `HH:MM:SS` clock on line 1 and both alarm times on line 2:
```
 06:04:33
1>06:00  2>21:00
```

### Alarm sequence
When the current time matches a slot's alarm time:
1. LCD shows `* TAKE YOUR MED * / [ SLOT 1 ]`
2. Buzzer pulses at 2 kHz (400 ms ON / 300 ms OFF) and LED flashes
3. If the reed switch is closed (lid shut) → alarm loops until lid is opened
4. If reed is absent or already open → 15 fixed buzz cycles as fallback
5. On lid open → buzzer and LED cut immediately, `medTaken[slot] = true`
6. LCD confirms `Slot 1: Done! / Med taken! :)` for 2.5 s

### Configuring alarm times
Press `BTN_NEXT` in idle mode to enter the config wizard:

```
Step 0 → Select slot (CHANGE cycles 1/2, NEXT confirms)
Step 1 → Set hour  (CHANGE increments 0–23, NEXT confirms)
Step 2 → Set minute (CHANGE increments 0–59, NEXT saves)
```

No computer, no app, no serial terminal — entirely on-device.

---

## State machine

```
                     ┌─────────────────────────────────────────────┐
                     │                                             │
          BTN press  ▼                  SET                        │
CONFIG ◄────────── IDLE ──────────────────────────────► CONFIG     │
                     │                                             │
         alarm time 1│                        alarm time 2│        │
                     ▼                                    ▼        │
                  RING₁                               RING₂        │
              (buzz + LED)                        (buzz + LED)     │
                     │ open lid                         │ take med │
                     ▼                                  ▼          │
               STOP_PING₁                         STOP_PING₂       │
               (lid open)                       (LED/buzz off)     │
                     │ close lid                        │          │
                     └──────────────────────────────────┘          │
                                      back to IDLE ────────────────┘
```

---

## Serial log

Every event is printed at 9600 baud. Tags:

| Tag | When | Example |
|---|---|---|
| `[BOOT]` | Power-on after RTC seed | `[BOOT] lastDay seeded to: 6` |
| `[RTC]` | Clock reset after power loss | `[RTC] Reset to compile time` |
| `[CONFIG]` | Each wizard step | `[CONFIG] Saved slot 1 -> 06:00` |
| `[ALARM]` | Alarm triggered / resolved | `[ALARM] Slot 2` / `[ALARM] Done - slot 2` |
| `[SYSTEM]` | Day rollover | `[SYSTEM] New day: 7` |

---

## Log Analyser

Capture a session's serial output from the Arduino IDE Serial Monitor, paste it into the **MedKit Log Analyser** web dashboard, and click **Generate report**.

The dashboard shows:
- Session health (Healthy / Warning)
- Alarms fired, configs saved, day resets, boot day
- Per-slot alarm times with fire counts
- Per-alarm outcomes (Completed / Missed)
- Full event timeline

> The log analyser is a standalone HTML/JS file — open it in any browser, no server needed.

---

## Dependencies

Install via Arduino Library Manager:

- [`RTClib`](https://github.com/adafruit/RTClib) by Adafruit
- `LiquidCrystal` (built-in with Arduino IDE)
- `Wire` (built-in)

---

## Getting started

```bash
# 1. Clone
git clone https://github.com/YOUR_USERNAME/smart-medkit.git

# 2. Open in Arduino IDE
#    File → Open → smart-medkit/smart_medkit.ino

# 3. Install RTClib via Library Manager

# 4. Select board: Arduino Uno, correct COM port

# 5. Upload
```

On first boot (or after battery replacement), the RTC is automatically set to the compile time.

To open the log analyser:
```bash
open log-analyser/index.html   # or just double-click it
```

---

## Repo structure

```
smart-medkit/
├── smart_medkit.ino        # Main Arduino sketch
├── log-analyser/
│   └── index.html          # Standalone web log analyser
├── docs/
│   ├── state-diagram.png
│   ├── wiring-diagram.png
│   └── SmartMedkit_ProjectReport.pdf
└── README.md
```

---

## Tech stack

- **Firmware** — Arduino C++ (RTClib, LiquidCrystal, Wire)
- **Log analyser** — Vanilla HTML/CSS/JS, no dependencies
- **RTC** — DS3231 over I2C
- **TRL** — 4 (lab validated)

---

## License

MIT — free to use, modify, and build on.
