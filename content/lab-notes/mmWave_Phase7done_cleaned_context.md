# RadarGuard — mmWave Cobot Safety System
# Context & Session State
*Last updated: May 21, 2026 (Phase 7 complete, arm build in progress)*

---

## Project Identity

**Product name:** RadarGuard
**Repo name:** RadarGuard-mmwave-cobot-safety-system
**GitHub org:** DNTD-Dynamics
**GitHub URL:** github.com/DNTD-Dynamics/RadarGuard-mmwave-cobot-safety-system
**Repo visibility:** PUBLIC as of May 20, 2026
**Company:** DNTD Dynamics (Snohomish, Washington)
**Contact:** contact@dntddynamics.com
**Website:** dntddynamics.com (GitHub Pages + Cloudflare, repo separate from RadarGuard)

**License:** Business Source License 1.1 (BSL 1.1)
  - Free for non-commercial use (research, education, personal projects)
  - Commercial use requires license from DNTD Dynamics
  - Change date: 4 years from first public release → Apache 2.0
  - LICENSE file manually added (GitHub picker does not include BSL 1.1)
  - Do NOT use "Boost Software License" — that is a different, unrelated license

**GitHub repo topics:** mmwave, ros2, cobot-safety, robotics, ti-radar,
  iwr6843aop, iwrl6432aop, robot-safety, human-detection

**Git remotes (on Jetson):**
  - `github` → git@github.com:DNTD-Dynamics/RadarGuard-mmwave-cobot-safety-system.git
  - `gitea`  → http://192.168.254.117:3000/ShireFolk/dntd-mmwave.git (local backup)

Push commands:
  - `git push github main` — public, always reachable
  - `git push gitea main`  — local backup, only when on home network

---

## Hardware

### IWR6843AOPEVM — PRIMARY (fully validated)
- **Connection:** Jetson Orin Nano Super via USB
- **Ports:** `/dev/ttyUSB0` (CLI, 115200) · `/dev/ttyUSB1` (Data, 921600)
- **Firmware:** `out_of_box_6843_aop.bin` — version 03.06.00.00
- **Config:** `~/mmwave/configs/profile_AOP.cfg` — validated, 10Hz, ±60° FOV
- **Mounting:** Vertical, heat shield (antenna face) pointing outward into workspace
- **Switch config (normal):** S1 all off, S2.1 off, S2.2/3/4 on, S3 on
- **Switch config (flash):** S1 all off, S2.1+S2.2 on, S2.3+S2.4 off, S3 on

### IWRL6432AOPEVM — IN PROGRESS (out of stock, 3 units on order)
- Battery-powered / mobile robot variant — primary product vision
- Identical TLV format to 6843 — same parser, same driver node
- Main new challenge: platform ego-motion (whole robot body moving)
- SDK: MMWAVE-L-SDK v1.4.0 (released Aug 2025) — recently stable
- Almost no open source work on this chip — wide open territory
- Out of stock due to demand outpacing supply — market signal
- NOTE: Udopproc (micro-doppler DPU) IS available for IWRL6432AOP via
  MMWAVE-L-SDK — not available for IWR6843. Future opportunity to use
  raw range-doppler heatmap on the 6432 for richer classification.

### Planned sensor array

**Industrial fixed arm (IWR6843AOP):**
- 3× IWR6843AOPEVM — mounted 120° apart on forearm link (between joint 3
  and joint 4) for full 360° azimuth coverage of the distal workspace
- Mount point is the straight forearm link — stable orientation, doesn't
  rotate on its own axis, flat mounting surface for clean sensor placement
- 4th sensor deliberately NOT added — introduces housekeeping MCU for
  signal arbitration, USB enumeration complexity, and additional failure
  modes with minimal safety gain. 3 sensors cover the geometry cleanly.
- Wrist close-range gap: Phase 9 problem, handled by mobile variant sensor

**Mobile robot (IWRL6432AOP):**
- 3× IWRL6432AOP — mobile/battery robot applications
- Separate product configuration — same core pipeline, different ego-motion
- No wrist sensor needed on mobile variant — 3x 6432AOP covers the geometry
- Dynamic self-model replaces static background map (whole robot body moves)

