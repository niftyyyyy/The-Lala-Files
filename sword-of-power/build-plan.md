# Sword of Power — Build Plan (Version 3 · simplified)

CC300 · Semester 2 · Automated Systems capstone
Prepared for Liam Cloete & team. Lecturers: Will Basson · Juan Taljaard.
Prices checked 2 Oct 2026, incl. VAT.

He-Man's Sword of Power as a film-set prop, wirelessly linked to a solid crystal that glows when the sword is swung or raised. Both props are kept simple and built mostly from parts the team already owns. Each part is chosen for what the brief marks: silent on set, the same result every take, IP22, survives a 1.5 m drop, and a real failsafe.

Version 3 follows the brief: the crystal is a solid part with electronics inside and doesn't move, and the sword stays simple. Removed: the motorised plinth, the stepper, the dust sensor and the extra boards. The plan uses the team's own parts first: ESP32-S2, ESP32, MPU6050, FSR pad, vibration motor, WS2812 strip and ring, power bank, and the kit parts. The shopping list fell from about R4 300 to about R1 760.

## 01 · The idea — two props, one system

**The sword.** The actor holds it, swings it, raises it. The blade glows and surges on every swing. A sound plays (showcase only), and a small buzz in the grip gives silent cues. Grip sensing and fall detection shut it down safely.

**The crystal.** A solid printed crystal with an LED ring and sensors inside. It glows in sync with the sword, measures the set (temperature, humidity, light, knocks) and hosts the phone dashboard. It never moves.

| Your idea | Keep it simple, but | Why |
|---|---|---|
| Buzzer for sounds | DFPlayer Mini + small speaker, with a silent Set Mode | A buzzer sounds cheap on video. The DFPlayer plays MP3 files from a microSD card, one line of code per sound. Set Mode stays silent and logs each swing with a timestamp for the sound editor, which covers "silent operation". |
| MPU6050 | Keep it (owned) | Swing, clash and raise detection, and free-fall checked in software 200 times a second |
| Vibration motor | Keep it (owned). Drive it with a kit transistor and diode. | Silent "go" cue to the actor, and error buzz patterns |
| (new) FSR pad (owned) | Under the grip wrap as a hand-on-grip (dead-man) sensor | The blade can only ignite when someone is holding the sword. Let go, especially while falling, and it shuts down. Strong actor-safety failsafe, costs nothing. |
| Crystal over Wi-Fi | ESP-NOW sword → crystal. The crystal also hosts the Wi-Fi dashboard. | Direct radio between the boards: a few ms of delay and no router needed |
| 0% infill PLA | 10 mm aluminium spine + PETG blade halves + TPU tip and pommel | Hollow PLA will crack in a 1.5 m drop, and the blade has to open to fit the LEDs |

**The one risk of keeping it simple: mechatronics marks.** Both rubrics list "Mechatronic considerations: rails, spools, actuators", and the brief mentions closed-loop position control. With a static crystal and a simple sword, the only actuator is the vibration motor. Strong areas: automation, sensing, wireless control, failsafes and toughness. Ask Will or Juan whether a haptic actuator plus the automated light/sound system is enough. If not, the cheapest fix that leaves both props as they are is a small display stand with the FS90R servo turning the crystal slowly in Demo Mode — an extra base, not a change to the crystal.

**Credit the model.** The sword is 3dworkbench's "HE-MAN Power Sword MOTU" on Printables, CC BY 4.0, so attribution is required. Put the credit in the process document and video end card. Use the model's 10 mm rod version (sword_with-rod-*.stl) as the base for the spine.

## 02 · Rubric coverage — brief → how we answer it

