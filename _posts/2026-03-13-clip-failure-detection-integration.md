---
title: "Integrating CLIP into the Grasp Failure Detection Pipeline"
date: 2026-03-13 12:00:00 +0000
categories: [Project Updates, Failure Detection]
tags: [clip, failure detection, pipeline, post-grasp, binary classification]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 13 March.
{: .prompt-info }

With the CLIP classifier trained and validated, this session was spent collecting additional training data with more varied backgrounds and lift heights, then integrating the classifier into the full grasp failure detection pipeline.

---

## Additional Data Collection

The earlier dataset was collected mostly at a fixed height and against a limited set of backgrounds. Before integration I captured more HOLDING and EMPTY examples across:

- Different lift heights
- Different wrist angles
- More background variation including the blue lab floor, the table edge, and mixed clutter behind the gripper

The expanded dataset was used to retrain the probe, improving generalisation before wiring it into the live pipeline.

---

## How Failure Detection Works

The full grasp failure detection pipeline combines three data sources at three checkpoints in the pick sequence.

**Checkpoint 1 — immediately after gripper closure (before lift):**

The DI8 digital input signal and the jaw width are read. If the jaw width suggests the gripper closed on nothing (fully closed or close to it), and the DI8 confirms no object detected, the grasp is classified as a missed grasp before the robot even lifts.

**Checkpoint 2 — after the 200 mm lift:**

The CLIP classifier runs on a cropped ROI from the live camera frame. The result is either HOLDING, EMPTY, or UNCERTAIN:

- **HOLDING** (`p ≥ 0.85`) → continue to transit
- **EMPTY** (`p ≥ 0.85`) → trigger recovery sequence
- **UNCERTAIN** (`p < 0.85`) → return to scan position, no recovery triggered

**Checkpoint 3 — continuous during transit:**

The slip monitor polls jaw width every 0.3 seconds. If it detects a significant width drop mid-transit (suggesting the object has slipped), it triggers a CLIP re-verify before taking any action:

- **HOLDING** confirmed → treat width drop as a false alarm, continue transit
- **EMPTY** confirmed → trigger recovery
- **UNCERTAIN** → abort transit conservatively, return to scan

---

## Recovery Sequence

When a confirmed EMPTY result is detected at either checkpoint 2 or 3, the robot:

1. Moves to a safe intermediate position
2. Opens the gripper fully
3. Returns to the scan position above the workspace
4. The pipeline re-runs the detection and pick cycle from scratch

The recovery is only triggered on a confirmed EMPTY — not on UNCERTAIN — to avoid unnecessary interruptions when the classifier is genuinely unsure.

---

## Next

The pipeline is now wired up end to end. The next step is running full pick-and-place trials to stress-test the failure detection under realistic conditions and measure how often each checkpoint catches a failure vs produces a false positive.