### Demo / Validation Arm — ToolBox Robotics EB300
- **Design:** Open source 6-DOF collaborative arm
  - Thingiverse: https://www.thingiverse.com/thing:6283770
  - ToolBox Robotics site has STEP files for exact link length measurements
- **Print status:** Printing now in PLA on two printers
  - PLA acceptable for demo and validation — not production
  - Joints near Nema 23s are highest stress — reprint in PETG/ABS for permanent build
  - Aesthetic flanges skipped — get it moving first, pretty later
  - Forearm link (sensor mount location) is low stress — PLA fine permanently
- **Motors:** Nema 17 (joints 1-3) + Nema 23 (joints 4-6) stepper motors
- **Drivers:** TB6600 stepper drivers (one per motor, up to 4A)
- **Motion controller:** ESP32 (on hand)
- **Link lengths:** To be measured after assembly — populate joint_geometry
  in dntd_mmwave_config.yaml from physical measurements with calipers

**IMPORTANT — joint geometry measurement guide (when assembled):**
For each joint, measure straight-line distance from center of that joint's
rotation axis to center of next joint's rotation axis along each dimension.
Record as origin_xyz for each joint in the YAML.
```
joint1: base rotation → shoulder pivot          Z height (vertical)
joint2: shoulder pivot → elbow pivot            X along upper arm
joint3: elbow pivot → forearm sensor mount      X along forearm
joint4: sensor mount → wrist bend               X remaining forearm
joint5: wrist bend → wrist rotation             Y short offset
joint6: wrist rotation → end effector           Y or Z short offset
```
Bring measurements to next session — will translate directly to YAML.

### Motion Control Architecture (ESP32 → TB6600 → ROS 2)

**Why position feedback is needed:**
Swept-volume clipper and ego-motion compensator both call update(q) every
frame with current joint angles. Without real position, q=zeros — system
thinks arm is always at home, swept-volume geometry is wrong, ego-motion
compensation is blind to actual motion.

**Chosen approach: Stepper dead reckoning (no encoders for demo)**
- Steppers don't slip under normal load at demo speeds
- Count steps from known home position, convert to angles
- Homing routine on startup drives each joint to limit switch, zeros count
- Good enough for demo and early validation
- Add AS5600 magnetic encoders (I2C, ~$5-10 each) for production build

**Architecture:**
```
Jetson (ROS 2)
    ↕ USB serial (same pattern as mmWave UART pipeline)
ESP32
  — Generates STEP/DIR pulses to 6× TB6600
  — Tracks cumulative step count per motor
  — Homing routine on startup (limit switches)
  — Publishes joint angles at 10Hz over serial
  — Accepts velocity/position commands from Jetson
    ↕ STEP/DIR signals
6× TB6600
    ↕
6× Nema 17/23
```

**Jetson ROS 2 bridge node (to be written):**
- Reads joint angles from ESP32 over serial
- Publishes to /joint_states — feeds safety node directly
- Forwards motion commands from ROS 2 to ESP32

**ESP32 GPIO allocation (avoids strapping pins and ADC2/WiFi conflicts):**
```
Motor 1 (base)     STEP: GPIO13  DIR: GPIO12  LIMIT: GPIO34
Motor 2 (shoulder) STEP: GPIO14  DIR: GPIO27  LIMIT: GPIO35
Motor 3 (elbow)    STEP: GPIO26  DIR: GPIO25  LIMIT: GPIO32
Motor 4 (forearm)  STEP: GPIO33  DIR: GPIO19  LIMIT: GPIO39
Motor 5 (wrist1)   STEP: GPIO18  DIR: GPIO17  LIMIT: GPIO36
Motor 6 (wrist2)   STEP: GPIO16  DIR: GPIO4   LIMIT: GPIO23
```

**Outstanding questions before writing ESP32 firmware:**
- Limit switches: on hand or ordering with motors this week?
- Confirm ESP32 board variant (DevKit v1, S3, etc.) for pin validation

