---
title: "Risk Assessment II — Updated Project Risk Register"
date: 2026-02-12 10:00:00 +0000
categories: [Project Updates, Planning]
tags: [risk-assessment, safety, project-management, ur10, lab-safety]
author: myuwaishin
pin: false
last_modified_at: false
---

After the gripper experiments last week, I revised the [initial risk assessment]({% post_url 2026-02-05-risk-assessment %}) and filled out the formal university risk register template to cover everything more precisely. I went through each activity properly, assigned scores, and documented the actual control measures I have in place.

The completed form is available here:

📄 **[Download: PDE3823 Project Risk Assessment (Excel)]({{ '/assets/files/PDE3823 Project Risk Assessment.xlsx' | relative_url }})**

---

## What's in the Form

The register covers nine hazards across four areas:

- **Physical safety** — robot arm collision, gripper jaw contact during manual placement, gripper overheating, dropped/ejected objects, unexpected arm sweep paths, and camera mounting during eye-in-hand configuration
- **Software & data** — code/model data loss, and YOLO misclassification sending incorrect poses to the robot
- **Project delivery** — recovery logic implementation complexity vs. submission deadline

Each entry includes an initial likelihood × severity risk score, current control measures, and a post-control residual score.

---

## Contingency Plans

The contingency plans (A/B/C) remain unchanged from the [initial assessment]({% post_url 2026-02-05-risk-assessment %}). Plan B is the minimum viable deliverable. Better to have a solid, well-documented failure detector than a half-working recovery system.

| Plan | Scope | When to Activate |
|---|---|---|
| **Plan A** | Full failure detection + vision-guided pose estimation + recovery system | On schedule — all milestones met |
| **Plan B** | Failure detection + basic retry and re-grasp recovery | Milestone slip ≤ 2 weeks |
| **Plan C** | Failure detection only + documented analysis of recovery strategies | Major delays — demonstrate detection pipeline works end-to-end |

---


*This risk register will be updated when relevant as the project progresses.*