| Brief / rubric line | Our answer | Proof on the day |
|---|---|---|
| Main function automated by a microcontroller | ESP32-S2 in the sword runs gestures, blade, sound and haptics. ESP32 in the crystal runs sync, sensors and the dashboard. | Live demo + .ino files |
| Mechatronics: rails, spools, actuators | Vibration actuator, structural spine. The weakest area. Confirm with lecturers. | Haptic cue on command |
| Environmental analysis | Temp/humidity, light, knock/vibration in the crystal. Motion, set vibration and bay temperature in the sword. Each changes behaviour (section 08). | Sensor tiles + event log |
| Silent operation | Set Mode: no sound at all. Haptics only between lines. | Toggle Set/Demo on camera |
| Repeatable, 3+ takes | Take Reset, fixed cue sequences, take counter, per-take log | Three takes in a row + 10-take test sheet |
| IP22 | Gasketed seams, IP65 button, plugged charge port, speaker mesh | Drip test video |
| Drop 1.5 m and still works | Spine, PETG/TPU, cell held in a sleeve, foam-mounted board, fall shutdown, self-test after | Drop → self-test green → next take |
| Failsafe: actor safety, killswitch, error indicator | Grip dead-man, fall shutdown, master switch, dashboard and crystal kill, low-battery cut, status colours | Let go of the sword, it goes dark |
| Wireless, fits production workflow | Phone dashboard: ARM / ROLL / CUT, cues, take counter, CSV log | Director runs it from a phone |
| Camera-ready finish | Frosted PETG blade, painted and weathered hilt, flicker-tested LEDs | Final video |

## 03 · Architecture — how the two props talk

- **Sword**: ESP32-S2 (owned), 1× 18650. IN: MPU6050, FSR grip pad, ignite button, battery voltage. OUT: ~78× WS2812B blade, DFPlayer → 40 mm speaker, vibration motor, status pixel. Master switch, USB-C charge.
- **Crystal** (solid, static): ESP32 (owned), power bank (owned). IN: BME280 temp + humidity, BH1750 light level, SW-420 knock/vibration, kill button. OUT: WS2812B ring, e-ink status (optional). HOSTS: Wi-Fi AP "GRAYSKULL-SET", dashboard + event log.
- **Phone**: director / prop master. ARM · ROLL · CUT · RESET · KILL.
- Sword → crystal over ESP-NOW (~5 ms): swing, clash, raise, drop, heartbeat. Crystal → sword: cue, mode, lux, KILL. Crystal ↔ phone over WebSocket: status 5 Hz, commands.

The swing path has to feel instant: a swing reaches the crystal over ESP-NOW in a few ms. The phone runs at human speed over the crystal's own Wi-Fi, so campus Wi-Fi and Blynk aren't needed (a set has no reliable internet). If the crystal or phone drops out, the sword keeps working on its own.

ESP-NOW and Wi-Fi can run together on one ESP32 if both use the same channel. Pin the crystal's AP to channel 1 and set the sword's ESP-NOW peer to channel 1. ESP-NOW works between an ESP32-S2 and a classic ESP32.

## 04 · Behaviour — modes, gestures and the take cycle

States: OFF → (switch on) → STANDBY → (ARM; self-test OK + 3 s button hold) → ARMED → (ROLL) → ROLLING → (CUT → take+1) → ARMED. Any state → SAFE / FAULT on fall, KILL, grip released + fall, or battery < 3.4 V.

CUT always returns to ARMED and resets the effects, so every take starts the same way. ROLLING locks the blade brightness so it can't change mid-take. Any fault, from any state, goes to SAFE: blade dark, sound and buzz off.

| Trigger | Detection (starting values, tune them) | Sword does | Crystal does |
|---|---|---|---|
| Arm | Dashboard ARM. Sword held still 1.5 s to calibrate. | Self-test: blade sweeps R/G/B, one buzz, battery check | Ring sweep, reports READY |
| Ignite | Hold button 0.6 s and FSR shows a hand on the grip | Blade fills base → tip over 400 ms | Fades in to 30% |
| Swing | Gyro magnitude > 250 °/s (rising edge), 150 ms lockout | Brightness surge along the blade, whoosh | Pulse, brightness ∝ swing speed |
| Clash | Acceleration spike > 4 g | White flash, short buzz | Bright flicker |
| "By the power of Grayskull" | Pitch > 70° held 1.2 s | Lightning crawl, long buzz swell | Full lightning flare, then steady glow |
| Director cue | Dashboard button | Silent 2-pulse buzz: "go" | (nothing) |
| Fall | Total acceleration < 0.35 g for > 60 ms (sampled 200×/s) | SAFE immediately, logs DROP | Shows DROP alert |