**Future upgrade: AS5600 magnetic encoders**
- One per joint, mounted on motor shaft
- Absolute position, no homing needed
- I2C direct to ESP32 (~$5-10 each, ~$40 total for 6 joints)
- Feeds both safety system and future closed-loop control
- Natural Sterling integration via MQTT (same pattern as other robot work)

### Compute Platforms
- **Primary:** Jetson Orin Nano Super (JetPack 6.2.2, Ubuntu 22.04 jammy)
  - IP: 192.168.254.117 (local) · 100.67.44.69 (Tailscale)
- **Compatible:** Raspberry Pi 5 (Ubuntu 22.04 jammy)
  - Handles 1–3 sensors with current zone logic + background model
  - Limit: micro-doppler classifier + 3-sensor fusion at 10Hz → needs Jetson
- **Any Ubuntu 22.04 ARM/x86** — no Jetson-specific dependencies in stack

---

## Software Stack

### ROS 2
- **Distro:** Humble (jammy)
- **IMPORTANT:** JetPack reports Ubuntu codename as `jammy` not `noble`
  Jazzy requires noble — always use Humble on this hardware
- **Source:** added to `~/.bashrc` — auto-sourced on every login
- **Packages:** ros-humble-ros-base, ros-humble-rosbridge-suite,
  ros-humble-sensor-msgs, ros-humble-sensor-msgs-py

### SSH / GitHub
- Jetson SSH key added to github.com (nicd85 account / DNTD-Dynamics org)
- `ssh -T git@github.com` confirms authentication working
- Use SSH remote (`git@github.com:...`) not HTTPS for push

### Local config override pattern
- `dntd_mmwave_config.yaml` — committed, safe for public GitHub
- `dntd_mmwave_config.local.yaml` — gitignored, Jetson-only overrides
  - Contains: `output_mqtt_broker: "192.168.254.117"`
  - Pattern: `configs/*.local.yaml` in `.gitignore`
- Run command includes both `--params-file` flags in order (local wins)

### File Structure

```
~/mmwave/
├── configs/
│   ├── profile_AOP.cfg                    ✅ Validated chirp config
│   ├── dntd_mmwave_config.yaml            ✅ Safety node parameters (public)
│   ├── dntd_mmwave_config.local.yaml      🔒 Local overrides (gitignored)
│   ├── dntd_mmwave_driver_config.yaml     ✅ Driver node parameters
│   └── background_map.npz                 🔒 Persistent background map (gitignored)
├── logs/
│   └── classifier_training.csv            🔒 Auto-logged OBJECT suppressions
│                                             (gitignored — training data)
├── src/
│   ├── tlv_parser.py                      ✅ TLV frame decoder (original)
│   ├── uart_reader.py                     ✅ MmwaveReader UART reader (original)
│   ├── zone_logic.py                      ✅ Zone classifier + ZoneOutputs
│   ├── background_model.py                ✅ Voxel background + persistence
│   ├── cluster.py                         ✅ DBSCAN cluster builder + features
│   ├── classifier.py                      ✅ Micro-doppler rule-based classifier
│   ├── swept_volume.py                    ✅ Swept-volume workspace clipper
│   ├── dntd_mmwave_driver_node.py         ✅ UART → PointCloud2 ROS 2 node
│   ├── dntd_mmwave_safety_node.py         ✅ Full pipeline ROS 2 node
│   ├── dntd_mmwave_launch.py              ✅ Multi-sensor launch file
│   ├── fake_joint_states.py               ✅ Stick test / no-arm helper
│   ├── arm_controller_node.py             🔧 ESP32 → /joint_states bridge (TODO)
│   ├── live_angle.py                      ✅ FOV validation tool (original)
│   └── main.py                            ✅ Standalone runner (no ROS 2)
├── context.md                             ← this file
├── README.md                              ✅ Public-facing docs
└── LICENSE                                ✅ BSL 1.1
```

### Files removed (no longer needed)
- `uart_reader_adapter.py` — deleted, driver uses MmwaveReader directly
- `tlv_parser_adapter.py` — deleted, no longer in pipeline

---

## Architecture

### Full Pipeline (as of Phase 7)

