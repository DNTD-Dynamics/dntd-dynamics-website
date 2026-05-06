---
title: "Project Lumina — first light through the sphere"
date: 2026-04-15
tags: [lumina, optics, first-light]
summary: "The sphere is beautiful. The projection isn't — yet. Caustic interference is worse than the ray-trace simulator predicted at close projection distances. Adjusting DLP-to-sphere distance next."
showToc: false
---

The 50mm borosilicate glass sphere arrived. It's optically perfect — no visible inclusions, smooth surface, completely clear.

The Canon Rayo S1 pico projector projects into the bottom of the sphere. The plan: light refracts through the sphere, scatters off a frosted acrylic dome above, and you see a floating image in the dome. The north star is the holographic display from the film *65* — compact, self-contained, genuinely volumetric-looking.

## What the simulator said

The Python ray-trace simulator (`raytrace_sim.py`) predicted the sphere would focus light fairly cleanly through its center point and diverge it upward into the dome with manageable aberration, depending on projection distance.

## What actually happened

Caustic interference patterns. The curved surface of the sphere acts as a converging lens, and at close projection distances the focal point lands inside or near the dome surface rather than behind it. The result is concentric ring artifacts — bright bands where the refracted rays intersect, dark bands where they diverge.

The simulator underestimated the severity at short distances because it was modeling monochromatic rays. The Rayo S1 projects broadband visible light, and the sphere introduces chromatic aberration on top of the geometric caustics. The combined effect is worse than the simulation.

## What's next

Two variables to test:
1. **Increase DLP-to-sphere distance** — moving the projector further away changes the angle of incidence and shifts where the focal point lands relative to the dome
2. **Frosted acrylic density** — a more diffuse scatter surface may break up the caustic patterns at the cost of image brightness

The plano-convex lens fallback is still on the table if the sphere aberration can't be tamed. But the sphere is the aesthetic — if the sphere doesn't work, the project changes character entirely. Worth spending more time on it before punting.

Phase 1 POC criterion: a recognizable image visible in the dome under normal room lighting. Not there yet. Close enough to keep going.
