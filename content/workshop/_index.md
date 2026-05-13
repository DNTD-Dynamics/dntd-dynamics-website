---
title: "Ohm & Iron Workshop"
description: "Hardware sensing kits for builders, researchers, and robotics developers."
url: /workshop/
layout: page
showToc: false
---

Hardware for builders who need to sense the world.

Purpose-built sensing kits with full ROS2 integration, open-source firmware, and documentation written by someone who actually debugged the hardware.

---

## mmWave 360° Sensing Kit — IWR6843AOP

<span class="badge badge-dev">Pre-order open — ships Q3 2026</span>

The sensor your robot uses when it can't see.

No camera. No lidar. The IWR6843AOP is a 60–64 GHz FMCW radar on a chip — it sees through dust, darkness, and direct occlusion, outputs a 3D point cloud over USB at 10 Hz, and connects to ROS2 in under an hour with the included driver package.

Built and tested on a Jetson Orin Nano Super. The config file included with the kit is the one actually running on the bench — not a sanitized example from a TI application note.

<div class="preorder-box">

### Pre-order — $279

<div class="preorder-price">$279</div>
<p class="preorder-note">Free domestic shipping · Evaluation board for development use · Charge on order · Ships Q3 2026</p>

**[→ Pre-order on the store page](/store/)**

</div>

### What's included

- IWR6843AOPEVM evaluation module
- ROS2 driver package (Python, tested on JetPack 6.2.2)
- Working config file — tuned for arm-mounted collision detection
- Zone detection library with `STOP` / `CAUTION` / `CLEAR` output
- Getting started guide — written for builders, not datasheets
- Private repository access for all kit owners

{{< figure src="/images/workshop/workshop-evm.jpg" alt="IWR6843AOP EVM evaluation module" >}}

### Specs

<table class="spec-table">
  <tr><td>Frequency</td><td>60–64 GHz (FMCW)</td></tr>
  <tr><td>Range</td><td>0.1 – 8.9 m</td></tr>
  <tr><td>Field of view</td><td>±90° azimuth / ±40° elevation</td></tr>
  <tr><td>Output</td><td>x/y/z point cloud at 10 Hz via USB</td></tr>
  <tr><td>Interface</td><td>Dual UART over USB (Silicon Labs CP2105)</td></tr>
  <tr><td>Power</td><td>5V USB · ~1.2W typical</td></tr>
  <tr><td>Firmware</td><td>out_of_box_6843_aop — SDK 3.5.x / fw 3.6.0.0</td></tr>
  <tr><td>Tested on</td><td>Jetson Orin Nano Super, JetPack 6.2.2</td></tr>
  <tr><td>Ships as</td><td>Evaluation board — development use only</td></tr>
</table>

{{< figure src="/images/workshop/workshop-evm-working.jpg" alt="IWR6843AOP mmWave output" >}}

### Why no camera, why not lidar

Camera-based collision detection fails in direct sunlight, low light, dust, and any time the obstacle is behind something else in frame. Lidar is expensive, mechanically complex, and has its own occlusion problems on manipulator arms in motion.

mmWave radar has no moving parts, costs under $200, works in complete darkness, and doesn't care about dust or spray. The tradeoff is resolution — you get a sparse point cloud, not a photo. For collision detection and presence sensing on a robot arm, that's exactly the tradeoff you want.

{{< figure src="/images/workshop/workshop-jetson.jpg" alt="Jetson Orin development setup" >}}

---

## IWRL6432AOP Mobile Kit — Coming Soon

<span class="badge badge-soon">Out of stock — notify me</span>

Lower-power mobile variant of the kit. Same ROS2 integration, same documentation standard, optimized for legged robots and mobile platforms where power budget matters.

Currently out of stock at DigiKey. Join the list to be notified when available at **$199**.

{{< email-signup label="Notify me when the 6432 kit is available" button="Notify me" >}}
