---
title: "TLV frames live on the Jetson — wrong firmware was the whole problem"
date: 2026-05-03
tags: [mmwave, jetson, debug]
summary: "Two days chasing a sensor that accepted every config command, then emitted 16 bytes of 0xFF. Root cause: ISK firmware on AOP hardware. The fix was one file."
showToc: true
---

Two days. Every config command returning `Done`. `queryDemoStatus` reporting Sensor State 1. Data port emitting exactly 16 bytes of `0xFF` and nothing else.

The 16 bytes is an ackPing — the firmware's way of saying "I heard you start the sensor." It's not TLV data. The sensor was never actually streaming.

## The root cause

`xwr68xx_mmw_demo.bin` is the ISK variant of the mmW demo firmware. ISK = Industrial Single-chip — a different board with a discrete linear antenna array. The IWR6843AOP has a completely different antenna geometry: a 3TX/4RX patch array in an L-configuration, all on-chip.

When you flash ISK firmware onto an AOP board, the internal angle-of-arrival tables don't match the physical antenna positions. The sensor boots, accepts config, starts — and then the DSP can't reconcile the geometry, so the data port produces nothing useful.

TI doesn't make this obvious in the SDK directory structure. The `xwr68xx/mmw/` folder contains ISK firmware. The AOP firmware lives in the OOB demo folder: `xwr68xx/out_of_box_6843_aop/`.

## The fix

Flash `out_of_box_6843_aop.bin` (401KB) via UniFlash in SOP2 mode. The ISK binary is 538KB — if you're looking at that file, you have the wrong one.

After reflashing and sending the corrected config:

```
Sensor State: 2
Data port baud rate: 921600
```

124,960 bytes captured in ~15 seconds. Magic word at byte 0:

```
02 01 04 03 06 05 08 07
```

First decoded point: `(+0.04, +0.61, +0.24)` at 16.0 dB SNR — consistent with the desk edge ~60 cm in front of the sensor.

## What's next

The OOB and mmW demo firmwares accept the same CLI command set and emit TLV frames with the same magic word. For SDK 3.5.x, the OOB binary is the only one TI ships with AOP support — there is no AOP mmW demo variant.

Next up: building `uart_reader.py` for continuous dual-UART reads, then wiring point clouds into `zone_logic.py` for live `STOP` / `CAUTION` / `CLEAR` output on the arm.
