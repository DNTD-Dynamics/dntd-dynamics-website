---
title: "Swept-volume workspace clipper — the sensor finally knows what the arm can reach"
date: 2026-05-21
tags: [mmwave, ros2, robotics, safety, kinematics]
summary: "This part teaches the safety system the arm's reachable envelope, so detections the arm can't physically reach stop triggering stops. The safety zones stop being uniform spheres and start being shaped to what the arm can actually threaten."
showToc: true
---

Every mmWave safety system shipping today — including the enterprise solutions — mounts the sensor somewhere fixed: a wall, a post, a ceiling. The sensor sees the whole room. Detections anywhere in the monitored zone trigger a stop.

That's safe. It's also blunt. A person three meters away from an arm with a one-meter reach isn't in danger — the arm can't get there. A fixed-mount sensor can't make that distinction. It doesn't know where the arm is, and it doesn't know what the arm can reach.

This next step makes the system understand the arm's reachable envelope. Detections outside what the arm can physically reach are suppressed before they ever reach the zone logic. The safety zones are no longer uniform spheres — they're shaped to what the arm can actually threaten, and that shape moves with the arm.

## Why this is the differentiator

The whole premise of an arm-mounted sensor is that it knows things a fixed sensor can't. Where the arm is. Where it's pointed. What's actually inside its working envelope versus what's just nearby.

Reachability is direction-dependent, and that's the part a fixed sphere can't capture. A person standing directly in front of the arm at full extension is in danger. The same person standing behind the base, at the same distance, is not reachable at all. A uniform zone around a fixed sensor treats those two identically. A system that knows the arm's pose doesn't — it can tighten where the arm can strike and relax where it can't, continuously, as the arm moves.

That's the entire reason RadarGuard mounts on the arm instead of the wall. This session is where that reason became real behavior instead of a design intention.

## Fail-safe by design

The workspace logic follows the same rule as the rest of the stack: it can only ever *suppress* detections the arm can't reach. It can never suppress a detection the arm *can* reach, and any failure in the reach computation defaults to passing every detection through to the zone logic. The feature can make the system smarter. It can't make it blind.

During bench testing without a real arm, there's no workspace to clip to — so the clipping bypasses itself entirely and every detection passes straight to zone logic. That's correct: no arm, no envelope, no suppression.

## What this locked

This is what locked the sensor mounting decision for the industrial arm — the mount location, orientation, and coverage geometry are all chosen around making the reachable-envelope logic work cleanly. The specific mounting and tuning that make it reliable are what ship with the kit; the approach is what's described here.

## Where the pipeline stands

The safety node now runs the full chain end to end: ego-motion compensation, background filtering, clustering, person/object discrimination, reachable-envelope enforcement, and zone classification, out to the serial, GPIO, and MQTT outputs that drive a real arm.

When the arm is fully assembled, the real link geometry goes into the configuration and the reachable-envelope shaping becomes physical instead of theoretical. A full working demo video will follow.

---

*Build logs from DNTD Dynamics. The approach is here; the production tuning ships with the kit. Questions about the engineering are welcome — [contact@dntddynamics.com](mailto:contact@dntddynamics.com).*
