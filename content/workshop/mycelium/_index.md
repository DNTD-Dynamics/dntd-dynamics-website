---
title: "MyceliumVision"
description: "ML-based contamination detection for mushroom cultivation."
url: /workshop/mycelium/
layout: page
showToc: false
---

**MyceliumVision — contamination detection for mushroom cultivation, by DNTD Dynamics.**

[← Back to the Workshop](/workshop/)

<span class="badge badge-dev">In development</span>

---

## What it is

Cultivating mushrooms at scale means catching contamination early — mold, bacterial blotch, competitor fungi — before it spreads through a grow. MyceliumVision is a computer-vision pipeline that watches cultivation jars and flags contamination visually, before it's obvious to the eye.

Lion's Mane (*Hericium erinaceus*) is the primary subject, grown and monitored on the bench at DNTD.

## Current stack

- PyTorch + OpenCV for image classification
- Running on the NucBox K6
- SQLite for observation logging

## Where this connects

Contamination events logged here are the motivating data for DNTD's genetics research thread — the biological observations MyceliumVision generates are what the PySCF quantum chemistry work eventually investigates at the molecular level (hericenones and erinacines, the neurogenic compounds Lion's Mane is known for).

## Status

Early-stage. No dataset, model, or field results published yet — this page will grow into build logs and results as the work gets there, the same way the ForeForce kit's documentation did.

{{< email-signup label="Get notified when MyceliumVision build logs go live" button="Notify me" >}}
