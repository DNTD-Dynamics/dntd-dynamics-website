---
title: "Persistent background map — the sensor shouldn't relearn the room every boot"
date: 2026-05-19
tags: [mmwave, ros2, robotics, safety]
summary: "The startup background-learning phase is necessary — but only once. Phase 5 makes the learned map persist across reboots, and closes the loophole where a person standing still long enough could be quietly absorbed into the background."
showToc: true
---

The previous stack worked. Background learning masked out the walls, the sensor mount, the static clutter. The motionless-person problem was solved. Zone transitions were clean.

But every time the system restarted, the sensor spent its startup learning phase in a mandatory STOP while it relearned a room it had already mapped. In a real deployment that's friction. In a safety system it's a window where the arm is frozen for no reason. This needs to be adjusted so that it is a one-time event instead of an every-boot tax.

## Learn once, remember it

After the first learning phase completes, the background map is written to disk and reloaded on every subsequent boot. The write is atomic, so an interrupted save can't leave a corrupt map behind. On the next start, the system validates the saved map against the current configuration and loads it if it's consistent — skipping the startup STOP entirely.

If anything about that validation fails — configuration changed, file unreadable — it falls back to a fresh learn. The default is always the safe one: when in doubt, relearn rather than trust a map that might not match the room.

The map is environment-specific. A voxel map of one workspace is meaningless in another, so it lives on the device and isn't part of the repo. What's shared is the code and the configuration, not the learned room.

## The part that actually mattered: not absorbing a stationary person

Background learning works by noticing what's consistently present and treating it as permanent. That creates a subtle danger. If the model kept refreshing indefinitely, a person standing motionless in the workspace would slowly accumulate "always there" evidence and, given enough time, get absorbed into the background — at which point the system would stop seeing them. That is the exact wrong failure for a safety system.

This loophole needed to be closed. The model now distinguishes between something that has been present since learning began — a wall — and something that appeared *after* learning, which is what a person who walked in and stopped looks like. The first is allowed to be background. The second is never promoted into it, no matter how long they hold still. A stationary person holds their zone indefinitely.

The wall has been there since the beginning. The person has not. The model now treats those two things differently, and that distinction is the whole point.

## What's next

This closes the background model. The sensor knows the room, remembers it across restarts, and can't be fooled by someone standing still.

The next gap is discrimination. The sensor can tell something new is there — but it can't yet tell a person walking through from a cart rolling past.

---

Questions welcome — [contact@dntddynamics.com](mailto:contact@dntddynamics.com).