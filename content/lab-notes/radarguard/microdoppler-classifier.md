---
title: "Micro-doppler classifier — not everything that moves is a person"
date: 2026-05-20
tags: [mmwave, ros2, robotics, safety, classifier]
summary: "This part adds a layer that tells a walking person apart from a box on a cart before the detection reaches the zone logic — so the system stops crying wolf at harmless objects, without ever being able to miss a real person."
showToc: true
---

The background model tells the safety node what's always there. What it doesn't know is what *kind* of thing just appeared where something wasn't before.

A person walking through the workspace and a box being pushed across it look similar at the zone level: both are new, both have range, both can cross a zone boundary. Before both triggered CAUTION and STOP. That's technically safe — but it's also wrong, and a safety system that cries wolf too often is a safety system that gets switched off. Nuisance stops are not a cosmetic problem. They're how a good system ends up disabled on the floor.

This adds a discrimination layer between background filtering and zone classification. New returns get grouped, and each group is labeled as a person, an object, or unknown. Only the ones that could be a person reach the zone logic.

## The one rule that makes it safe

The classifier is allowed to do exactly one thing: suppress things it's confident are *not* people. It can never suppress a person, and anything it can't confidently label is treated as a person and passed straight through. The system is never less safe with the classifier on than with it off — the worst case is that it occasionally stops for an object, which is exactly the baseline behavior. The only thing the classifier can change is *reducing* false alarms, never increasing missed detections.

This is the design principle for the whole stack, and it's worth stating plainly because it's the part that matters: every layer added to this pipeline is built so that if it fails, it fails toward stopping, not toward missing. Cleverness that can make a safety system less safe isn't worth having. Cleverness that can only make it less annoying is.

## Why radar can tell the difference at all

The reason this is even possible without a camera is that a walking human and a rigid moving object don't look the same to radar, even when they're the same size and moving at the same speed. A person isn't one rigid reflector — different parts move differently, and the motion isn't symmetric the way a box sliding in one direction is. Those differences are visible in the returns. That's the signal the classifier leans on. The specific features and tuning that turn that signal into a reliable label are the part that took the time to get right, and they're what ships dialed-in with the kit.

## It gets better with use

The rule-based version ships to everyone and works out of the box. Separately, every confident object-suppression is logged locally on the device, building a labeled record of real workspace activity over time. Down the line that record becomes the basis for an optional trained upgrade for operators who've run enough hours to generate meaningful data. The logged data never leaves the device and isn't part of the public repo.

## Where it sits in the pipeline

New returns are grouped, labeled, and only the person-or-unknown ones continue to the zone logic, which still makes the actual CLEAR / CAUTION / STOP call. The discrimination layer just filters what the zone logic is allowed to see. The safety decision stays where it was; the noise reaching it goes down.

---

*Build logs from DNTD Dynamics. The approach is here; the production tuning ships with the kit. Questions about the engineering are welcome — [contact@dntddynamics.com](mailto:contact@dntddynamics.com).*
