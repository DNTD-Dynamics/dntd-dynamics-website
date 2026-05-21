# RadarGuard — mmWave Cobot Safety System

**Real-time human presence detection and collision avoidance for robot arms and mobile robots, powered by mmWave radar.**

Built by [DNTD Dynamics](https://dntddynamics.com) · Licensed under [BSL 1.1](#license)

---

## What is this?

RadarGuard is an open-source safety system that uses mmWave radar to detect people in a robot's workspace and output **CLEAR / CAUTION / STOP** zone commands in real time.

Unlike camera-based safety systems, mmWave radar:
- Works in complete darkness, dust, smoke, and welding flash
- Detects stationary people — not just movement
- Carries no PII — a radar return is not a face
- Runs at 10Hz with sub-100ms zone transition latency

Unlike fixed-mount safety scanners, RadarGuard mounts **on the arm itself** and understands the arm's reachable workspace via forward kinematics — so it only triggers on detections the arm can actually reach.

RadarGuard is designed to be **plug-and-play with any robot arm**. Drop in your URDF, set your zone distances in a YAML file, and run. No code changes required.

---

## Hardware Requirements

| Component | Part | Source |
|-----------|------|--------|
| mmWave sensor | Texas Instruments IWR6843AOPEVM | DigiKey / Mouser (~$179) |
| Compute | Jetson Orin Nano / NX, Raspberry Pi 5, or any Ubuntu 22.04 ARM/x86 board | — |
| Cable | USB-A to USB-B (standard) | — |

**Single-sensor configurations** work out of the box. The full industrial configuration is **3× IWR6843AOPEVM** mounted 120° apart on the forearm link for 360° azimuth coverage — see [Roadmap](#roadmap) (Phase 8).

**Coming soon:** IWRL6432AOP support for battery-powered mobile robots and humanoids.

---

## How it works

```
IWR6843AOP sensor (UART, forearm-mounted)
        ↓
  Driver node — decodes TLV frames → PointCloud2
        ↓
  Safety node
    ├── Ego-motion compensation (Jacobian × q_dot from /joint_states)
    ├── Background learning + persistent map (walls, fixtures masked)
    ├── DBSCAN cluster builder
    ├── Micro-doppler classifier (PERSON / OBJECT / UNKNOWN, fail-safe)
    ├── Swept-volume workspace clip (kinematic reachability)
    └── Zone classification + hysteresis
        ↓
  CLEAR / CAUTION / STOP
        ├── ROS 2 topic  (/dntd/safety_zone)
        ├── Serial UART  (Arduino, any microcontroller)
        ├── GPIO pins    (Raspberry Pi)
        └── MQTT         (home automation, custom integrations)
```

**Ego-motion compensation** — the sensor mounts on the moving arm itself. RadarGuard reads `/joint_states` from your ROS 2 controller and subtracts the arm's own velocity from every radar return, so only real-world motion triggers zone changes.

**Background learning** — on startup, RadarGuard learns the static environment (walls, fixtures, the arm mount). After learning, only novel objects trigger responses. A person standing still in the danger zone holds CAUTION — they don't disappear when they stop moving. The learned map is saved to disk and reloaded on subsequent boots, so the 15-second learning phase only runs once per workspace.

**Micro-doppler classifier** — DBSCAN clusters novel returns and a rule-based classifier labels each cluster PERSON / OBJECT / UNKNOWN using velocity spread, height span, point count, and doppler asymmetry. UNKNOWN passes through as PERSON (fail-safe). Suppressed OBJECT detections are auto-logged for future ML training. Drop-in ML upgrade planned.

**Swept-volume workspace clip** — RadarGuard computes the arm's reachable envelope from the kinematic chain every frame. Detections outside the arm's reach are suppressed — a person 3m away in front of a 1m arm doesn't trigger STOP. As the arm moves, the envelope updates with it. This is the primary differentiator over fixed-mount safety scanners.

**Hardware-agnostic outputs** — your arm controller doesn't need to speak ROS 2. Serial, GPIO, and MQTT outputs work simultaneously so any downstream controller can consume the safety signal.

---

## Quickstart

### 1. Hardware setup

Mount the IWR6843AOPEVM with the heat shield (antenna face) pointing outward into the workspace. Connect via USB.

Verify detection:
```bash
ls /dev/ttyUSB*
# Should show /dev/ttyUSB0 (CLI) and /dev/ttyUSB1 (data)
```

### 2. Install dependencies

```bash
# ROS 2 Humble (Ubuntu 22.04)
sudo apt install ros-humble-ros-base ros-humble-rosbridge-suite \
                 ros-humble-sensor-msgs ros-humble-sensor-msgs-py
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc

# Python dependencies
pip3 install pyserial numpy
```

### 3. Clone and configure

```bash
git clone https://github.com/DNTD-Dynamics/RadarGuard-mmwave-cobot-safety-system.git
cd RadarGuard-mmwave-cobot-safety-system
```

Edit `configs/dntd_mmwave_config.yaml`:
```yaml
# Set your URDF link name for the sensor mount
sensor_mount_link: "tool0"       # UR5/UR10 default

# Tune zone distances for your arm
stop_range_m:    0.5             # hard stop radius
caution_range_m: 1.2             # slow-down radius

# Background learning duration (seconds)
# Longer = more thorough static environment mask
background_learning_s: 15.0

# Swept-volume clipper (set false to disable kinematic clipping)
swept_volume_enabled: true
swept_volume_mount_joint_idx: 3  # forearm link mount
```

### 4. Run

```bash
# Terminal 1 — sensor driver
cd src && python3 dntd_mmwave_driver_node.py

# Terminal 2 — safety node
cd src && python3 dntd_mmwave_safety_node.py \
  --ros-args \
  --params-file ../configs/dntd_mmwave_config.yaml \
  -r /dntd/mmwave/raw_points:=/mmwave/raw_points

# Terminal 3 — watch zone output
ros2 topic echo /dntd/safety_zone
```

Stand clear during the 15-second background learning phase. After learning completes, walk toward the sensor — you will see `CLEAR → CAUTION → STOP` transitions.

Send resume after any fault:
```bash
ros2 topic pub --once /dntd/safety_resume std_msgs/Bool "data: true"
```

Force a fresh background relearn (after moving the arm or rearranging the workspace):
```bash
ros2 topic pub --once /dntd/relearn_background std_msgs/Bool "data: true"
```

---

## Adapting to your arm

All arm-specific geometry lives in `configs/dntd_mmwave_config.yaml`. No Python changes required.

**Step 1** — Set `sensor_mount_link` to the URDF link the sensor is attached to:
```yaml
sensor_mount_link: "tool0"        # UR5, UR10
sensor_mount_link: "link6"        # xArm6
sensor_mount_link: "torso_link"   # humanoid chest mount
```

**Step 2** — Set `sensor_mount_xyz` and `sensor_mount_rpy` to the physical offset from that link to the sensor face (measure with calipers or read from CAD).

**Step 3** — Copy your joint names and geometry into the `joint_geometry` block. Values come directly from your URDF `<joint>` elements. The swept-volume clipper uses this to compute reachability — if it sees a placeholder chain, it bypasses automatically (fail-safe).

**Step 4** — Tune `stop_range_m` and `caution_range_m` to match your arm's reach envelope and maximum speed.

**Step 5** — Set `swept_volume_mount_joint_idx` to the index of the joint your sensor mounts on (3 = forearm link for a 6-DOF arm). The max reach sphere is auto-computed from distal link lengths, or override with `swept_volume_max_reach_m`.

---

## Key parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `stop_range_m` | 0.5 m | Hard stop — anything inside this range triggers immediate STOP |
| `caution_range_m` | 1.2 m | Slow down — anything inside this range triggers CAUTION |
| `fast_approach_mps` | -0.8 m/s | Emergency stop velocity threshold — catches stumbles and falls |
| `background_learning_s` | 15 s | Scene learning duration. Increase for cluttered environments |
| `hysteresis_frames` | 3 | Frames to confirm zone upgrade. Increase to reduce boundary oscillation |
| `clear_hysteresis_frames` | 6 | Frames to confirm zone downgrade. Higher = safer resume after STOP |
| `min_snr_db` | 8.0 dB | Minimum SNR for valid detection |
| `classifier_enabled` | true | Set false to bypass micro-doppler classifier |
| `classifier_score_threshold` | 2 | Votes (1–4) needed to label a cluster PERSON |
| `swept_volume_enabled` | true | Set false to disable kinematic workspace clipping |
| `swept_volume_self_radius_m` | 0.15 m | Arm body self-exclusion sphere |
| `swept_volume_reach_margin_m` | 0.20 m | Safety buffer added to computed max reach |

---

## ROS 2 topics

| Topic | Type | Description |
|-------|------|-------------|
| `/dntd/safety_zone` | String | `CLEAR` / `CAUTION` / `STOP` |
| `/dntd/safety_fault` | String | Fault reason, empty when healthy |
| `/dntd/heartbeat` | Header | 5Hz watchdog pulse — subscribe in your controller |
| `/dntd/compensated_points` | PointCloud2 | Ego-motion compensated world-frame point cloud |
| `/dntd/safety_resume` | Bool | Send `true` to resume after fault |
| `/dntd/relearn_background` | Bool | Send `true` to retrigger background learning |
| `/mmwave/raw_points` | PointCloud2 | Raw driver output (~19Hz) |
| `/mmwave/diagnostics` | DiagnosticStatus | Sensor health |

---

## Supported hardware

| Sensor | Status | Notes |
|--------|--------|-------|
| IWR6843AOPEVM | ✅ Validated | Primary development platform |
| IWRL6432AOPEVM | 🔲 In progress | Battery-powered / mobile robot variant (Phase 9) |

| Compute platform | Status |
|-----------------|--------|
| Jetson Orin Nano Super (JetPack 6.2.2) | ✅ Validated |
| Raspberry Pi 5 (Ubuntu 22.04) | ✅ Compatible (1–3 sensors with current pipeline) |
| Any Ubuntu 22.04 ARM/x86 | ✅ Compatible |

---

## Fault handling

RadarGuard fails safe. If `/joint_states` stops publishing (arm controller crash, E-stop, cable fault), the system immediately publishes `STOP` and raises a fault on `/dntd/safety_fault`.

**Recovery requires an explicit resume** — the arm will not restart automatically when the fault clears:
```bash
ros2 topic pub --once /dntd/safety_resume std_msgs/Bool "data: true"
```

Your arm controller should also subscribe to `/dntd/heartbeat`. If the heartbeat stops (RadarGuard process dies), the controller should independently STOP without waiting to be told.

The classifier is also fail-safe — clusters labeled UNKNOWN pass through as PERSON. The swept-volume clipper is fail-safe — if it detects a placeholder kinematic chain or suppresses all points, originals pass through with a warning.

---

## Roadmap

- [x] IWR6843AOP driver and TLV parser
- [x] ROS 2 pipeline (Humble)
- [x] Ego-motion compensation via URDF Jacobian
- [x] Background scene learning with novelty-aware refresh
- [x] Motionless person detection
- [x] Persistent background map (no relearn on boot)
- [x] Hardware-agnostic outputs (serial / GPIO / MQTT)
- [x] Configurable hysteresis and zone parameters
- [x] Micro-doppler classifier — person vs. object discrimination (rule-based)
- [x] Swept-volume workspace clip (forearm mount, kinematic reachability)
- [ ] ML classifier — drop-in upgrade to rule-based (RadIOCD + workspace data)
- [ ] 3-sensor 360° array fusion
- [ ] IWRL6432AOP pipeline (battery-powered and mobile robots)
- [ ] Heartbeat / vital-signs detection during STOP zone
- [ ] Occlusion awareness (blind spots fail-safe to CAUTION)
- [ ] Asymmetric zone shapes (reach-direction aware)
- [ ] Custom PCB (IWRL6432AOP, power-optimized, robot flange mount)
- [ ] systemd auto-start on boot
- [ ] FCC Part 15.255 certification

---

## Commercial use

RadarGuard is free for research, education, and non-commercial projects under the [Business Source License 1.1](LICENSE).

Commercial use requires a license from DNTD Dynamics.
Contact: **contact@dntddynamics.com**

Commercial use includes any product, service, or internal tooling that generates revenue or is deployed in a production environment.

---

## Contributing

Issues, pull requests, and hardware compatibility reports are welcome.

If you've validated RadarGuard on a new arm or platform, open a PR to add it to the supported hardware table. If you hit a blocker getting it running, open an issue — the setup process has sharp edges and your report helps the next person.

---

## About

RadarGuard is developed by [DNTD Dynamics](https://dntddynamics.com), a robotics and dynamics engineering company based in Snohomish, Washington.

Built because the gap between "mmWave chip exists" and "working safety system a researcher can deploy in an afternoon" was too wide and too important to leave open.

---

## License

Business Source License 1.1

- **Non-commercial use:** Free — research, education, personal projects
- **Commercial use:** Requires a license from DNTD Dynamics
- **Change date:** Four years from first tagged release
- **Change license:** Apache 2.0 (becomes fully open after change date)

See [LICENSE](LICENSE) for full terms.