```
3× IWR6843AOP UART (forearm mount, 120° spacing)
      ↓
dntd_mmwave_driver_node.py (one per sensor, separate namespace)
  MmwaveReader (uart_reader.py) → Frame objects → PointCloud2
  Publishes: /mmwave/raw_points, /mmwave/diagnostics
      ↓
  [topic remap: /dntd/mmwave/raw_points → /mmwave/raw_points]
      ↓
dntd_mmwave_safety_node.py
  ← /joint_states (fake_joint_states.py during stick test)
  → Ego-motion compensation (Jacobian × q_dot)
  → Background model (observe → filter_novel)
      — Persistent map: loads from disk on boot, skips 15s STOP
      — Novelty-aware refresh: stationary people never absorbed
  → Cluster builder (DBSCAN → Cluster objects with features)
  → Micro-doppler classifier (PERSON / OBJECT / UNKNOWN)
      — Fail-safe: UNKNOWN passes through as PERSON
      — OBJECT suppressed + logged to classifier_training.csv
  → Swept-volume workspace clip
      — Mount point: world-frame position of joint 3 (forearm link)
      — Self-exclusion sphere: arm body returns suppressed
      — Max reach sphere: auto-computed from distal link lengths
      — Fail-safe: placeholder chain or disabled → all points pass
  → Zone classification (CLEAR / CAUTION / STOP)
  Publishes: /dntd/safety_zone, /dntd/safety_fault,
             /dntd/heartbeat, /dntd/compensated_points
  Subscribes: /dntd/safety_resume, /dntd/relearn_background
  Downstream: Serial / GPIO / MQTT (ZoneOutputs)
```

### Future Pipeline (Phase 8+)

```
Novel points → cluster → classifier → swept-volume clip
        ↓
Occlusion mask (blind spots behind links → fail-safe CAUTION)  ← Future
        ↓
3-sensor fusion (union of 3 point clouds, coord-aligned)       ← Phase 8
        ↓
Zone decision
```

### Topic Map
| Topic | Direction | Type | Notes |
|-------|-----------|------|-------|
| `/mmwave/raw_points` | driver → safety | PointCloud2 | raw sensor frames ~19Hz (measured; chirp config is 10Hz, driver publishes per-TLV) |
| `/mmwave/diagnostics` | driver → | DiagnosticStatus | sensor health |
| `/joint_states` | arm/fake → safety | JointState | ego-motion input |
| `/dntd/safety_zone` | safety → | String | CLEAR/CAUTION/STOP |
| `/dntd/safety_fault` | safety → | String | fault reason or empty |
| `/dntd/heartbeat` | safety → | Header | 5Hz watchdog pulse |
| `/dntd/compensated_points` | safety → | PointCloud2 | world-frame cloud |
| `/dntd/safety_resume` | → safety | Bool | clear fault, resume arm |
| `/dntd/relearn_background` | → safety | Bool | retrigger background learning |

---

## Current Status

### Validated ✅
- IWR6843AOP hardware — firmware, config, UART, TLV parsing
- Full ROS 2 Humble pipeline end-to-end — confirmed live zone transitions
- CLEAR → CAUTION → STOP tracking person moving through space
- Background learning — 15s startup, walls and mount masked out
- Motionless person detection — holds zone when movement stops
- Fast approach detection — STOP on stumble/fall velocity from caution zone
- Fault handling — joint_states timeout → STOP → explicit resume required
- Heartbeat watchdog — 5Hz, arm controller can self-stop if node dies
- Serial / GPIO / MQTT outputs (ZoneOutputs — all simultaneous)
- Configurable hysteresis via YAML
- README.md written and committed
- LICENSE (BSL 1.1) added and committed
- Repo public at github.com/DNTD-Dynamics/RadarGuard-mmwave-cobot-safety-system

### Phase 5 Complete ✅
- **Persistent background map** — voxel grid saved to
  `~/mmwave/configs/background_map.npz` after learning, reloaded on boot.
  Atomic write (tmp + rename). Version + voxel_size validated on load.
  Skips 15s STOP every boot after first learn.
- **Novelty-aware refresh gate** — novel voxels (people) are never
  refreshed into the background score. A stationary person stays detected
  indefinitely and is never absorbed into the background.