**Set Mode vs Demo Mode.** In Set Mode the sword never sends a play command to the DFPlayer, so it stays silent. Every event is still logged with a timestamp and exported as CSV so an editor can add sounds in post. Demo Mode plays the sounds for the class showcase. Show both on camera.

## 05 · Hero prop — sword physical layout

Battery (18650 in printed sleeve + foam) sits in the grip so the weight stays in the hand. USB-C + plug and master switch in the pommel. FSR pad under the grip wrap where the palm presses. Electronics (ESP32-S2 · MPU6050 · DFPlayer) on one tray in the guard, the stiffest and roomiest part. Speaker + mesh in the guard. WS2812B strip on both faces, 60/m. 10 mm aluminium tube spine runs from the grip about two-thirds up the blade and carries the LED wires inside. TPU tip.

**Blade**
- Reprint the blade as front and back halves (Fusion: Split Body on the flat plane) in clear or white PETG, 2 walls, 15% gyroid. Thin walls let light through.
- Model a channel for the 10 mm tube. Print a thin PETG rib that carries one strip on each face, 10–15 mm behind the skin, so the glow is even instead of dotty.
- Sand inside and out for frosting. Paint the ridge lines silver so it reads as steel when off.
- Close the halves with M3 heat-set inserts and screws so you can get back in.

**Hilt**
- Reprint guard and grip in PETG, 4 walls, 25% gyroid. Pommel, tip and the guard's round knobs in TPU: they hit first in a drop.
- Measure the grip bore. An 18650 needs ≥ 20 mm inside diameter with a sleeve.
- All electronics on one printed tray that slides into the guard, held by two screws, with one plug to the blade and one to the grip. Mount on 2 mm foam tape.
- FSR pad: glue flat on the grip under the wrap (leather, cord or TPU sleeve). Don't fold it over a sharp edge.
- Current prints become the fit-test mock-up.

**Sword pin map (ESP32-S2, assumes Wemos/LOLIN S2 Mini, 34 × 26 mm)**

| Pin | Connects to | Notes |
|---|---|---|
| VBUS / 5V | MT3608 boost output, set to 5.0 V | Set voltage with a multimeter before connecting. Don't plug USB in while battery is on. Flash over Wi-Fi with ArduinoOTA. |
| GPIO1 | Battery voltage: 100k/100k divider + 100 nF | ADC1 (GPIO1–10 on the S2) |
| GPIO2 | FSR grip pad, with 10k to GND | ADC1. Higher reading = firmer grip. |
| GPIO4 / 5 | MPU6050 SDA / SCL | Short wires, both on the tray |
| GPIO7 | 330 Ω → status pixel → blade face A → face B | One chain. 1000 µF across LED power at the strip. |
| GPIO9 / 10 | DFPlayer RX (via 1k) / TX | Serial1. Speaker straight onto SPK1/SPK2. |
| GPIO12 | 1k → 2N2222 base. Motor between 3V3 and collector, 1N4148 across the motor. | PWM for buzz patterns. All kit parts. |
| GPIO13 | Ignite button → GND (INPUT_PULLUP) | |
| GPIO15 | Onboard LED (S2 Mini) | Debug only |

Check pins against the actual board's pinout and avoid strapping pins 0, 45 and 46. If it doesn't fit the guard, drop-in replacement: Waveshare ESP32-S3-Zero (23.5 × 18 mm, Communica R149) with the same pin numbers.

## 06 · Power — sword power and budget

USB-C 5 V (TPU plug) → TP4056 + DW01 (charge 1 A, short/flat cut) → 18650 1S (3.0–4.2 V, 3000 mAh) → master switch (≥ 3 A latching, in pommel) → splits to: ~78× WS2812B direct (cap 1.2 A); MT3608 boost → 5.0 V → ESP32-S2 5V and DFPlayer; ESP32 3V3 out → MPU, FSR, motor.

