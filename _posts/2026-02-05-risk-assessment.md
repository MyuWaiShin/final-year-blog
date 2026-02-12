---
title: "Risk Assessment and Mitigation"
date: 2026-02-05 10:00:00 +0000
categories: [Project Updates, Planning]
tags: [risk-assessment, safety, project-management]
author: myuwaishin
pin: false
last_modified_at: false
---

We were asked to do a blog on risk assessment for our projects. Here's what could go wrong and how I'm planning to handle it.

## Risk Overview

| **Risk Category** | **Specific Risk** | **Impact** | **Mitigation Strategy** |
|-------------------|-------------------|------------|-------------------------|
| Timeline | Recovery logic too complex | High | Fallback: Demonstrate failure detection + basic retry recovery only |
| Scope| Trying too many failure modes | Medium | Limit to two failure methods, do comparison analysis |
| Scope | Stuck on perfect pose estimation | Medium | pre-grasp pose estimation |
| Hardware | Robot malfunction/calibration | Medium | Document baseline readings, test in sim first |
| Software | Integration failures | Medium | Test components independently, modular code |
| Software | Data/model loss | Low | Git version control, OneDrive auto-sync |
| Hardware | Sensor inconsistency | Low | Backup scripts, report issues immediately |
| Hardware | Equipment collision/damage | Low | Software boundaries, slow speeds, emergency stop ready |
| Safety | Human injury from robot | Very Low | Lightweight styrofoam objects, table-height operation |
| Safety | Dropped objects | Very Low | Styrofoam won't cause damage |

---

## Safety Considerations

**Physical risk is very low:**
- All test objects are **lightweight styrofoam** (cubes, cylinders, arcs)
- Robot operates at **safe table-level height** (not overhead)
- UR10 has **built-in collision detection**
- Working in **controlled lab environment**
- Completed safety induction

**Safety protocols:**
- Keep clear of workspace during operation
- Emergency stop always within reach
- No loose clothing/jewelry near moving parts
- Follow lab safety guidelines

---

## Contingency Plans

If things don't go as planned:

| **Plan** | **Scope** | **When to Use** |
|----------|-----------|-----------------|
| **Plan A** | Full failure detection + recovery system | Ideal timeline, everything works |
| **Plan B** | Reliable failure detection + basic retry recovery | Some delays, focus on core features |
| **Plan C** | Failure detection only + recovery discussion | Major delays, demonstrate detection works |

**Philosophy:** Better to have a solid failure detector than a half-working recovery system.

---

## Additional Notes

### Hardware Risks - Details

**Robot Malfunction:**
If the UR10 stops responding or needs recalibration, I'll have documented baseline gripper force readings to detect sensor drift. Simulation testing minimizes hardware wear.

**Equipment Damage:**
Workspace limits are enforced in software. Starting with slow speeds and only increasing after validation. Styrofoam objects won't damage the gripper even if closed too hard.

### Software Risks - Details

**Data Loss:**
All code is version-controlled with Git. Trained models auto-sync to OneDrive. This blog serves as ongoing documentation of experimental results.

**Integration Issues:**
The Camera → Robot → Gripper pipeline will be tested component-by-component. Code is kept modular for easy component swapping.

---

*This risk assessment will be updated as the project progresses.*