- **Local config override** — `dntd_mmwave_config.local.yaml` gitignored,
  loaded as second `--params-file` (overrides main config). No secrets
  in committed files.
- **Driver config path fix** — removed hardcoded `/home/nic/` path,
  replaced with `~/mmwave/configs/` for portability.

### Phase 6 Complete ✅
- **cluster.py** — DBSCAN cluster builder. Groups novel DetectedPoints
  into Cluster objects with features: centroid, range, point_count,
  velocity_spread, height_span, doppler_asymmetry, snr_mean,
  temporal_variance (centroid movement history across frames).
  Persistent cluster IDs across frames via centroid matching.
- **classifier.py** — Rule-based micro-doppler classifier. Labels each
  cluster PERSON / OBJECT / UNKNOWN. Fail-safe: UNKNOWN passes through.
  Soft voting (4 feature votes): velocity_spread, height_span,
  point_count (gated by corroborating features), doppler_asymmetry.
  All thresholds YAML-tunable. Automatically logs suppressed OBJECT
  detections to `~/mmwave/logs/classifier_training.csv` for future
  ML training data collection.
- **dntd_mmwave_safety_node.py** — updated to wire cluster builder and
  classifier into the pipeline. `classifier_enabled: false` in YAML
  restores original behavior instantly.
- **dntd_mmwave_config.yaml** — classifier parameters added with full
  tuning guide comments.

### Phase 7 Complete ✅
- **swept_volume.py** — SweptVolumeClipper computes reachable workspace
  from sensor mount joint (joint 3, forearm link) each frame. Defines:
  - Mount position: world-frame origin of mount joint at current q
  - Self-exclusion sphere: suppresses arm body returns (default 0.15m)
  - Max reach sphere: auto-computed from distal link lengths (joints 4→6
    + end effector + sensor mount offset) or YAML override
  - Reach margin: 0.20m safety buffer added to max reach
  - Fail-safe: placeholder chain detected and bypassed automatically
  - Fail-safe: if all points suppressed, originals pass through with warning
  - `swept_volume_enabled: false` bypasses entirely
- **dntd_mmwave_safety_node.py** — SweptVolumeClipper wired in after
  classifier, before zone logic. Startup log prints computed max reach.
- **dntd_mmwave_config.yaml** — swept-volume parameters added with full
  tuning guide. `swept_volume_mount_joint_idx: 3` for forearm mount.
- **Configuration-space awareness** — clipper updates with q every frame,
  so envelope tracks arm motion (person reachable at full extension vs.
  not reachable behind the base, automatically).

**Hardware configuration locked:**
- Industrial fixed arm: 3× IWR6843AOP, 120° spacing, forearm link mount
- Mobile robot: 3× IWRL6432AOP (Phase 9) — separate product configuration
- No 4th sensor on industrial arm — housekeeping MCU overhead not justified

### Known Issues / Outstanding Work
- **Zone boundary oscillation** — tune hysteresis_frames and
  clear_hysteresis_frames in YAML. Relearn from outside FOV first.
- **Classifier thresholds need real-world tuning** — defaults are
  conservative (fail-safe). Run with real arm + real workspace and
  review classifier_training.csv to dial in.

---

## Running the Stack

### Every Session (4 terminals)

```bash
# Terminal 1
cd ~/mmwave/src && python3 fake_joint_states.py

# Terminal 2
cd ~/mmwave/src && python3 dntd_mmwave_driver_node.py

# Terminal 3
cd ~/mmwave/src && python3 dntd_mmwave_safety_node.py \
  --ros-args \
  --params-file ~/mmwave/configs/dntd_mmwave_config.yaml \
  --params-file ~/mmwave/configs/dntd_mmwave_config.local.yaml \
  -r /dntd/mmwave/raw_points:=/mmwave/raw_points

# Terminal 4
ros2 topic echo /dntd/safety_zone
```

### After First Boot (background map not yet saved)
```bash
# Clear initial joint_states fault
ros2 topic pub --once /dntd/safety_resume std_msgs/Bool "data: true"

# Trigger clean relearn (stand outside FOV first)
ros2 topic pub --once /dntd/relearn_background std_msgs/Bool "data: true"
# Map saves automatically after 15s — subsequent boots skip this
```

