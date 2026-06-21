---
title: "Full safety stack live — background learning, ego-motion compensation, and the motionless person problem"
date: 2026-05-16
tags: [mmwave, ros2, robotics, safety]
summary: "The IWR6843AOP is now running a complete ROS 2 safety pipeline on the Jetson — CLEAR/CAUTION/STOP zone detection with ego-motion compensation, voxel background learning, fault handling, and a heartbeat watchdog. One non-obvious bug dominated the session: velocity filtering was silently eating motionless people."
showToc: true
---

The safety stack is complete enough to demo. The IWR6843AOP is running a full ROS 2 pipeline on the Jetson Orin Nano Super — real-time zone detection with ego-motion compensation, background scene learning, fault handling, and a heartbeat watchdog. If a robot arm were connected right now, it would stop when a person entered the zone.

Here's what got built, what broke, and what the non-obvious parts actually were.

## Architecture

Two ROS 2 nodes, cleanly separated:

**`dntd_mmwave_driver_node.py`** — owns the hardware. Reads TLV frames from the sensor over UART, decodes them, publishes raw point clouds to `/mmwave/raw_points` and sensor health to `/mmwave/diagnostics`. Nothing in here knows about safety zones or robot arms.

**`dntd_mmwave_safety_node.py`** — owns the safety logic. Subscribes to point clouds and joint states, runs ego-motion compensation, filters background, classifies zones, publishes `CLEAR` / `CAUTION` / `STOP` to `/dntd/safety_zone`. Also publishes a 5Hz heartbeat — if the safety node dies for any reason, the arm controller can self-stop independently.

The separation matters. The driver is hardware-specific. The safety node is hardware-agnostic — it works with any sensor that publishes `PointCloud2`. When the IWRL6432AOP arrives, the driver changes, the safety logic doesn't.

## Ego-motion compensation

The sensor is arm-mounted, which means every point in the cloud is in sensor frame — a moving reference. When the arm swings, the walls appear to move. Without compensation, the zone classifier sees phantom objects everywhere.

The fix:

```
v_world = v_measured - J(q) · q_dot
```

Where `J(q)` is the Jacobian of the sensor position with respect to joint angles, and `q_dot` is joint velocities from `/joint_states`. During the stick test (no real arm), all joints are zero so compensation is effectively disabled — which is correct, the sensor isn't moving.

With a real arm, `ros2_control` feeds actual joint velocities and the compensation becomes meaningful. The URDF geometry lives in `dntd_mmwave_config.yaml` under `joint_geometry` — swap in real DH parameters when the arm arrives, no code changes required.

## Background learning

The sensor sees the world before it sees danger. Walls, the mount, furniture — all of it shows up as point cloud returns that aren't threats. Without background masking, the zone classifier is useless.

The background model uses a voxel grid (10cm cubes). At startup it runs a 15-second learning phase: counts returns per voxel, builds a map of what's always there. Once active, returns in background voxels are ignored. Returns in novel voxels go to the zone classifier.

Stand outside the field of view during the learning phase. If you're in frame during learning, you become part of the background and the system will ignore you — exactly the wrong outcome.

New persistent objects (a box left on the floor, a chair moved into the workspace) gradually become background after 30–60 seconds of consistent returns. The voxels continuously decay if not refreshed, so objects that move back out of frame are eventually forgotten.

## The motionless person problem

This was the session's main bug and it was completely non-obvious.

Early versions of the zone classifier used a velocity filter — points with near-zero doppler velocity were discarded as static clutter. The logic seemed sound: static clutter is the walls and floor, moving returns are potential hazards.

The problem: a person standing still in the workspace has near-zero doppler velocity. The velocity filter was silently discarding them. Walk into the zone, freeze — the arm would resume. Exactly backwards from safe behavior.

The fix was to remove the velocity filter from the zone classifier entirely and let the background model handle static clutter instead. The background model knows the difference between a wall (always there, every frame, consistent position) and a person standing still (novel voxel, wasn't there before learning completed). The wall is background. The person is not.

After removing the velocity filter:

- Person walks in → CAUTION → STOP ✅
- Person stops moving → stays STOP ✅
- Person leaves → CLEAR ✅

The background model is doing the work the velocity filter was supposed to do, but correctly.

## Fault architecture

Two fault modes, both fail-safe:

**Fault 1 — joint states timeout.** If `/joint_states` stops publishing (arm controller crash, cable pulled, whatever), the safety node immediately publishes STOP and a fault reason to `/dntd/safety_fault`. The arm can't resume until an explicit `True` is published to `/dntd/safety_resume`. Joint states recovering does not auto-resume — explicit operator acknowledgment required.

**Fault 2 — safety node dies.** The 5Hz heartbeat on `/dntd/heartbeat` stops. The arm controller monitors this and self-stops independently of whatever the safety node was doing. The safety stack failing safe is more important than the safety stack staying up.

## Hysteresis

Zone transitions without hysteresis produce oscillation at the boundary — CAUTION, STOP, CAUTION, STOP, three times a second, on a person standing exactly at the zone edge. Annoying and potentially damaging to the arm.

Two configurable parameters in `dntd_mmwave_config.yaml`:

- `hysteresis_frames: 3` — number of consecutive frames required to upgrade to a more dangerous zone (CLEAR → CAUTION, CAUTION → STOP). Higher = less sensitive, less bouncy.
- `clear_hysteresis_frames: 6` — frames required to downgrade to a safer zone. Intentionally asymmetric — the system is faster to escalate than to clear.

These are starting values. Tune them during stick testing against the actual environment.

## What's next

The stack is ready for a real arm. The immediate next steps:

- Fix the topic remap in `dntd_mmwave_launch.py` so it doesn't need to be specified manually at startup
- Push everything to Gitea before the Jetson SD card has an opinion about it
- Order a cheap 6-DOF arm — the kinematic chain needs real DH parameters to validate ego-motion compensation properly
- Begin micro-doppler classifier research — person vs. object discrimination, TI range-doppler matrix dataset

The demo video becomes possible once the arm is connected and the remap is fixed. That's the next gate.