The LEDs run straight off the cell, the way commercial pixel-blade sabers do: no big boost converter and no level shifter (3.3 V data reads reliably at a 3.4–4.2 V LED supply). The small MT3608 holds the ESP32 and DFPlayer at a steady 5 V, so a bright swing can't brown out the board mid-take.

| Load (sword) | Typical mA | Peak mA | How we cap it |
|---|---|---|---|
| ESP32-S2 with ESP-NOW | 90 | 300 | |
| Blade, ~78× WS2812B | 700 | 1 200 | FastLED setMaxPowerInVoltsAndMilliamps(3.7, 1200). Uncapped full white ≈ 4 700 mA. |
| DFPlayer + speaker (Demo Mode, via boost) | 100 | 550 | Volume ceiling. 0 in Set Mode. |
| Vibration motor | 5 | 90 | Pulses only |
| MPU6050, FSR, divider | 5 | 5 | |
| Total from the cell | ≈ 900 | ≈ 2 150 | Under the protection board's ~3 A trip |

**Runtime.** 3000 mAh at 80% usable and 0.9 A ≈ 2 h 40 min blade-on, or 100+ one-minute takes.

**Cells.** Use cells from the RC catamaran pack for bench testing. For the final sword buy one new cell (Communica INR18650-30HW, R135). Check datasheet for ≥ 5 A continuous discharge.

**Measure before you trust the table.** In week 9, power 15 LEDs plus the ESP32 at 3.7 V with a multimeter in series. Record idle glow, brightest swing and full white at the cap, then scale by 78/15. Put measured numbers in the process document.

**Crystal power.** Netstar power bank (4000 mAh, 5 V 2 A) via USB cable into the base. ESP32 (~120 mA) + ring capped at 300 mA ≈ 0.42 A, roughly 6 hours. Some power banks switch off at very low load; ESP32 with Wi-Fi normally draws enough, but test it.

## 07 · Set piece — crystal: solid, static, sensing

All electronics are in the base. The LED ring faces up into the crystal, so the crystal stays a single solid print. The base has one small vent for the temperature/humidity sensor and a clear window for the light sensor. USB in.

- **Light.** Print the crystal in translucent PETG or clear PLA, no infill or light gyroid, sanded frosted. WS2812B ring under it facing up. If the ring is small (7–8 LEDs) and the crystal tall, add a short strip up a hollow core.
- **Sensors in the base.** BME280 behind a small louvred vent (needs air), BH1750 behind a clear window (needs light), SW-420 glued to the base floor, kill button on the back.
- **Brain.** ESP32 board: receives sword events, drives the ring, reads sensors, serves the dashboard from its own AP.
- **Optional e-ink status** on the back: take number, sword battery, temperature, mode. Doesn't glow on camera, keeps its image unpowered. Only if time allows.
- **Drop and IP22.** Solid base walls (4 walls), board on foam tape, vent louvres angled down, gasket where crystal meets base.

**Crystal pin map (ESP32 DevKit)**

| GPIO | Connects to | Notes |
|---|---|---|
| VIN / USB | Power bank via USB cable | Ring takes 5 V from the same input |
| GPIO13 | 330 Ω → WS2812B ring DIN | 1000 µF across ring power |
| GPIO21 / 22 | I²C SDA / SCL → BME280 (0x76), BH1750 (0x23) | No address clash |
| GPIO27 | SW-420 knock module DO | Interrupt on knock |
| GPIO32 | Kill button → GND (INPUT_PULLUP) | Broadcasts SAFE to the sword |
| GPIO18 / 23 / 5 / 19 / 15 / 35 | (optional) e-ink SCK / MOSI / CS / DC / RST / BUSY | GxEPD2 library |

First-draft swaps from the kit: DHT11 on GPIO4 instead of BME280, and an LDR on GPIO34 (10k divider) instead of BH1750. Upgrade both for the final.

## 08 · Environmental analysis — every sensor changes something