### After Workspace Change (force fresh learn)
```bash
ros2 topic pub --once /dntd/relearn_background std_msgs/Bool "data: true"
# Deletes saved map, relearns, saves new map
```

### Useful Diagnostics
```bash
ros2 topic list
ros2 topic hz /mmwave/raw_points          # expect ~19Hz
ros2 topic hz /dntd/safety_zone           # expect ~10Hz
ros2 topic echo /mmwave/diagnostics
tail -f ~/mmwave/logs/classifier_training.csv  # watch suppressions live
```

---

## Key YAML Parameters

### dntd_mmwave_config.yaml (safety node)
| Parameter | Default | Notes |
|-----------|---------|-------|
| `sensor_mount_link` | "tool0" | URDF link sensor attaches to |
| `stop_range_m` | 0.5 | Hard stop radius |
| `caution_range_m` | 1.2 | Slow-down radius |
| `fast_approach_mps` | -0.8 | Emergency stop velocity |
| `min_snr_db` | 8.0 | Detection quality threshold |
| `background_learning_s` | 15.0 | Scene learning duration |
| `background_voxel_size_m` | 0.10 | 10cm voxel resolution |
| `background_hit_threshold` | 0.30 | Fraction of frames → background |
| `hysteresis_frames` | 3 | Frames to confirm zone upgrade |
| `clear_hysteresis_frames` | 6 | Frames to confirm zone downgrade |
| `classifier_enabled` | true | Set false to bypass classifier |
| `classifier_eps_m` | 0.40 | DBSCAN cluster radius |
| `classifier_velocity_spread_min` | 0.08 | Person vel spread threshold |
| `classifier_height_span_min` | 0.10 | Person height threshold |
| `classifier_score_threshold` | 2 | Votes needed for PERSON (1–4) |
| `classifier_log_enabled` | true | Log suppressed objects to CSV |
| `swept_volume_enabled` | true | Bypass with false |
| `swept_volume_mount_joint_idx` | 3 | Forearm link mount |
| `swept_volume_self_radius_m` | 0.15 | Arm body self-exclusion |
| `swept_volume_reach_margin_m` | 0.20 | Safety buffer on max reach |
| `output_mqtt_broker` | "" | Set in .local.yaml — never commit IP |

---

## Future Work — Detailed Notes

### Phase 6b — ML Classifier (Drop-in upgrade to Phase 6 rule-based)

**Design intent:** Drop-in replacement for the rule-based classifier in
classifier.py. Same Cluster feature vector input, same
PERSON/OBJECT/UNKNOWN output. No pipeline changes required.

**Training data:** classifier_training.csv auto-logs every suppressed
OBJECT detection with full feature vector. Every hour of operation
with a real arm builds labeled training data automatically.

**Recommended dataset:** RadIOCD — 76,821 samples, sparse point cloud
from mmWave radar, 5 classes (backpack, chair, desk, human, wall),
3 environments. Published Aug 2024. Use for initial model training,
fine-tune on workspace-specific data from classifier_training.csv.
URL: https://www.nature.com/articles/s41597-024-03678-2

**Model target:** sklearn RandomForest or lightweight tflite model.
Must run at 10Hz on Jetson AND Raspberry Pi 5. No GPU inference.
Target: <5ms inference per frame.

**Commercial opportunity:** Pre-trained models as paid optional add-on.
Rule-based (free, ships to everyone) + ML upgrade (paid, for those
without time/resources to train). Natural tiered offering under BSL.

**IWRL6432AOP opportunity:** Udopproc DPU available in MMWAVE-L-SDK
for IWRL6432AOP (not available for IWR6843). Raw range-doppler heatmap
accessible on the 6432 → richer micro-doppler features possible →
better classifier accuracy. Design Phase 6b to support both feature
vectors (point cloud features for 6843, heatmap + point cloud for 6432).

### Phase 8 — 3-Sensor 360° Fusion

