---
title: "ForeForce"
description: "mmWave safety perception kits for robot arms and mobile robots."
url: /workshop/foreforce/
layout: page
showToc: false
---

**ForeForce by DNTD Dynamics — mmWave safety perception for robot arms and mobile robots.**

Purpose-built sensing kits with focused integration, source-available software (BSL 1.1), and documentation written by someone who actually debugged the hardware.

[← Back to the Workshop](/workshop/)

---

## Industrial — ForeForce Sensing Kit

<span class="badge badge-dev">Waitlist open — ships Q4 2026</span>

For fixed installations: robot arms, machine cells, workstations.

The sensor your robot uses when it can't see.

No camera. No lidar. Built on 60–64 GHz FMCW radar-on-chip (TI IWR6843AOP), the kit senses through dust, darkness, and direct occlusion, outputs a 3D point cloud over USB at 10 Hz, and connects to ROS2 in under an hour with the included driver package.

Developed and tested on a Jetson Orin Nano Super. Ships with a working, tuned configuration for arm-mounted collision detection — not a sanitized example from an application note.

<div class="preorder-box">

### Board — $449

<div class="preorder-price">$99 refundable deposit</div>
<p class="preorder-note">Balance of $350 invoiced at ship · Deposit refundable anytime · Ships Q4 2026 · Development kit — for design professionals and B2B use</p>

Ordering opens soon. Join the waitlist and you'll be first in line when it does.

{{< email-signup label="Join the waitlist for the Industrial kit" button="Join waitlist" >}}

</div>

Commercial deployments also require a [ForeForce commercial software license](/store/) — one per deployed system. Read, run, learn, and non-commercial use are free under BSL 1.1.

### What's included

- DNTD-designed mmWave sensing board (custom PCB built on the TI IWR6843AOP)
- ROS2 driver package (Python, tested on JetPack 6.2.2)
- Working configuration tuned for arm-mounted collision detection
- Zone detection library with `STOP` / `CAUTION` / `CLEAR` output
- Getting started guide — written for builders, not datasheets
- Repository access for all kit owners

{{< figure src="/images/workshop/workshop-evm.jpg" alt="mmWave development board" >}}

### Specs

<table class="spec-table">
  <tr><td>Frequency</td><td>60–64 GHz (FMCW)</td></tr>
  <tr><td>Range</td><td>0.1 – 8.9 m</td></tr>
  <tr><td>Field of view</td><td>±90° azimuth / ±40° elevation</td></tr>
  <tr><td>Output</td><td>x/y/z point cloud at 10 Hz via USB</td></tr>
  <tr><td>Interface</td><td>Dual UART over USB</td></tr>
  <tr><td>Power</td><td>5V USB · ~1.2W typical</td></tr>
  <tr><td>Radar SDK</td><td>TI mmWave SDK 3.5.x</td></tr>
  <tr><td>Tested on</td><td>Jetson Orin Nano Super, JetPack 6.2.2</td></tr>
  <tr><td>Ships as</td><td>Development kit — development and evaluation use only</td></tr>
</table>

<p class="dev-kit-notice">Sold as a development kit for evaluation and design by professionals. Not an FCC-authorized end product. Buyers integrating ForeForce into a deployed system are responsible for conducting a risk assessment and validating conformity per applicable standards (including ISO 10218 / ISO/TS 15066, ANSI/RIA R15.06) in their complete system.</p>

{{< figure src="/images/workshop/workshop-evm-working.jpg" alt="mmWave point cloud output" >}}

### Why no camera, why not lidar

Camera-based collision detection fails in direct sunlight, low light, dust, and any time the obstacle is behind something else in frame. Lidar is expensive, mechanically complex, and has its own occlusion problems on manipulator arms in motion.

mmWave radar has no moving parts, works in complete darkness, and doesn't care about dust or spray. The tradeoff is resolution — you get a sparse point cloud, not a photo. For collision detection and presence sensing on a robot arm, that's exactly the tradeoff you want.

{{< figure src="/images/workshop/workshop-jetson.jpg" alt="Jetson Orin development setup" >}}

---

## Mobile Robotics — ForeForce Mobile Kit

<span class="badge badge-soon">Coming soon — join the list</span>

For battery-powered platforms: AMRs, mobile robots, anything where power budget matters.

Lower-power variant built on the TI IWRL6432AOP. Same ROS2 integration, same documentation standard, optimized for standalone and mobile platforms.

Pricing and availability will be announced to the list first.

{{< email-signup label="Notify me when the Mobile kit is available" button="Notify me" >}}
