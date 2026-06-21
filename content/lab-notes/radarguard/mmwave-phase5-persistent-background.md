---
title: "Persistent background map — the sensor shouldn't relearn the room every boot"
date: 2026-05-21
tags: [mmwave, ros2, robotics, safety]
summary: "The 15-second background learning phase is necessary — but only once. Phase 5 adds a persistent voxel map saved to disk after the first learn, reloaded on every subsequent boot. Also: a novelty-aware refresh gate that means a stationary person can never be silently absorbed into the background."
showToc: true
---

The previous stack worked. Background learning masked out the walls, the sensor mount, the static clutter. The motionless person problem was solved. Zone transitions were clean.

But every time the Jetson rebooted — or the stack restarted — the sensor spent 15 seconds in a mandatory STOP while it relearned a room it had already mapped. In a real deployment that's friction. In a safety system it's a 15-second window where the arm is frozen for no reason.

Phase 5 fixes that. The background map now saves to disk after the first learn, and reloads on every subsequent boot. The 15-second STOP is a one-time event.

## Persistent map

After the learning phase completes, `background_model.py` writes the voxel grid to `~/mmwave/configs/background_map.npz`. Atomic write — tmp file first, then rename — so a crash mid-write doesn't corrupt the saved map.

On the next boot, the safety node looks for `background_map.npz`. If it finds one:

- Validates the saved voxel size matches the current config
- Validates the version field
- Loads the voxel scores directly into the background model
- Skips the 15-second STOP entirely

If validation fails (config changed, file corrupt), it falls back to a fresh learn. Fail-safe default.

The map file is gitignored. It's environment-specific — a voxel map of one workspace is meaningless in another. What ships in the repo is the code and the config. The map lives on the Jetson.

```bash
# Force a fresh learn (workspace rearranged, new mount position, etc.)
ros2 topic pub --once /dntd/relearn_background std_msgs/Bool "data: true"
# Deletes the saved map, relearns the room, saves a new map
```

## The novelty-aware refresh gate

Background learning works by counting returns per voxel. A voxel that gets hit in most frames becomes "background" and gets suppressed. The problem is what happens when someone stands still long enough.

In the original implementation, the background scores refreshed continuously. A person standing motionless in the workspace would accumulate background hits over time. Given enough frames, they'd cross the threshold and get absorbed into the background model. The system would stop seeing them.

The fix is a novelty gate in the voxel update logic. Before a voxel's score is incremented, the model checks whether it was novel at the last observe pass. If it was — if it contained points that passed through as detections — it doesn't get refreshed. Novel voxels and background voxels are tracked separately.

A stationary person stays in novel voxels. They are never promoted to background. They hold their zone indefinitely.

The wall has been there since learning started. The person has not. The model now treats those two things differently.

## Local config override

One housekeeping fix that's been on the list since the beginning: the MQTT broker IP was hardcoded in `dntd_mmwave_config.yaml`, which is committed to the public repo. That's a small thing until it isn't.

The pattern now:

- `dntd_mmwave_config.yaml` — committed, safe for public GitHub, no IPs
- `dntd_mmwave_config.local.yaml` — gitignored, Jetson-only overrides
- `configs/*.local.yaml` in `.gitignore`

The safety node takes both `--params-file` flags in order. Local wins on any key that appears in both files. No code changes, no secrets in the repo.

## What's next

Phase 5 closes the background model. The sensor now knows the room, remembers it, and can't be fooled by someone standing still.

The next gap in the pipeline is discrimination: the sensor sees novel returns, but it can't tell the difference between a person walking through the workspace and a cardboard box on a cart. That's Phase 6.