| Sensor | Measures | Behaviour change |
|---|---|---|
| BH1750 (crystal) | Set lighting, lux | Sets blade and crystal brightness at ARM so they expose consistently, then locks it during ROLLING |
| BME280 (crystal) | Temperature, humidity | Above 32 °C, blade brightness cap drops to 70%. Above 85% humidity: condensation warning (IP22 covers drips only). |
| SW-420 (crystal) | Knocks to the crystal or its table | Logs the knock and flashes the dashboard. During ROLLING, the take is flagged for continuity. |
| MPU6050 (sword, at rest) | Set vibration (dolly track, generator, footsteps) | Reports vibration level. When the set is shaky, the swing threshold rises. |
| MPU6050 temperature (sword) | Electronics tray temperature | Above 50 °C: brightness halves. Above 60 °C: SAFE. |
| FSR (sword) | Is a hand on the grip? | Allows ignite, powers down when put down |

Dust: a PMS5003 (R528, Micro Robotics) needs a fan and air path, which fights a sealed crystal. Mention in the write-up that it was considered and why it was left out. If a lecturer insists, it fits in the crystal base behind a vent.

## 09 · Safety — failsafes

| Hazard | Detection | Action | Indicator |
|---|---|---|---|
| Nobody holding the sword | FSR reading below threshold | Can't ignite. In STANDBY/ARMED, blade dims to idle after 2 s. Grip release + fall = instant SAFE. During ROLLING, release alone is only logged. | Status pixel amber |
| Prop dropped | MPU6050 free-fall (200×/s) | Blade, sound, buzz off before impact, then self-test after landing | Amber, then green on pass. DROP in log. |
| Short / overload / flat cell | TP4056 protection (DW01) + battery ADC | Hardware cut. Software SAFE below 3.4 V. | 3 red blinks + long buzz |
| Anything, by a person | Master switch, crystal kill button, dashboard KILL | Power off (switch), or SAFE broadcast to both props | Both props red |
| Left on and forgotten | 10 min no motion and no commands, outside ROLLING | Blade off, low-power standby | Slow breathe |
| Code hang / link lost | Watchdog, heartbeat 4×/s | Reboot into SAFE. Crystal goes to idle glow. | "LINK" tile grey |

Lithium handling: use the protection board, don't charge unattended, keep the cell in a sleeve, never pot or foam-fill around it. Swap a cell that is dented after a drop test.

## 10 · Hardening — IP22 and the drop test

IP22: first 2 = nothing ≥ 12.5 mm gets in (a finger). Second 2 = dripping water at 3 mm/min does no harm with the prop tilted up to 15°, 2.5 min in each of 4 positions (IEC 60529).

- 1.5 mm silicone O-ring cord in printed grooves on blade halves and guard
- IP65 anti-vandal ignite button
- USB-C port facing down in the pommel with a tethered TPU plug
- Speaker behind downward louvres and a fine mesh disc
- Primer and clear coat to seal print pinholes
- Wording: "IP22-ready, tested to the IPX2 drip procedure", not "IP22 certified".
- Test on video: spray bottle dripping at a 15° tilt, 2.5 min per side, then run a take. Push a 12.5 mm dowel at every opening.

Surviving 1.5 m:
- Spine carries the load. Epoxy it into the grip and cross-pin with an M3 bolt.
- Cell in a tight printed sleeve with a foam end-pad. Zip-tie the holder too.
- Tray on foam tape. No Dupont jumpers or loose header pins: solder silicone wire, hot-glue strain relief.
- LED strip on its rib with VHB tape, joints covered in hot glue.
- TPU knobs, pommel and tip land first.
- Low-expansion foam only in sealed decorative gaps (guard C-arms), never in the blade or near the cell.
- Drop in stages: empty shell with dummy weight, then tray with a dead cell, then the live cell. Six orientations from 1.5 m onto the floor you'll be marked on.

Camera flicker check: WS2812B dims by fast flicker, cameras can show banding. Film the blade at 25 fps, 1/50 shutter (SA 50 Hz mains) before locking effects. Low brightness bands worst, so dim with diffusion rather than very low values.

