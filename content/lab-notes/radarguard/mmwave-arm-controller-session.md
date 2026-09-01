---
title: "Arm controller — getting the ESP32 into the pipeline before the arm is assembled"
date: 2026-05-21
tags: [mmwave, ros2, robotics, hardware, esp32, stepper]
summary: "The EB300 arm is printing. Motors and hardware arrived today. Rather than wait for assembly, tonight's session wrote the full 6-axis ESP32 stepper firmware, the ROS 2 bridge node, a tkinter GUI controller, and a complete motor test guide — so the moment the first print finishes, there's a validated pipeline ready to wire into."
showToc: true
---

The safety pipeline is done through Phase 7. Swept-volume workspace clipping works. The classifier is running. The background model persists across reboots. What the pipeline doesn't have yet is real joint angles — `fake_joint_states.py` is publishing zeros and the swept-volume geometry is computing from a static home position.

To fix that, the arm needs to move. And for the arm to move, the ESP32 needs firmware, the Jetson needs a bridge node, and both need to be wired together before the full assembly is ready to test. Tonight covered all of that, plus a jig test approach that means the first live end-to-end validation doesn't have to wait for a finished arm.

## The jig test idea

The EB300 prints are running on two machines. Some of the longer prints will take a while. The motors and hardware arrived today — which means there's a Nema 17, a TB6600, a limit switch, and an ESP32 all ready to go right now, while the arm body is still printing.

The plan: tape a flat stick or piece of wood to the motor shaft, secure it to the bench, run a continuous sweep. Walk into the arc. The mmWave stack sees ego-motion from the sweeping joint, runs the background model, runs the classifier, runs the swept-volume clipper, and triggers CAUTION → STOP as the person enters the workspace. Step back, resume, the sweep continues.

That's a complete end-to-end test of the full Phase 7 pipeline with one motor and a piece of scrap wood.

## ESP32 firmware

`radarguard_arm_controller.ino` drives all six motors from a single ESP32 DEVKITV1. The GPIO allocation was already pinned in the context from earlier planning:

```
Motor 0 (base)     STEP: GPIO13  DIR: GPIO12  LIMIT: GPIO34
Motor 1 (shoulder) STEP: GPIO14  DIR: GPIO27  LIMIT: GPIO35
Motor 2 (elbow)    STEP: GPIO26  DIR: GPIO25  LIMIT: GPIO32
Motor 3 (forearm)  STEP: GPIO33  DIR: GPIO19  LIMIT: GPIO39
Motor 4 (wrist1)   STEP: GPIO18  DIR: GPIO17  LIMIT: GPIO36
Motor 5 (wrist2)   STEP: GPIO16  DIR: GPIO4   LIMIT: GPIO23
```

The single most important decision in the firmware is making microstep resolution adjustable from one define:

```cpp
#define MICROSTEP_DIVISOR  8
```

Change that one number and every steps-per-degree calculation in the firmware adjusts automatically. The TB6600 DIP switches and the firmware value just need to match. For the demo build, 1/8 microstep is a good balance of smoothness and torque — smooth enough to look intentional, responsive enough to stop fast.

Steps-per-rev at 1/8: `200 × 8 = 1600`. One full revolution is 1600 steps. A 180° jog is 800 steps. Those numbers appear throughout the test guide because they're easy to remember and easy to verify by watching the shaft.

### Homing and limit switches

The KW12-3 SPDT limit switches are wired in normally-closed configuration — COM to GND, NC terminal to the ESP32 input pin with a pullup resistor to 3.3V. Normal state is HIGH. Triggered state is LOW. A broken wire also reads LOW. The system fails safe toward triggered, not toward clear.

There's a hardware catch worth noting: GPIO 34, 35, 39, and 36 on the DEVKITV1 are input-only pins with no internal pullup hardware. The firmware calls `INPUT` not `INPUT_PULLUP` on those pins — an external 10kΩ resistor from pin to 3.3V is required. Joints 2 and 5 use GPIO 32 and GPIO 23 which support `INPUT_PULLUP` and need no external resistor. Missing the external pullup on the other four joints means the limit pins float and the homing routine misbehaves.

The homing routine drives each joint toward its limit at a reduced speed, zeros the step count on contact, backs off about 18°, and re-zeros. Timeout is 8 seconds — if the limit isn't reached, the firmware warns and skips rather than driving the motor indefinitely.

### Serial protocol and commands

The firmware publishes joint angles over USB serial at 10Hz:

```
J,0.0000,0.0000,0.0000,0.0000,0.0000,0.0000
```

Six comma-separated radian values, one line per frame. The bridge node on the Jetson reads this and publishes to `/joint_states`. The safety node subscribes to `/joint_states` exactly as it does from `fake_joint_states.py` — no pipeline changes.

For individual joint testing, the firmware accepts serial commands:

```
HOME 0          — home joint 0 (or HOME for all)
JOG 0 800       — jog joint 0 forward 800 steps
JOG 0 -800      — jog joint 0 backward 800 steps
SWEEP 0 800 10  — sweep joint 0 ±800 steps, 10 times
SPEED 0 600     — set joint 0 step period to 600µs
SPEED ALL 800   — set all joints to 800µs
STATUS          — print step counts, angles, limit states
STOP            — halt all motion immediately
```

