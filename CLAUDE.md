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
2. **Press the pushbutton** on top of the red beacon (arm deployment) — 20 pts
3. **Find the green beacon** (10 kHz IR flash) — 10 pts
4. **Push/carry the green beacon** across the starting line out of the
   arena — 10 pts

Plus 10 pts for weight (<2.0 kg ideal), 5 pts creativity, 5 pts time bonus,
5 pts lightest bonus. Report + presentation = another 80 pts separately.

## What is already in this repo

- `gotobeacon.c` — the lab-provided sample program. Demonstrates the IRB
  read sequence (`ReadPD`, `Expose_and_read`, `Find_max`) and a basic
  proportional go-to-beacon move. DO NOT rewrite this file's IRB timing —
  the delays and exposure sequence are hardware-correct and were tuned by
  the course staff.

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

## Pragma / wiring assumptions in `robot_main.c`

These MATCH the lab sample (confirmed from the guidelines doc):
- `analog 1` = IR intensity output from IRB
- `digital 10` = frequency select (0 = 1 kHz red, 1 = 10 kHz green)
- `digital 11` = exposure control (0 = shutter open, 1 = close + advance)
- `digital 12` = counter reset to detector #0
- `port 1` = right motor (reversed), `port 10` = left motor (reversed)

These are GUESSES — verify against the actual robot before burning time
debugging:
- `digital 5` = front bumper switch
- `digital 6` = arm limit switch (trips when arm is fully down)
- `digital 7` = ultrasonic sensor (SONAR_cm)
- `port 2` = arm motor (continuous-rotation servo). Sign convention in
  code: negative PWM drops the arm, positive raises it — flip if reversed.
- `digital 1/3` = optical encoders (declared but unused; safe to delete)

If any of the GUESSED assignments don't match, the fix is to edit the
`#pragma config(Sensor, ...)` lines at the top of `robot_main.c`. Don't
refactor the logic — just correct the port names.

## Tuning parameters (top of `robot_main.c`)

All in globals, grouped together for easy adjustment. Defaults are
intentionally conservative because the team did not have time to calibrate
before the run:

- `expose_time = 5` (ms, 3–8 valid — higher = more IR sensitivity/range)
- `ambient_level = 200` (PD_sum below this → enter search/spin mode)
- `slow_level = 5000` (PD_sum above this → drop to slow speed)
- `stop_level = 6000` (PD_sum above this → consider beacon reached)
- `near_beacon_cm = 15` (sonar stop distance — independent of IR)
- `forward_speed = 35`, `slow_speed = 25`, `spin_speed = 50`
- `steer_sensitivity = 20` (heading-error gain, PD4 = straight ahead)
- `arm_drop_ms = 1200`, `arm_raise_ms = 1000` (servo timing fallbacks)
- `backup_ms = 700`, `turn_180_ms = 1400`, `push_out_ms = 4000`

Two independent stop conditions run in parallel in `goToBeaconStep()`:
IR saturation (`PD_sum > stop_level`) OR sonar close (`sonar < 15 cm`).
Whichever trips first wins. This was deliberate — if the IR thresholds
are off, the sonar still saves us from ramming the beacon.

## Known gotchas from the guidelines

- **§7.4 frequency bleed-through**: when the green beacon is within ~1–2 ft,
  the 1 kHz-tuned circuit will still see it. `confirmBeaconFrequency()`
  addresses this — it samples both frequencies and only accepts a reading
  if the target frequency is stronger than the other.
- **Digital ports 9–12 are taken by the IRB ribbon cable** — never
  reassign these.
- **Reset-to-detector-0 must happen before each 8-detector sweep** —
  `ReadPD()` does this via the counter reset line. Don't remove it.
- The red beacon's pushbutton must be physically depressed by the arm; no
  electrical shortcut exists. The robot must approach close enough that
  a reasonable arm length can reach over the top of the beacon.

## Branch / workflow state

- Work branch: `hrittik-testing`
- `robot_main.c` was pushed here via a fork (`DemonicDeception/robotc-project-code`)
  because the Linux session's GitHub token didn't have direct push rights
  on `acrossthestreet/robotc-project-code`. A PR from the fork into
  `acrossthestreet:hrittik-testing` needs to be opened manually (the PAT
  lacked `pull_request: write`).
- If the Windows session has a better-scoped token or is the repo owner,
  just push directly to `origin/hrittik-testing` and skip the fork dance.

## Immediate next steps when resuming on Windows

1. Open `robot_main.c` in RobotC on Windows.
2. Open **Robot → Debugger → Sensors** with the Cortex plugged in via USB.
3. Wiggle each physical sensor (bumper, arm limit, sonar) one at a time
   and note which port number lights up in the debugger.
4. Edit the pragma lines at the top of `robot_main.c` to match reality.
5. Compile. If the arm moves the wrong direction, flip the sign of the
   PWM values in `pressBeaconButton()` (currently `-60` drops, `+60` raises).
6. Optional: do a single 30-second "point at red beacon from 1 ft" test to
   sanity-check `stop_level`. If the robot stops 2 ft short, lower it;
   if it rams the beacon, raise it. The sonar stop at 15 cm is the
   backstop either way.
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
