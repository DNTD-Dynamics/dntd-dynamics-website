---
title: "IWR6843AOPEVM arrived — bringup notes and the mmWave Studio trap"
date: 2026-04-21
tags: [mmwave, hardware, setup]
summary: "EVM in hand, USB detection working immediately. mmWave Studio looked like the obvious next step. It was a trap — the AOP standalone board can't connect. UniFlash is the right tool."
showToc: true
---

The IWR6843AOPEVM arrived from DigiKey. $178.80. Smaller than expected — the antenna array is the whole top face of the board, which is the point.

## USB detection — immediate

Plugged into the Jetson Orin Nano Super. Two devices enumerated right away:

```bash
$ lsusb | grep Silicon
Bus 001 Device 004: ID 10c4:ea70 Silicon Labs CP2105 Dual UART Bridge
```

```bash
$ ls /dev/ttyUSB*
/dev/ttyUSB0  /dev/ttyUSB1
```

The CP2105 is a dual UART bridge — one chip, two serial ports. On the Jetson:

- `/dev/ttyUSB0` — CLI port (115200 baud, send config commands here)
- `/dev/ttyUSB1` — Data port (921600 baud, point cloud TLV frames come out here)

User permissions: add yourself to the `dialout` group once, never think about it again:

```bash
sudo usermod -a -G dialout $USER
# log out and back in
```

## Switch configuration

The EVM has three DIP switch banks. For normal operation (not flashing):

| Switch | Setting |
|--------|---------|
| S1.1–S1.4 | Off |
| S2.1 | Off |
| S2.2–S2.4 | On |
| S3 | On |

For flashing (SOP2 mode):

| Switch | Setting |
|--------|---------|
| S1.1–S1.4 | Off |
| S2.1 | On |
| S2.2 | On |
| S2.3–S2.4 | Off |
| S3 | On |

Always power-cycle after changing switches. The SOP mode is latched at boot.

## The mmWave Studio trap

mmWave Studio is TI's GUI tool for the mmWave EVM family. It's the obvious first thing to try. Don't.

mmWave Studio requires the **MMWAVEICBOOST carrier board** for full functionality. The BOOST board has FTDI GPIO lines that Studio uses to control the SOP pins and issue hardware resets in software. The IWR6843AOPEVM standalone board doesn't have this, so Studio always fails at `Calling_ConnectTarget`.

There's also a `FTD2XX.dll` dependency that isn't included in the installer — you'll spend time fixing that first, then hit the connection failure anyway.

For the standalone AOP EVM: **use UniFlash for flashing, use Python serial for everything else.** That's it.

## What actually works

UniFlash flashing the correct AOP firmware. Python `pyserial` for sending config and reading data. That combination works completely and doesn't require any TI GUI software after the initial flash.

Next post covers the two bugs that kept the sensor from streaming after the firmware was correct.