The individual joint test mode was a deliberate design decision. When the arm is first wired, you want to commission one joint at a time — home it, jog it through range, confirm the limit switch fires in the right direction — before running all six together. `JOG 0 400` and `JOG 0 -400` with the shaft in view is faster and safer than trying to debug a 6-axis homing sequence.

## ROS 2 bridge node

`arm_controller_node.py` sits on the Jetson and does two things: reads the serial joint state stream from the ESP32 and publishes it to `/joint_states`; and subscribes to `/arm_cmd` (std_msgs/String) and forwards any string it receives to the ESP32 over serial.

The bridge runs in a background thread with reconnect logic — if the serial connection drops, it retries every 3 seconds without blocking the ROS 2 spin. A watchdog timer logs a warning if no joint data has arrived in 2 seconds, which is enough to catch a disconnected cable before the safety node's own joint states timeout fault fires.

The `/arm_cmd` passthrough is the mechanism that lets everything — the terminal, the GUI, and eventually any other ROS 2 node — talk to the ESP32 using the same string protocol:

```bash
ros2 topic pub --once /arm_cmd std_msgs/String "data: 'SWEEP 0 800 5'"
```

No special serial access needed from any node that isn't the bridge. One owner of the serial port, everyone else goes through the topic.

## GUI controller

`arm_controller_gui.py` is a tkinter desktop app that runs on the Jetson. Tkinter is in the standard library — no install, no dependencies beyond what's already there for ROS 2.

The layout:

- **Joint selector** — six buttons across the top, one per joint, highlights the active one. All jog, sweep, speed, and single-joint home commands target the selected joint.
- **Jog panel** — ◀ JOG − and JOG + ▶ buttons with a configurable step count entry.
- **Sweep panel** — amplitude and count inputs, single launch button.
- **Speed slider** — 200–2000µs range, set per joint or all at once.
- **Homing** — home selected, home all, and STATUS in one row.
- **STOP** — full-width red button, always visible at the bottom of the left column. Sends immediately.
- **Joint angle readout** — live 10Hz display of all six joint positions in radians, updated from `/joint_states`.
- **Console** — timestamped log of every command sent and every line echoed back from the ESP32, color-coded: commands in accent blue, confirmations in green, warnings in yellow, errors in red.

The GUI also has offline mode — if `rclpy` isn't importable it still launches, logs commands to the console with an `[OFFLINE]` prefix, and lets you validate the UI layout without a running ROS stack.

The current palette is dark with cyan accents — functional, readable, easy on the eyes during long bench sessions. The DNTD Dynamics wordmark is in the title bar as a placeholder for the eventual branded launcher.

## Motor test guide

`MOTOR_TEST_GUIDE.md` replaces the original single-motor wiring doc with a two-stage guide that covers the full path from zero hardware to a validated 6-axis arm.

**Stage 1 — Single motor jig test.** Wiring tables for the TB6600 signal side, power side, and KW12-3 limit switch. The external pullup requirement called out explicitly with the circuit logic spelled out: what voltage the pin reads in each switch state and why a broken wire reads triggered rather than clear. DIP switch table with the 1/8 default marked. Step-by-step commissioning from `HOME 0` through a live sweep with the mmWave stack running. The jig test scenario — stick on the shaft, walk into the arc, confirm CAUTION → STOP → resume — is Stage 1's final validation.

**Stage 2 — Full 6-axis arm.** Additional parts, PSU current sizing, the pullup resistor map by joint, the common ground requirement, and motor cable routing guidance. Commissioning follows a strict sequence: verify each limit switch by hand before powering motors, home one joint at a time, jog through range, then home all and confirm STATUS before starting the ROS 2 stack. Swept volume stays disabled until physical link lengths are measured and entered in the YAML — enable it only after the startup log's computed max reach matches the physical arm.

## What changed in the pipeline

```
Before tonight:
  fake_joint_states.py → /joint_states → safety node

After tonight:
  ESP32 (6× stepper + 6× limit switch)
    ↕ USB serial
  arm_controller_node.py → /joint_states → safety node
  arm_controller_gui.py  → /arm_cmd → arm_controller_node.py → ESP32
```

`fake_joint_states.py` stays in the repo as the no-hardware fallback. The bridge node replaces it the moment the ESP32 is flashed and wired.

## What's next

The arm is printing. Motors and drivers are on the bench. The next session with hardware in hand:

1. Wire single motor jig — TB6600, KW12-3, external pullups on GPIO 34
2. Flash `radarguard_arm_controller.ino`, confirm serial output at 115200
3. `HOME 0` — confirm limit triggers, backoff, re-zero
4. Start `arm_controller_node.py`, confirm `/joint_states` publishing
5. Run full mmWave stack, attach sweep arm, walk into arc — validate live zone transitions with real joint angles for the first time

After that: measure link lengths with calipers, populate the YAML, enable swept volume, tune `swept_volume_self_radius_m` to match the arm link diameter, and get the demo video.
