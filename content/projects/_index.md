---
title: "Projects"
description: "Everything active at DNTD Dynamics — hardware, software, and research."
url: /projects/
layout: page
showToc: false
---

Everything on the bench right now.

---

## 01 — mmWave Sensing Kit <span class="badge badge-active">Active</span>

360° collision detection and presence sensing for robot arms. IWR6843AOP on Jetson Orin Nano Super. TLV point cloud pipeline live at 10 Hz. Zone detection integrating now. Pre-order launching Q3 2026.

Hardware goal: three-sensor array covering the full arm sweep volume. Current phase: single-sensor bringup, building toward the demo video.

[→ Workshop page](/workshop/) · [→ Lab notes tagged mmwave](/tags/mmwave/)

---

## 02 — Sterling AI <span class="badge badge-active">Active</span>

Local AI runtime for the homelab and eventually the sensing platform. Mistral 7B via Ollama, ChromaDB RAG for long-term memory, Home Assistant integration, hybrid intent router. SQLite for action logging.

Running on NucBox K6 (Ryzen 7 7840HS, 32GB DDR5). Wake word detection is the next major feature — always-running Python process before porting to ESP32 per room.

Long-term role: the OS layer bundled with the sensing middleware. Sterling manages sensor pipelines, logs experimental data, runs inference, handles automation. Privacy-first, no cloud dependency required.

[→ Lab notes tagged sterling](/tags/sterling/)

---

## 03 — Project Lumina <span class="badge badge-poc">POC</span>

Holographic refractive display. Architecture: DLP projector → 50mm borosilicate glass sphere → frosted acrylic dome → SLM layer for multi-view steering.

Parts in hand: 50mm borosilicate sphere, Canon Rayo S1 pico projector. Python ray-trace simulator built. Phase 1 POC in progress — single DLP beneath the sphere, frosted acrylic dome, ~$150 total.

North star: the compact holographic display from the film *65*. Phase 2 adds SLM layer and dual DLPs entering the sphere from the sides. Phase 1 must prove before Phase 2 begins.

[→ Lab notes tagged lumina](/tags/lumina/)

---

## 04 — MyceliumVision <span class="badge badge-dev">Dev</span>

ML contamination detection for mushroom cultivation. PyTorch + OpenCV on NucBox K6. SQLite for logging. Primary subject: Lion's Mane (*Hericium erinaceus*).

Motivating application for the genetics research thread — contamination detection in cultivation generates the biological observations that quantum chemistry simulation eventually investigates at the molecular level.

[→ Lab notes tagged mycology](/tags/mycology/)

---

## 05 — Gardening Robot Arm <span class="badge badge-dev">Dev</span>

6-DOF arm on Raspberry Pi 5 with ROS2 and ESP32 motor control. Rosbridge dashboard confirmed end-to-end — joint commands flowing from browser to motors.

Primary role now: mmWave test platform and reference implementation for the sensing kit. The demo video for the pre-order launch will be shot on this arm.

[→ Lab notes tagged robotics](/tags/robotics/)

---

## 06 — Piezo / Frequency Analyzer <span class="badge badge-dev">Dev</span>

Plant biorhythm sensing using piezo vibration and INA828 + OPA1679 signal chain. Arduino FFT firmware, browser-based spectrum visualizer. Bridges the environmental sensing and biosystems verticals.

---

## 07 — Genetics / PySCF <span class="badge badge-soon">Research</span>

Quantum chemistry simulation for Lion's Mane neurogenic compounds — hericenones and erinacines. PySCF 2.12.1 running on NucBox K6.

Activated once biological questions from the mycology and genetics work are specific enough to simulate meaningfully. The pipeline: biology generates observations → genetics investigates mechanisms → quantum simulation models molecules from first principles.

---

## Standalone incubations

**BetweenSystems** — Unity 6 Android space trading game. Separate audience, own timeline. Incubated independently.

**DNTD Dynamics website** — you're looking at it.