## 11 · Code — firmware and dashboard

Libraries: FastLED (blade and ring, power cap); Adafruit_MPU6050 or MPU6050_light; DFRobotDFPlayerMini; esp_now.h; ArduinoOTA; ESPAsyncWebServer (dashboard + WebSocket); Adafruit_BME280, BH1750; GxEPD2 if e-ink.

Rules: no delay(), one state machine per board, every effect timed with millis().

Dashboard (served by the crystal): slate (scene, take number, ARM / ROLL / CUT / RESET); mode switch SET (silent) / DEMO (sound); two or three fixed cue sequences; tiles for sword battery, grip, link, temperature, humidity, lux, knocks; big red KILL always visible; event log with CSV download. Reuse the IoT-term dashboard work, served locally instead of through Blynk.

```cpp
// Shared by both ESP32s: one small packet for every event
enum Evt : uint8_t { HEARTBEAT, IGNITE, SWING, CLASH, RAISE, DROP, FAULT,
                     CUE, MODE, ARM, ROLL, CUT, KILL, ACK };

struct __attribute__((packed)) Packet {
  uint8_t  evt;     // Evt
  uint8_t  take;    // take counter, so the log lines up with the slate
  uint16_t seq;     // spot lost or repeated packets
  uint32_t ms;      // sender millis() at the event
  float    value;   // swing deg/s, clash g, lux, battery V...
};

// Latency: the crystal sends back an ACK with the same seq.
// The sword logs (ack_time - send_time) / 2 as the one-way delay.
```

Test log (from week 11): Take, Action (ignite, raise, 3 swings, cut), Sword → crystal ms, Batt V start/end, Self-test, Faults, Pass — for takes 1 to 10. Also log false triggers per minute while walking with the sword.

## 12 · Parts — what you have, what to buy

Owned: ESP32-S2 (sword brain, check fit), ESP32 (crystal + dashboard), MPU6050, FSR pad (SmartPos), vibration motor, WS2812 strips (final needs 60 LEDs/m, ~1.6 m), WS2812B ring, e-ink module (optional), Netstar power bank, 18650 cells (bench only), kit buttons/LEDs/resistors/transistors/diodes/DHT11/LDR/SW-420.
Not needed: Arduino Pro Micro, ST7735, geared motor, L298N, LM2596, 9 V battery, Blynk. FS90R only if lecturers ask for a moving part.

**Buy: sword**

| Part | Job | Supplier | Price | Stock 2 Oct |
|---|---|---|---|---|
| 18650 3000 mAh (new) | Sword power | Communica INR18650-30HW | R135.00 | check discharge spec |
| 18650 single holder | Cell mount | Micro Robotics 18650-1C-BOX | R18.40 | Stellenbosch |
| TP4056 USB-C charger with protection | Charging, protection | DIYElectronics | R9–11 | in stock |
| MT3608 boost module | Steady 5 V | Next Dimension | R13.35 | in stock |
| DFPlayer Mini + microSD | Sound | Communica HKD MP3-TF-16P | R34.00 + ~R60 | in stock |
| 40 mm speaker 3 W 4 Ω | Sound | DIYElectronics | R46.00 | in stock |
| WS2812B 60/m strip, non-waterproof, 2 m | Final blade (skip if owned strip is 60/m and long enough) | Communica (R85/m) | R170.00 | in stock |
| 16 mm anti-vandal momentary, IP65 | Ignite button | Communica AVP16DW-M1S | R79.35 | in stock |
| Latching power switch ≥ 3 A DC | Master switch | Communica / Micro Robotics | ≈ R80 | estimate |
| 10 mm aluminium tube, 1 m | Spine + conduit | Hardware store | ≈ R90 | estimate |
| PETG 1 kg (clear or white) | Blade, guard, grip, crystal | Next Dimension FIL X | R318.00 | in stock |
| TPU 95A | Tip, pommel, knobs, plug | Ask the lab first (both shops out) | ≈ R400 | find stock |
| Silicone O-ring cord 1.5 mm, 2 m | Seam seals | Next Dimension | R31.62 | in stock |
| Silicone wire, JST-PH, heat-shrink | Wiring | Any | ≈ R150 | estimate |

