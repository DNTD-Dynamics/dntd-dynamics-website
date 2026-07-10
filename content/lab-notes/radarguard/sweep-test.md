---
title: "Sweep test — the pipeline finally ran on a moving arm with a real person in the loop"
date: 2026-06-10
tags: [mmwave, robotics, safety, demo]
summary: "The first full hardware test with the sensor mounted on a rotating arm and a person walking through the workspace. CLEAR, CAUTION, and STOP all fired where they should. The arm stopped. The sensor kept up."
showToc: false
---

Everything up to this point was bench testing. The sensor sat still, the person moved, the zone transitions fired cleanly. That's useful for validating the pipeline logic, but it isn't the real problem. The real problem is a sensor bolted to a robot arm that's sweeping through space while a person is somewhere in that space — and getting a reliable safety signal out of that situation.

This was the first test where both things moved at once.

{{< youtube VIDEO_ID_HERE >}}

## The setup

The IWR6843AOP EVM is mounted at the end of a planetary gearbox arm — the sensor face pointing radially outward into the swept arc as the arm rotates. The arm sweeps continuously. A person walks toward the arc, stops inside it, and walks back out.

The sensor is on the moving thing. Everything the sensor sees is relative to a reference frame that's rotating. A static wall looks like it has velocity. The mount hardware looks like it has velocity. The person looks like they have velocity — but a different one, and it doesn't cancel the same way.

The question the test was answering: does any of that produce false triggers?

## The result

It didn't.

CLEAR held while the arm swept with nobody in the workspace. CAUTION fired as the person entered the arc at about one to one-point-two meters. STOP fired as they stepped closer. CAUTION and then CLEAR came back as they stepped out. No false triggers from the rotating mount, no hesitation at the transitions, no phantom stops between passes.

The arm halted on STOP and resumed on CLEAR. The split-screen in the video shows both at once — the physical arm on one side, the terminal output on the other — so the timing is visible without having to take anyone's word for it.

## Why the rotating mount didn't cause problems

A metal mount in the antenna beam at a fixed range would produce a persistent false return at that range. Wood and plastic are transparent to mmWave. The mount is wood and plastic. That's the whole answer for the hardware side.

On the software side, the arm sweeps at a speed that keeps the sensor's own induced radial velocity well below the static-clutter filter threshold. The filter that drops walls and furniture also drops the sensor's own motion signature at normal operating speeds. The person walking through produces a clearly different velocity signature — one that the filter passes and the zone logic acts on.

Those two things together — transparent mount materials and velocity separation between sensor motion and human motion — are what made the test clean. Neither is accidental.

## What the test isn't

This is the standalone pipeline. No ego-motion compensation, no background model, no micro-doppler classifier. The Phase 5, 6, and 7 work is built and wired into the ROS 2 safety node — those posts cover what each layer does and why. But for a hardware-in-the-loop demo with a single sensor and a rotating arm, the standalone path is the right one to validate first, and it's the path the kit ships with today.

The ROS 2 node with the full pipeline running against real joint states is the next hardware milestone.

## Where this lands

The pipeline is demo-ready. The sensor works mounted on a moving arm. The zone transitions are clean and legible to someone who's never seen the system before — which is the bar a demo video has to clear to be useful.

Join the waitlist at [dntddynamics.com/store](https://dntddynamics.com/store) to be notified when ordering opens.

---

*Build logs from DNTD Dynamics. The approach is here; the production tuning ships with the kit. Questions about the engineering are welcome — [contact@dntddynamics.com](mailto:contact@dntddynamics.com).*
