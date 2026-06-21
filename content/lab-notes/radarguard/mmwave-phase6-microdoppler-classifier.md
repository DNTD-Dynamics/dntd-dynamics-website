---
title: "Micro-doppler classifier — not everything that moves is a person"
date: 2026-05-21
tags: [mmwave, ros2, robotics, safety, classifier]
summary: "Phase 6 adds a DBSCAN cluster builder and a rule-based micro-doppler classifier to the pipeline. Novel returns are now labeled PERSON, OBJECT, or UNKNOWN before they reach the zone logic. UNKNOWN passes through as PERSON — the classifier can never make the system less safe than the baseline."
showToc: true
---

The background model tells the safety node what's always there. What it doesn't know is what kind of thing just appeared in a novel voxel.

A person walking through the workspace and a box being pushed across it look similar at the zone classifier level: both are novel, both have range, both can cross zone boundaries. Before Phase 6, both would trigger CAUTION and STOP. That's technically safe, but it's also wrong — and a system that cries wolf too often gets disabled.

Phase 6 adds a layer between background filtering and zone classification: cluster the novel returns, extract features, classify each cluster as PERSON, OBJECT, or UNKNOWN, and only pass PERSON detections into the zone logic. The classifier can suppress false alarms. It cannot suppress genuine threats.

## Cluster builder

`cluster.py` runs DBSCAN on the novel point cloud each frame. Each cluster becomes a `Cluster` object with extracted features:

- **centroid** — spatial center of mass
- **range** — distance from sensor origin
- **point_count** — number of returns in cluster
- **velocity_spread** — range of doppler velocities across the cluster
- **height_span** — vertical extent of the cluster
- **doppler_asymmetry** — imbalance between positive and negative velocity components
- **snr_mean** — average signal quality across returns
- **temporal_variance** — how much the centroid has moved across the last N frames

Clusters persist across frames via centroid matching. A cluster that was at position X last frame and is at position X+delta this frame is the same cluster — it gets a stable ID and an accumulating history.

## Rule-based classifier

`classifier.py` scores each cluster using four feature votes. The default threshold requires 2 of 4 votes for a PERSON label. Everything below threshold is OBJECT. Everything that doesn't fit either cleanly is UNKNOWN.

**Vote 1 — velocity spread.** Humans have mixed doppler: arms, torso, and legs all move differently. A person walking produces a wide spread of velocities across the cluster. A moving box produces a narrow, coherent velocity signature. High spread → PERSON vote.

**Vote 2 — height span.** A person is tall. A box is not. Returns from a person span significant vertical range even at the resolution of the IWR6843AOP. Low vertical span → OBJECT vote.

**Vote 3 — point count.** A person is a large, complex reflector. But this vote is gated — it only fires when a corroborating feature (velocity spread or height span) also supports PERSON. Alone, point count is too ambiguous.

**Vote 4 — doppler asymmetry.** Human gait is asymmetric: the leading foot approaches, the trailing foot recedes. Returns from a walking person will have both positive and negative doppler simultaneously. A rigid object moving in one direction won't. Asymmetry → PERSON vote.

All thresholds are YAML-tunable. The defaults are conservative — they're designed to minimize false negatives (missed people) rather than false positives (spurious stops). Tune them from real workspace data.

## Fail-safe by design

UNKNOWN is not a third safe state. The classifier has three possible outputs:

- **PERSON** — passes to zone logic, can trigger CAUTION/STOP
- **OBJECT** — suppressed, logged, never reaches zone logic
- **UNKNOWN** — passes through as PERSON

If the classifier sees something it can't confidently label, the detection passes through. The pipeline is never less safe than the baseline without the classifier. The only thing the classifier can do to zone behavior is suppress OBJECT-labeled returns.

```yaml
# Bypass classifier entirely — restores Phase 3 behavior instantly
classifier_enabled: false
```

## Training data pipeline

Every OBJECT suppression gets logged to `~/mmwave/logs/classifier_training.csv`:

```
timestamp, centroid_x, centroid_y, centroid_z, range_m, point_count,
velocity_spread, height_span, doppler_asymmetry, snr_mean, temporal_variance,
label
```

The label field is `OBJECT` — these are suppressed detections the classifier was confident about. Every hour of operation with a real arm in a real workspace builds a labeled dataset automatically.

When Phase 6b arrives — a sklearn RandomForest or lightweight tflite drop-in — this CSV becomes the fine-tuning dataset. The rule-based classifier ships to everyone. The trained model becomes an optional upgrade for those who've run enough hours to generate meaningful data.

`logs/` is gitignored. Training data stays on the Jetson.

## What changed in the pipeline

```
Novel points → cluster (DBSCAN) → classifier (PERSON/OBJECT/UNKNOWN)
    → OBJECT suppressed + logged
    → PERSON/UNKNOWN → zone logic → CLEAR/CAUTION/STOP
```

The architecture between background model and zone logic now has a deliberate decision point. The zone classifier still makes the safety call. The micro-doppler classifier just filters what it sees.
