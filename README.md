# Focuser

Motorised focuser for the 150/750 reflector (rack-and-pinion, 1.25"): Arduino Nano + DRV8825
stepper driver, belt drive to the focus knob. Speaks the **Moonlite** protocol, so it works with the
INDI/Ekos **MoonLite** driver, and remembers its position across power cycles.

## Hardware

| Signal | Nano pin |
|---|---|
| STEP / DIR | A0 / A1 |
| SLP (driver sleep) | D5 |
| M0 / M1 / M2 (microstepping, 1/32) | D11 / D10 / D9 |
| In / out buttons (to GND) | D3 / D2 |

The driver needs its own 12 V supply; USB only powers the Nano.

Mechanics (measured 2026-10-18):

| | |
|---|---|
| Belt | 16T motor : 36T knob (2.25 : 1) |
| Microsteps per knob turn | 14400 |
| Drawtube travel per knob turn | 23.9 mm |
| One microstep | 1.66 µm |
| Microsteps per mm | 601 |

## Flashing

The board is a clone Nano (CH340 USB chip, `1a86:7523`) with the **old bootloader**:

```
arduino-cli lib install AccelStepper
arduino-cli compile --fqbn arduino:avr:nano:cpu=atmega328old -u -p /dev/ttyUSB0 .
```

- Keep only one `.ino` in this folder: arduino-cli compiles every `.ino` in a sketch folder together.
- Disconnect the focuser in Ekos first, it holds the serial port.
- On Ubuntu, if the Nano shows in `lsusb` but no `/dev/ttyUSB0` appears, the braille-display service
  grabbed it: `sudo apt remove brltty`, then replug.

## Behaviour

- **Position** is 0–65535, larger = drawtube out. An empty EEPROM starts at 30000 (~50 mm of room
  each way). The position is saved every time the motor stops, in a wear-levelled EEPROM ring.
- **Buttons** jog while held, slowly for fine manual focus. Jogs stop at 0 and 65535.
- **Sleep**: the driver is switched off 5 s after the last move (no holding torque). `:SK01#` keeps it
  powered permanently (saved in EEPROM), `:SK00#` restores sleep.
  **Keep it awake on the telescope** (`:SK01#`, set on this unit since 2026-10-25). With the driver
  asleep, the camera's weight back-drives the drawtube when the tube points up: during autofocus,
  which pauses for an exposure after every move, the best focus wandered by ~700 steps
  (13000 → 12300) in one evening. With the driver awake the position holds (see below). The motor
  runs warm, not hot.
- **Speeds** (Ekos "Focus Speed" → microsteps/s):

  | Ekos speed | Moonlite code | µsteps/s | Drawtube |
  |---|---|---|---|
  | 1 | 02 | 800 | 1.3 mm/s |
  | 2 (default) | 04 | 400 | 0.67 mm/s |
  | 3 | 08 | 200 | 0.33 mm/s |
  | 4 | 10 | 100 | 0.17 mm/s |
  | 5 | 20 | 50 | 0.08 mm/s |

  Button jogs use 50 µsteps/s (low, default) or 200 (high), set with `l` / `h`.

## Serial protocol

9600 baud. Opening the port resets the Nano; wait ~2 s before sending.

| Command | Reply | Meaning |
|---|---|---|
| `:GP#` | `XXXX#` | current position (hex) |
| `:GN#` | `XXXX#` | target position |
| `:SPxxxx#` | | set current position (sync) |
| `:SNxxxx#` | | set target position |
| `:FG#` | | go to target |
| `:FQ#` | | stop (decelerates) |
| `:GI#` | `01#` / `00#` | moving / stopped |
| `:GV#` | `10#` | firmware version |
| `:GD#` / `:SDxx#` | `XX#` | speed code (table above) |
| `:GH#` | `00#` | half step (not supported) |
| `:GT#` | `0000#` | temperature (no sensor) |
| `:GC#` | `00#` | temperature coefficient |
| `:GK#` / `:SKxx#` | `01#` / `00#` | keep driver awake (extension) |
| `:SH#` `:SF#` `:SCxx#` `:POxx#` `:C#` `:+#` `:-#` | | accepted, ignored |

Legacy single characters, outside a `:...#` frame: `i` / `o` jog in / out until `s` (stop),
`h` / `l` high / low jog speed.

## Ekos setup

In the Ekos profile choose the **MoonLite** focuser, port `/dev/ttyUSB0` (or the
`/dev/serial/by-id/usb-1a86_USB2.0-Serial-...` link), 9600 baud. After connecting, the INDI Control
Panel → MoonLite → Main Control shows the position.

## Zero point and session start

**Once:** rack the focuser fully in (in button or by hand on the knob, never a long Ekos move into
the hard stop), then INDI Control Panel → MoonLite → Main Control → **Sync** = 0 → Set. The firmware
never goes below 0, so the inner end is now also a soft limit for Ekos and the in button.

Then move out to just short of the outer end, note the position and enter a slightly lower value as
**Max Position** on the same tab, so Ekos (including autofocus) stays inside the travel. Note the
in-focus position of each camera too: next time go there directly and fine-tune with autofocus.

**Each session:** just connect. The position is saved every time the motor stops, so it stays valid
across power-offs.

**Re-zero** (rack fully in, Sync = 0) only if the knob was turned by hand while the motor was asleep,
the motor stalled against an end, or steps might otherwise have been lost.

Autofocus is still needed each session: focus shifts with temperature and with the camera used.

## Measured on the sky (2026-10-25)

150/750 + ASI290MC at prime focus, ASI120MM Mini guide camera, HEQ5, north balcony, 98% Moon,
targets 24–32° up, seeing 7–9″. Position numbers are this unit's counter with its zero point as of
that night.

| | |
|---|---|
| Best focus, ASI290MC, approached **outward** | **12600** |
| Same focus, approached **inward** (how Ekos Linear 1 Pass finishes) | **12413** (two autofocus runs 40 min apart: 12414, 12413) |
| **Backlash** on reversal | **~200 steps (0.33 mm)** |
| Image shift on reversal | ~90 px (~70″) on the ASI290MC: the drawtube rocks when it changes direction |
| Repeatability, same direction | identical star size on two returns to 12600 from below |
| Holding (driver awake) | no change after 45 s idle |
| Focus range with visible stars (6 s, gain 300, moonlit) | ~12100–12900; rings with a dark centre outside that |
| Focus drift | none measurable over 40 min after the first hour outside |

How the numbers were found: frames every 100 steps from 11300 to 13700 always moving outward, then
from 13100 to 12100 moving inward; star size measured in each frame. Both sweeps show the same
curve, shifted by 200 steps. **A focus position is only meaningful together with the direction it
was approached from.**

What went wrong before these settings (so it is recognisable if it happens again):

- **Step size 20**: the 11-point sweep covered only 220 steps, the curve was flat noise, and the fit
  put the "best" position outside the measured range (e.g. 13514 after measuring 12904–13224).
- **AF Overscan 100**, less than the backlash: the final move did not take up the slack, so Ekos
  stopped somewhere other than where it measured.
- **Driver asleep between moves**: the drawtube slid under the camera's weight (see Sleep above).

## Autofocus settings (150/750 + ASI290MC, Ekos)

Step size, overscan, exposure, gain and CFZ step were used on 2026-10-25 (two runs agreeing to
1 step). R² limit 0.6, Average Over 2 and the start scan off are suggestions not yet tried.

| Tab | Setting | Value | Why |
|---|---|---|---|
| Mechanics | Initial Step Size | **80** | stars are visible over ~800 steps; 11 × 80 spans the V |
| Mechanics | AF Overscan | **400** | twice the 200-step backlash |
| Mechanics | Number Steps | 11 | |
| Process | Algorithm | Linear 1 Pass, HFR, hyperbola | HFR copes with donut-shaped stars |
| Process | R² limit | **0.6** | at 7–9″ seeing fits land at 0.7–0.74 and 0.8 rejects good results |
| Process | Average Over | **2** | less seeing noise per point |
| Main | Exposure / gain | **5 s / 200** | enough stars without the noise of gain 500+ |
| CFZ | Step Size | **1.66 µm** | real drawtube travel per microstep |
| — | Start scan ("Scan for Start Position") | **off** once focus is known | the scan costs ~3 min of back-and-forth |

- Start autofocus near the known focus (12400–12600): Linear 1 Pass only searches around where it
  starts.
- A run takes ~5 min with the start scan, ~2 min without.
- The critical focus zone at f/5 is ~67 µm (4.88 · λ · N², λ = 0.55 µm), about 40 steps; an 80-step
  sweep point is 2 zones wide.
- **With the 2x Barlow:** keep the same step size. The focus zone at f/10 is 4x larger, but each
  focuser step also moves the Barlowed focal plane about 4x more, so the tolerance in focuser
  steps stays about 40.

A good result is a V-shaped or parabolic curve with points on both sides of a clear minimum. Only
one arm visible means focus is outside the sweep: move 300–400 steps that way and run again. A
"POOR result" with the solution outside the measured positions means the step is too small.

### Measuring backlash again

Two sweeps on a star field, as above: outward in 100-step steps through focus, then inward over the
same range. The difference between the two best positions is the backlash. Quicker, by eye:

1. Move in by 500 steps, so the slack is taken up in the inward direction.
2. Watch a star at high magnification in live view (or a tape mark on the drawtube).
3. Move out in steps of 10 and count the steps until the star (or mark) starts to move.
   That count is the backlash; set AF Overscan to about 2x it.

## Testing after changes

1. In Ekos, set 33000 from 30000: the drawtube must move **out** about 5 mm, then back with 30000.
2. Go 30000 → 35000 → 30000 a few times: a tape mark on the drawtube should line up each time.
3. Move to 31234, disconnect, unplug USB, reconnect: the position should still be 31234.
4. Hold each button ~10 s: about 1 mm of slow movement, stopping on release.
