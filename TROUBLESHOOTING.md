# Troubleshooting: SpeedyBee F405 WING Used as a Quad FC

This guide documents real issues encountered (and fixed) when running a **SpeedyBee F405 WING** flight controller — a board originally designed for fixed-wing planes — on a 4-motor QuadX build. If you're repurposing this FC (or a similar wing/plane-oriented FC) for a quad, start here.

---

## Background

The F405 WING ships with a Betaflight target that only defines **2 motor outputs** (for a twin-motor plane) plus **4 servo outputs** (for control surfaces). A quad needs 4 motor outputs, so the servo pads have to be repurposed as motors via the CLI `resource` command.

Default target layout:
| Function | Pin |
|---|---|
| MOTOR 1 | B07 (TIM4 CH2) |
| MOTOR 2 | B06 (TIM4 CH1) |
| SERVO 1 | B00 (TIM3 CH3) |
| SERVO 2 | B01 (TIM3 CH4) |
| SERVO 3 | C08 (TIM8 CH3) |
| SERVO 4 | C09 (TIM8 CH4) |

---

## 1. Betaflight only finds 2 motors in QuadX mode

**Symptom:** Motors tab shows only M1/M2 responding; M3/M4 don't exist.

**Cause:** The default target has no MOTOR3/MOTOR4 resource defined at all.

**Fix (CLI):**
```
resource MOTOR 1 B07
resource MOTOR 2 B06
resource MOTOR 3 C08
resource MOTOR 4 C09
resource SERVO 1 B00
resource SERVO 2 B01
resource SERVO 3 NONE
resource SERVO 4 NONE
save
```
This frees the SERVO3/4 pads (C08/C09) and reassigns them as MOTOR3/4, giving you 4 clean motor outputs while keeping SERVO1/2 available for a gimbal or other aux output if needed.

---

## 2. Two motors don't spin when armed, but work fine as standalone servos/PWM

**Symptom:** After remapping, MOTOR3/4 (C08/C09, on TIM8) work fine, but MOTOR1/2 (B06/B07, on TIM4) don't respond at all.

**Cause:** TIM4 is on the APB1 bus, which runs at a lower clock than APB2 (where TIM8 lives). At **DSHOT600**, TIM4's output can be unreliable or dead on some F4 boards, while TIM8 handles it fine.

**Fix:** Drop the motor protocol one step:
```
set motor_pwm_protocol = DSHOT300
```
(Configuration tab → ESC/Motor Protocol also works.) DSHOT300 is well within TIM4's reliable range on the F405.

---

## 3. Confirm your actual physical wiring before trusting any resource map

If your ESC signal wires aren't soldered to the pads you *think* they are (easy to mix up S1–S11 on this board), Betaflight will "see" motors that don't correspond to where your props actually are. Always visually confirm each ESC signal wire against the board's silkscreen/pinout diagram before assuming a resource assignment is correct — don't rely on the target defaults alone.

Useful CLI commands for auditing your setup:
```
resource        # full pin assignment list
timer           # which timer/channel each pin uses
dma show        # confirms no DMA stream conflicts between outputs
```

---

## 4. ESC "brake on stop" — motors grab/lock instead of coasting

**Symptom:** Motors hard-stop the instant you cut throttle, instead of spinning down freely.

**Fix (in BLHeliSuite / Bluejay configurator, per ESC):**
- **Brake On Stop → Off**

Other settings worth setting for a mid-size (6-7") build while you're in there:
- **PWM Frequency:** 24kHz (better torque/heat for bigger motors; 48/96kHz suits smaller/higher-kv builds)
- **Motor Timing:** Medium–MediumHigh
- **Low RPM Power Protect:** On
- Always **Read Setup** and repeat on all 4 ESCs individually — settings are per-ESC, not global.

---

## 5. Violent full-range motor oscillation on one axis when armed

**Symptom:** All motors seem fine spun individually in the Motors tab, but the moment you arm, one axis (e.g. roll) violently oscillates — motors slamming between idle and full power, props threatening to shake screws loose — while other axes stay calm. Happens even with props off, on the bench.

**How to diagnose:** Pull a blackbox log (see below) from an armed bench test and look at `motor[]` and `gyroADC[]` over time.
- If one axis's gyro trace is swinging with large amplitude while others are flat, **and** motor pairs are alternating fully between min/max in sync with it — that's a real, sustained, single-axis oscillation, not just "vibration."
- Rule out mechanical resonance first: if it happens identically with props off, it's electrical/control-loop related, not aerodynamic or bearing/prop-balance related.

**Common root causes, in order of likelihood:**
1. **Noisy/unfiltered RC input** — if the transmitter/receiver link's input isn't smoothed, the FC can end up chasing raw jitter in the stick data as if it were real setpoint change, causing exactly this kind of full-amplitude, fast oscillation. **This was the actual root cause in this build** — fixed via the RC smoothing/filter settings on the PID tab.
2. **PID gains too aggressive** for the motor/prop/timer combination — try Betaflight's "reduce" filter/PID defaults or manually lower P/D ~15-20% as a test.
3. **Motor mixer order vs. physical position mismatch** — especially likely after manually remapping `resource MOTOR` pins (see sections 1-3). Confirm each motor number lands in the correct physical corner for QuadX via the Motors tab diagram.
4. **Board/gyro orientation mismatch** — check Configuration tab → Board and Sensor Alignment matches how the FC is physically mounted.

**Fix that worked here:** RC input smoothing/filter was not aggressive enough for a noisy link, so raw jittery stick data was being interpreted directly as setpoint changes. Tightened the RC smoothing filter settings in the PID tab — oscillation went away.

---

## 6. Reading blackbox logs without removing the SD card

Reboot into USB Mass Storage mode instead of pulling the SD card:
- **Configurator:** Blackbox tab → "Reboot into Mass Storage (MSC) Mode"
- **CLI:** type `msc`

The FC will mount as a normal USB drive — copy the `.BBL`/`.BFL` files off, then power-cycle to return to normal FC mode.

To decode logs on Linux/CLI (useful for scripted analysis, not just Blackbox Explorer):
```bash
git clone https://github.com/betaflight/blackbox-tools.git
cd blackbox-tools && make
./obj/blackbox_decode --stdout your_log.BFL > decoded.csv
```
`decoded.csv` gives you per-loop `motor[]`, `gyroADC[]`, `rcCommand[]`, `setpoint[]`, etc. — from there you can plot or script analysis of oscillation frequency, amplitude, and which axis/motors are involved.

**Note:** if the decode tool reports a large percentage of "missing/failed" loop iterations, that indicates the SD card can't keep up with the write rate (common with slow/low-quality cards). It doesn't cause flight instability by itself, but it does leave gaps in your diagnostic data — use a faster-rated SD card if you see this.

---

## Summary checklist for repurposing this FC as a quad

- [ ] Remap MOTOR3/4 onto former SERVO3/4 pads via `resource`
- [ ] Set `motor_pwm_protocol` to DSHOT300 (not 600) if using the TIM4-based outputs (MOTOR1/2)
- [ ] Physically verify ESC wiring against the actual pinout diagram, not just target defaults
- [ ] Disable "Brake On Stop" on all 4 ESCs
- [ ] Confirm Motors tab spin test matches expected QuadX motor order/corners
- [ ] Check Board and Sensor Alignment matches physical FC mounting
- [ ] Tighten RC input smoothing if you see fast, full-range, single-axis oscillation on arm
- [ ] Pull blackbox logs via MSC mode + `blackbox-tools` to confirm root cause before changing PID gains blindly
