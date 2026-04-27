# Project Context for Claude Code

This file exists so that a Claude Code session starting fresh on Windows can
pick up exactly where the Linux session left off. Read it first.

## What this repo is

Course: **E10 Robot Project, Spring 2026** (SJSU-style intro engineering
robotics lab). The assignment is an autonomous VEX Cortex robot that
performs a "search & rescue" task in an arena with two IR beacons.

Hardware platform: **VEX Cortex** controller, programmed in **RobotC** on
Windows. Robot chassis is a SquareBot (or team variant). The team has also
built a custom **Infrared Receiver Board (IRB)** with 8 forward-facing
photodiodes, a CD4051 8:1 mux, CD4052 dual 4:1 mux, MC33204P rail-to-rail
op-amp, and a CD4520 counter.

## The four graded tasks

1. **Find the red beacon** (1 kHz IR flash) and stop next to it — 25 pts
   (15 pts partial credit if it locates but doesn't stop)
2. **Press the pushbutton** on top of the red beacon (arm deployment) —
   20 pts (15 pts if arm operates but doesn't turn off the beacon, 10 pts
   if the arm fails to operate at all)
3. **Find the green beacon** (10 kHz IR flash) — 10 pts (5 pts if it
   wanders without finding it)
4. **Push/carry the green beacon** across the starting line out of the
   arena — 10 pts

Plus 10 pts robot weight (≤2.0 kg ideal), 5 pts creativity, 5 pts time
bonus (fastest), 5 pts lightest bonus. **Three runs allowed, best one
scored.** Report + presentation = another 80 pts separately.

## What is already in this repo

- `Beacon-Testing.c` — a tiny diagnostic program that prints the analog
  IR value to the debug stream every 100 ms. Useful for confirming the
  IRB's analog output reaches the Cortex on `analog 1` independent of
  the digital control signals. (Note: its pragma block uses a different
  port assignment than `robot_main.c`; treat it as a standalone bench
  test, not a wiring reference.)

- `robot_main.c` — the full submission program (added on branch
  `hrittik-testing`). State machine:
    - `phase_redBeacon()`  → Tasks 1 + 2
    - `phase_greenBeacon()` → Tasks 3 + 4
  Built on top of the GOTO_BEACON IRB routines. Adds:
    - arm servo control with limit switch + timeout fallback
    - bumper wall-recovery
    - ultrasonic close-range stop (more reliable than IR intensity alone)
    - dual-frequency confirmation per guidelines §7.4 (rejects
      green-bleed-through when tuned to red and vice versa)
    - status display: LCD top line shows the current phase, bottom line
      shows live IR telemetry — `f<freq> s=<sum> D<detector>` when IR is
      visible, **`NO IR sum=N`** when below ambient. Same info goes to
      the debug stream when tethered (with `IR ACQUIRED` / `IR LOST`
      transition events).

## Pragma / wiring assumptions in `robot_main.c`

These were confirmed against the physical robot by the team on 2026-04-27:
- `analog 1` = IR intensity output from IRB (3-wire stick connector)
- `port 1` = arm motor (continuous-rotation servo). Sign convention:
  negative PWM drops the arm, positive raises it — flip both signs in
  `pressBeaconButton()` if reversed.
- `port 9` = left motor (reversed)
- `port 10` = right motor (reversed)
- `digital 1` = ultrasonic sensor input/echo, declared `sensorSONAR_cm`.
  Pragma also implicitly reserves `digital 2` as the output/ping line.
  If sonar reads 0 always, swap the two wires (RobotC's port convention
  may be the reverse of what the team wired).
- `digital 5` = bumper switch
- `digital 8 / 9 / 10` = IRB rainbow ribbon (freqSelect / exposeCtrl /
  counterReset). **NOTE**: the published guidelines diagram (§7.5) puts
  the IRB ribbon on digital 9-12, but this team's wiring starts at
  digital 8. If the IRB doesn't respond after a download, this is the
  first thing to recheck — see "If the IRB looks dead" below.

These are still GUESSES (kept at the original pragma slot so the code
compiles):
- `digital 6` = arm limit switch (trips when arm is fully down). If you
  don't have a limit switch wired, the timeout `arm_drop_ms` (1200 ms)
  is the only stop — that's a safe fallback.
- Optical encoders: REMOVED from the pragma block (the code never used
  them and `digital 1` is now needed for sonar).

If any of these don't match, edit the `#pragma config` lines at the top
of `robot_main.c`. Don't refactor the logic — just fix the port names.

## If the IRB looks dead

Symptoms: `PD_sum` stays near zero on the LCD bottom line ("NO IR sum=N")
even when a beacon is clearly pointed at the bot, or the bot only spins
(search mode) and never acquires a target.

Most likely causes, in order:

1. **Wrong digital port for the IRB ribbon.** Our code assumes
   freqSelect=dgtl8, exposeCtrl=dgtl9, counterReset=dgtl10. If the ribbon
   is actually on dgtl9-11 (one port off) or dgtl10-12 (the published
   guidelines layout), the IRB control signals never reach the right
   pins. To rule this out: load `Beacon-Testing.c` and watch its
   `beacon` value with a beacon pointed at the IRB. If that value
   responds, the IRB analog side is fine and the problem is purely in
   the digital control lines — try shifting the three pragma ports up
   by 1 or 2 and recompile.
2. **Frequency-select line wrong** — IRB stuck at one frequency. Try
   the other beacon. If only red works, freqSelect isn't toggling.
3. **Counter-reset line wrong** — detector index never resets, PD0..PD7
   read garbage. Look at IRB's LED1-LED3 (Table 5 in the guidelines):
   in normal operation they cycle through the 8-detector pattern; if
   frozen, the counter-reset line is wrong.
4. **Exposure-control line wrong** — the shutter never opens, every PD
   reads near-zero. PD_sum stays well below `ambient_level=200`.

The IRB ribbon places three control signals on three consecutive digital
ports. Whatever offset the team used, the fix is one line per signal at
the top of `robot_main.c`.

## Tuning parameters (top of `robot_main.c`)

All in globals, grouped together for easy adjustment. Defaults are
intentionally conservative because the team did not have time to calibrate
before the run:

- `expose_time = 5` (ms, 3–8 valid — higher = more IR sensitivity/range)
- `ambient_level = 200` (PD_sum below this → enter search/spin mode)
- `slow_level = 5000` (PD_sum above this → drop to slow speed)
- `stop_level = 6000` (PD_sum above this → consider beacon reached)
- `near_beacon_cm = 15` (sonar stop distance — independent of IR)
- `forward_speed = 35`, `slow_speed = 25`, `spin_speed = 50` (fast spin
  for backup/recovery turns), `search_spin_speed = 22` (slow spin during
  IR-search mode in `goToBeaconStep()` so the 8 detectors get enough
  dwell time to catch a faint beacon as we sweep past it)
- `steer_sensitivity = 20` (heading-error gain, PD4 = straight ahead)
- `arm_drop_ms = 1200`, `arm_raise_ms = 1000` (servo timing fallbacks)
- `backup_ms = 700`, `turn_180_ms = 1400`, `push_out_ms = 4000`

Two independent stop conditions run in parallel in `goToBeaconStep()`:
IR saturation (`PD_sum > stop_level`) OR sonar close (`sonar < 15 cm`).
Whichever trips first wins. This was deliberate — if the IR thresholds
are off, the sonar still saves us from ramming the beacon.

## Known gotchas from the guidelines

- **§7.4 frequency bleed-through**: when the green beacon is within ~1–2
  ft, the 1 kHz-tuned circuit will still see it.
  `confirmBeaconFrequency()` addresses this — it samples both frequencies
  and only accepts a reading if the target frequency is stronger.
- **The IRB ribbon takes consecutive digital ports.** The guidelines
  diagram shows ports 9-12, but our team's wiring starts at port 8 (see
  pragma section above). The three control signals MUST be on
  consecutive digital outputs in the order the code expects.
