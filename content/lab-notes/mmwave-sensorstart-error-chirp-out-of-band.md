---
title: "sensorStart Error -1 — the chirp was running outside the 60–64 GHz band"
date: 2026-04-29
tags: [mmwave, config, regulatory]
summary: "All 12 calibrations passed. Sensor State 1. Then nothing. The chirp engine silently refused to run because the end frequency exceeded the regulatory band by 750 MHz."
showToc: true
---

All 12 boot-time calibrations passed. The debug output confirmed it:

```
Debug: Init Calibration Status = 0x1ffe
```

`0x1FFE` means bits 1–12 are all set — every RF calibration succeeded. The sensor advanced to State 1. Then `sensorStart` returned `Error -1` and the data port stayed silent.

State 1 means configured but not running. The error was happening in the chirp engine, after calibration, before any data could flow.

## The math I didn't check

The regulatory band for the IWR6843AOP is 60.0–64.0 GHz.

My `profileCfg` had:

```
profileCfg 0 60.75 30 7 57.14 0 0 70 1 256 5209 0 0 30
```

Bandwidth = `freqSlopeConst × rampEndTime` = 70 MHz/μs × 57.14 μs = **4.0 GHz**.

End frequency = 60.75 + 4.0 = **64.75 GHz**.

That's 750 MHz past the top of the band. The calibrations run at boot and don't care — they test the RF chain independently of the chirp profile. But the chirp engine validates the sweep boundaries before starting, and it refused.

No error message explaining this. Just `Error -1`.

## The fix

Change `startFreq` from `60.75` to `60.0` so the chirp ends exactly at 64.0 GHz. Sensor advanced to State 2 in under a second.

```
profileCfg 0 60.0 30 7 57.14 0 0 70 1 256 5209 0 0 30
```

## The check to run before every config change

```
endFreq = startFreq + (freqSlopeConst_MHz_us * rampEndTime_us / 1000)
```

For the IWR6843AOP: `endFreq` must be ≤ 64.0 GHz. If it's over, sensorStart will silently fail with `Error -1` every time, regardless of calibration status.

The TI E2E forums have multiple threads about this exact error. The answer is buried in replies. Saving you the search.