- 3× IWR6843AOPEVM arriving — coverage gaps closed
- Each sensor driver node in separate namespace (/dntd/sensor_0/1/2)
- Safety node subscribes to all three raw_points topics
- Fusion: union of novel point clouds from all sensors
- Cluster builder handles merged cloud (DBSCAN naturally handles it)
- Key challenge: coordinate frame alignment between sensor mounts

### Phase 9 — IWRL6432AOP Pipeline

- Same TLV format → same driver node, same parser
- New challenge: whole-platform ego-motion (robot body moving, not
  just sensor on arm)
- Dynamic background model replaces static learned map
- Udopproc DPU available → raw range-doppler heatmap accessible
- Almost no open-source work on this chip — first-mover territory
- Custom PCB target: IWRL6432AOP, power-optimized, robot flange mount

### Phase — Heartbeat Detection During STOP

**Status:** Unblocked (Phase 7 swept-volume work complete). Not yet started.

**Concept:** When a STOP zone is triggered and the arm halts, shift to
a heartbeat/vital-signs detection mode to maintain high-confidence
presence detection even when the person is completely stationary.
Holds the STOP until a clear is given rather than risking false CLEAR
from a motionless person.

**Architecture:**
- Heartbeat detection triggers on STOP zone entry
- Separate chirp config required (slower chirp for vital signs —
  different from current 10Hz motion config)
- Uses the vital signs TLV output from IWR6843AOP firmware
- Detection pipeline shifts: motion classification → presence
  confirmation via micro-motion (breathing: 2–5mm chest displacement,
  range ~1–2m)
- STOP held until both zone = CLEAR AND heartbeat signal absent
- On CLEAR from heartbeat mode, explicit resume still required

**Implementation notes:**
- Requires runtime firmware config switch or dual-mode config file
- Short range limitation (~1–2m) — complements rather than replaces
  zone-based detection at longer range
- Natural integration: heartbeat plugin in Sterling for ambient
  presence awareness (separate use case from safety)

### Phase — Payload / Tool Extension Awareness

**Problem:** Arm holding a 2m pipe has effective reach of arm_reach + 2m.
Current swept-volume calculation would underestimate danger zone.

**Approach:**
- `payload_length_m` and `payload_direction` parameters in YAML
- Operator declares payload before operation starts
- Kinematic envelope expands by payload vector
- Eventually: wrist-mounted sensor (already planned) auto-detects
  payload geometry from returns near end effector

**Mobile robot version (Phase 9+):**
- Robot body + payload must be excluded from detections
- Dynamic self-model replaces static background map (robot is moving)
- IWRL6432AOP on mobile robot — ego-motion problem extends to full body

### Phase — Occlusion Awareness

**Problem:** Sensor has blind spots behind arm links. System should know
when a zone is occluded and fail-safe to CAUTION rather than CLEAR.

**Approach:**
- Ray-cast from sensor origin through arm link geometry
- Voxels behind occluding links marked as UNKNOWN (not CLEAR)
- Zone logic treats UNKNOWN voxels as CAUTION-level
- Pairs naturally with swept-volume work (same geometry model)

### Phase — Asymmetric Zone Shapes

**Concept:** STOP/CAUTION zones currently uniform spheres around mount.
Real arm reachability is direction-dependent — full extension forward
vs. blocked by base behind. Shape zones to actual reach envelope
(builds on swept-volume foundation).

---

## Market Context

### Competitive Landscape
- **Algorized + KUKA** (announced CES Jan 2026) — IWR6843AOP, enterprise
  pricing, closed source, SIL2-certified. Validates the market entirely.
  DNTD position: open-source, accessible, plug-and-play alternative.
- **TI reference designs** — raw demos only, no safety logic, no ego-motion
- **GitHub mmwave ecosystem** — home automation, academic code, one ROS 1
  Melodic Docker wrapper. Nothing deployable for arm safety.
- **IWRL6432AOP** — one general PCB project (0 stars). No pipeline, no safety.

