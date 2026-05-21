---
title: "Swept-volume workspace clipper — the sensor finally knows what the arm can reach"
date: 2026-05-21
tags: [mmwave, ros2, robotics, safety, kinematics]
summary: "Phase 7 adds a swept-volume workspace clipper to the pipeline. Detections outside the arm's reachable envelope are suppressed before they reach zone logic. This is the primary technical differentiator over fixed-mount sensors — and the reason the hardware configuration locked at 3× IWR6843AOP on the forearm link."
showToc: true
---

Every mmWave safety system shipping today — including the enterprise solutions — mounts the sensor somewhere fixed: a wall, a post, a ceiling. The sensor sees the whole room. Detections anywhere in the monitored zone trigger a stop.

That's safe. It's also blunt. A person 3 meters away from an arm with a 1-meter reach isn't in danger. The arm can't get there. A fixed-mount sensor can't make that distinction — it doesn't know where the arm is, and it doesn't know what the arm can reach.

Phase 7 makes the system understand the arm's reachable envelope. Detections outside the workspace get suppressed before they ever reach zone logic. The safety zones are no longer uniform spheres — they're shaped to what the arm can physically threaten.

## The architecture

`swept_volume.py` runs `SweptVolumeClipper` after the micro-doppler classifier and before zone classification in the safety node pipeline:

```
Novel points → cluster → classifier → swept-volume clip → zone logic
```

Each frame, the clipper does three things:

1. **Computes the mount point** — the world-frame position of the forearm link (joint 3) from the current kinematic chain. This is where the sensors live.

2. **Self-exclusion sphere** — a configurable radius around the mount point (default 0.15m) that suppresses returns from the arm body itself. Without this, the arm's own structure would constantly show up as novel detections near the sensor.

3. **Max reach sphere** — auto-computed by summing the link lengths from joint 3 outward through joint 4, joint 5, joint 6, and the end effector, plus a configurable margin (default 0.20m). Detections beyond this radius are outside the envelope. They're suppressed.

The reach computation is live. As the arm moves to different joint configurations, the Jacobian updates the world-frame mount position every frame. The workspace boundary moves with the arm.

This matters more than it might seem. Reachability is direction-dependent — a person standing directly in front of the arm at full extension is in danger. The same person standing behind the base is not reachable regardless of range. A fixed sphere around the sensor can't make that distinction. Because the clipper recomputes the envelope from actual joint angles every frame, it can. The zones are continuously shaped to what the arm can physically threaten at that moment.

## Why the forearm link

The sensor mount location was decided during Phase 7, not after. The forearm link — the straight segment between joint 3 and joint 4 — is the right place for three reasons.

First, geometry: mounting on the forearm link gives the sensor a fixed orientation relative to the distal workspace. The wrist and end effector are the part of the arm that actually enters the human workspace during operation. The sensors need to see that zone clearly and consistently.

Second, stability: the forearm link doesn't rotate on its own axis the way the wrist links do. A flat mounting surface, stable during most operating moves, predictable sensor coverage.

Third, coverage: three sensors at 120° spacing on the forearm link gives 360° azimuth coverage of the distal workspace with no overlapping housekeeping. A fourth sensor would require an arbitration MCU, complicate USB enumeration, and add failure modes for minimal safety gain. Three covers the geometry cleanly.

## Fail-safe behavior

The clipper has two explicit failure modes, both of which default to passing all detections through.

**Placeholder chain detected.** During stick testing, the kinematic chain has no real joint data — the fake joint states publisher zeros everything. The clipper detects when joint positions look like a placeholder configuration and bypasses workspace clipping entirely. All novel detections pass to zone logic. This is correct: during testing without a real arm, there is no workspace to clip to.

**All points suppressed.** If the clipper's geometry math is wrong and would suppress every detection in a frame, it aborts the suppression, logs a warning, and passes the original points through. The clipper can never make the system blind.

```yaml
# Bypass swept-volume clipping entirely
swept_volume_enabled: false

# Tune to match actual arm link diameter
swept_volume_self_radius_m: 0.15

# Override auto-computed reach (leave at 0.0 for auto)
swept_volume_max_reach_override_m: 0.0

# Safety buffer added to auto-computed reach
swept_volume_reach_margin_m: 0.20

# Index of mount joint (forearm link = joint 3)
swept_volume_mount_joint_idx: 3
```

The startup log prints the computed max reach every time the safety node starts. First thing to check when the arm is assembled and physical measurements are in the YAML.

## Hardware configuration locked

Phase 7 work is what locked the hardware configuration:

**Industrial fixed arm:** 3× IWR6843AOP, 120° spacing, forearm link mount. The swept-volume clipper is designed around this geometry. The sensor positions in the kinematic chain YAML map directly to physical mounting positions.

**Mobile robot:** 3× IWRL6432AOP, separate product configuration. The mobile variant has a different ego-motion problem — the whole robot body is moving, not just a sensor on an arm — and the IWRL6432AOP gives access to the raw range-doppler heatmap via the Udopproc DPU in MMWAVE-L-SDK. That's Phase 9 territory and it'll get its own pipeline.

## What the pipeline looks like now

```
3× IWR6843AOP UART (120° spacing, forearm mount)
    ↓
Driver node — TLV decode → PointCloud2
    ↓
Safety node:
  Ego-motion compensation (Jacobian × q_dot)
    ↓
  Background model (novel voxel filter)
    ↓
  DBSCAN cluster builder
    ↓
  Micro-doppler classifier (PERSON / OBJECT / UNKNOWN)
    ↓
  Swept-volume clipper (workspace boundary enforcement)
    ↓
  Zone logic → CLEAR / CAUTION / STOP
    ↓
  Serial + GPIO + MQTT outputs
```

The arm is printing. When it's assembled, the link lengths go into the YAML and the swept-volume geometry becomes real. The demo video becomes possible after that.