Sword total ≈ R1 635 (electronics ≈ R645, materials ≈ R990).

**Buy: crystal**

| Part | Job | Supplier | Price |
|---|---|---|---|
| BME280 (5 V) | Temp / humidity (final) | Micro Robotics BME280-5V | R78.20 |
| BH1750 | Light level (final) | Next Dimension | R27.57 |
| SW-420 module (if not in kit) | Knock sensing | Communica | R17.00 |

Crystal total ≈ R123. Total to buy ≈ R1 760, of which about R1 000 is printing and build material. Communica ships free over R1 000. Next Dimension is in Strand (collect). Micro Robotics has a Stellenbosch branch. "DIYRobotics" was priced as DIYElectronics (diyelectronics.co.za).

## 13 · Schedule — week by week

Dates estimated from the Canvas calendar. The brief says final hand-in is Week 15 in one place and Week 16 in another — confirm with Will or Juan.

| Week | Build | Code | Evidence |
|---|---|---|---|
| 9 · ~13 Oct | Order parts. Measure the sword. Fusion: split blade, spine channel, tray, cell sleeve. | ESP-NOW between S2 and ESP32. Log MPU6050 data from real swings. Read the FSR. | Swing data plot, measured current |
| 10 · ~20 Oct | Print PETG blade, guard, crystal base | Blade effects with power cap. DFPlayer sounds. Crystal ring sync. | Sync video on the bench |
| 11 · ~27 Oct | Wire the sword tray. First fit. Dummy-weight drop. | Dashboard v1, crystal sensors (DHT11/LDR fine for now) | First drip test, latency numbers |
| 12 · ~3 Nov | FIRST DRAFT HAND-IN: ignite, swing → crystal, raise effect, Set/Demo, grip and fall failsafes working, unrefined. | | Stress-test feedback |
| 13 · ~10 Nov | Fix feedback. Seals, O-rings. Swap in BME280 and BH1750. | All failsafes, cue sequences, CSV log | 10-take sheet |
| 14 · ~17 Nov | Sand, prime, paint, weather. TPU parts. | Tune thresholds, lux lock | Six-orientation drop video |
| 15 · ~24 Nov | Shoot and edit the video. Spares kit. | Freeze the firmware | Final .ino + video |
| 16 · ~1 Dec | FINAL DEMONSTRATION (confirm Week 15 or 16) | | Live filming session |

Spares kit for the day: charged spare cell, charged power bank, a spare flashed board, 30 cm of LED strip, hot glue gun, USB-C cable, multimeter, screwdriver, printed take sheet.

## 14 · Demonstration video — shot list

1. Cold open in a dim room: "By the power of Grayskull", the blade lights the actor's face and the crystal flares.
2. Split screen: swing close-up next to the crystal pulse, to show the sync.
3. Phone over the shoulder: the director runs ARM → ROLL → cue → CUT.
4. Three takes back to back with the take counter in frame.
5. Set Mode vs Demo Mode with a sound-meter app on screen.
6. Environment: switch a lamp on before ARM and the blade brightness adapts. Knock the table and the crystal flags it.
7. Failsafes: let go of the grip, the KILL button, a fall with the self-test passing.
8. IP22 drip test and the 1.5 m drop, then the prop still works.
9. End card: team, parts, credit to 3dworkbench (CC BY 4.0).

## 15 · Before you buy — open questions

- Mechatronics: ask Will or Juan whether a haptic actuator plus automated light and sound meets "rails, spools, actuators". This is the one decision that could change the plan.
- Which ESP32-S2 board (S2 Mini?) and whether it fits in the guard.
- The WS2812 strip: LEDs per metre and length. If 60/m and ≥ 1.6 m, skip buying the strip.
- The ring: number of LEDs and diameter, against the crystal's base size.
- Sword measurements: blade length and grip inside diameter (≥ 20 mm for the cell).
- Final hand-in: Week 15 or 16, and which prop gets drop-tested.