### Why This Is Novel
1. Ego-motion compensation via URDF Jacobian for arm-mounted mmWave — not published
2. Background learning + motionless person detection — not published
3. Novelty-aware refresh gate — stationary people never absorbed — not published
4. Persistent background map with atomic save/load — not available
5. DBSCAN cluster + micro-doppler rule-based classifier — not available
6. Auto-logging training data pipeline for ML upgrade — not available
7. Hardware-agnostic outputs (serial/GPIO/MQTT + ROS 2) — not available
8. YAML-driven config for any arm — not available
9. IWRL6432AOP pipeline — essentially no competition exists yet
10. Swept-volume workspace clipping from arm mount point — no fixed-mount sensor can do this
11. Automatic distal reach computation from kinematic chain — YAML override supported
12. Two distinct product configs: 3× IWR6843AOP industrial, 3× IWRL6432AOP mobile

### Window
- IWR6843AOP: 6–12 months before Algorized wave creates more competition
- IWRL6432AOP: 12–24 months, chip only became stable Aug 2025

---

## Build Roadmap

```
✅ Phase 1  — Hardware validation (IWR6843AOP, TLV, FOV)
✅ Phase 2  — ROS 2 pipeline (driver, safety node, ego-motion architecture)
✅ Phase 3  — Background learning, motionless detection, YAML config
✅ Phase 4  — README, BSL 1.1 license, GitHub push (repo now PUBLIC)
✅ Phase 5  — Persistent background map, novelty-aware refresh, local config
✅ Phase 6  — Micro-doppler classifier (rule-based, fail-safe, training logger)
✅ Phase 7  — Swept-volume workspace clipper (forearm mount, 3× IWR6843AOP)
── Phase 6b — ML classifier (drop-in upgrade, RadIOCD + workspace data)
── Phase 8  — 3-sensor 360° fusion (when EVMs arrive)
── Phase 9  — IWRL6432AOP pipeline (when back in stock)
── Phase 10 — Custom PCB (IWRL6432AOP, power-optimized, robot flange mount)
── Phase 11 — FCC Part 15.255 + ISO 13849 functional safety path
── Future   — Heartbeat detection during STOP zone (unblocked)
── Future   — Payload/tool extension awareness
── Future   — Occlusion awareness (blind spots → fail-safe CAUTION)
── Future   — Asymmetric zone shapes (arm-reach-direction aware)
```

---

## Immediate Next Session Priorities

**Unblocked right now (no arm needed):**
1. Add `logs/` to `.gitignore` so training CSV never hits GitHub (30s fix)
2. Confirm limit switch situation — on hand or ordering with motors?
3. Confirm ESP32 board variant (DevKit v1, S3, etc.) for GPIO pin validation
4. Write ESP32 firmware — step/dir pulse gen, homing, serial joint state publisher
5. Update fake_joint_states.py with realistic EB300 joint angle ranges for testing
6. Order 2× more IWR6843AOPEVMs for Phase 8 — long lead time, get in pipeline now

**Needs assembled arm:**
7. Measure all joint link lengths with calipers — bring to next session for YAML update
8. Write arm_controller_node.py — serial → /joint_states ROS 2 bridge against real ESP32
9. Verify Phase 7 swept-volume max reach computation vs. physical arm
10. Run stack with `swept_volume_enabled: false` first — verify classifier behavior
11. Enable swept volume, tune `swept_volume_self_radius_m` to match arm link diameter
12. Tune `classifier_score_threshold` and `classifier_eps_m` from real data

**Demo prep (after arm moves):**
13. Shoot zone transition video — person walks in, CAUTION, STOP, arm halts
14. Add GIF/screenshot to README
15. Draft Hackaday + Reddit post (two-part series: current state + 3-sensor follow-on)

---

## Regulatory / Commercial Notes

- FCC Part 15.255 required before US commercial hardware sales (~$5–8k, 6–12 weeks)
- TIDEP-01033 reference design BOM → reuse TI test data for FCC
- IEC 62061 / ISO 13849 for industrial channel sales
- Export control: EAR ECCN for international hardware sales
- BSL license enables commercial licensing revenue before hardware certification
- Pre-commercial: evaluation board + software licensing OK now
- Commercial tiering opportunity: rule-based classifier (free/BSL) +
  pre-trained ML model (paid license) + workspace-tuned model (professional services)