- **Reset-to-detector-0 must happen before each 8-detector sweep** —
  `ReadPD()` does this via the counter reset line. Don't remove it.
- The red beacon's pushbutton must be physically depressed by the arm;
  no electrical shortcut exists. The robot must approach close enough
  that a reasonable arm length can reach over the top of the beacon.
- **Three runs allowed, best one scored** (per guidelines §3 grading).
  After a failed run, don't rewrite the code — first re-test what
  changed between runs (battery sag, beacon position, ambient IR from
  windows).

## Branch / workflow state

- Work branch: `hrittik-testing` on `demonicdeception/robotc-project-code`
  (the team fork).
- A PR from the fork into `acrossthestreet:hrittik-testing` may need to
  be opened manually (the original Linux PAT lacked `pull_request:write`).
- If the Windows session has a better-scoped token or is the repo owner,
  push directly to `origin/hrittik-testing` and skip the fork dance.

## Immediate next steps

(Pragmas were updated 2026-04-27 to match the physical robot. Status
display was added the same day so "NO IR" is visible on the LCD during
a run.)

1. Open `robot_main.c` in RobotC on Windows and compile.
2. Plug in the Cortex via USB. Open **Robot → Debugger → Sensors** and
   **View → Debug Stream**.
3. Verify each sensor:
   - Press the bumper by hand → `bumper` toggles 1↔0 in the debugger.
   - Trigger the arm limit switch (if wired) → `armLimit` toggles. If
     none, the timeout fallback covers it.
   - Wave a hand in front of the sonar → `sonarSensor` value changes in
     cm. If it stays at 0, swap the two sonar wires (dgtl1 ↔ dgtl2).
   - Power up a beacon ~3 ft away pointing at the bot → `PD_sum` rises
     above `ambient_level=200`, LCD bottom switches `NO IR sum=N` →
     `f0 s=N D…`, and the debug stream prints `IR ACQUIRED`.
4. If LCD bottom stays `NO IR` with a beacon pointed at it, the IRB
   rainbow ribbon is on the wrong digital ports — see "If the IRB looks
   dead" above.
5. Sanity-check the arm direction in `pressBeaconButton()`: the code
   uses `motor[armMotor] = -60` to drop, `+60` to raise. If reversed on
   the physical robot, flip both signs.
6. Optional pre-run calibration: point the bot at the red beacon from 1
   ft and watch the debug stream. If `PD_sum` saturates above 6000 from
   >2 ft, raise `stop_level`; if it never reaches 6000 even at
   point-blank, lower it. Sonar stop at 15 cm is the backstop either
   way.
7. Download to the Cortex and run.

## What NOT to do

- Do not change the IRB read sequence in `ReadPD()` / `Expose_and_read()`.
  The 5 ms settle, expose_time open window, and reset pulse ordering are
  hardware-correct per the course staff.
- Do not introduce multi-threaded tasks — RobotC cooperative scheduling
  is not needed here and will complicate debugging the single state
  machine.
- Do not rely on encoders for distance control — the guidelines explicitly
  say an optical encoder is not necessary, and the team didn't calibrate
  wheel circumference.
- Do not add a "return to start" phase — the task ends when the green
  beacon crosses the starting line. Extra motion risks DQ.
