---
title: "Eye-in-Hand Calibration II: ChArUco Board"
date: 2026-03-02 12:00:00 +0000
categories: [Project Updates, Calibration]
tags: [charuco, hand-eye calibration, camera, tcp, reprojection error, tsai-lenz]
author: myuwaishin
pin: false
last_modified_at: false
---

The camera mount has not arrived yet, so the camera position relative to the gripper is still temporary. But calibration can still be improved before the mount is fixed. This session switched to a ChArUco board calibration method and compared five hand-eye solvers to see which one gives the most reliable result.

---

## What is a ChArUco Board?

A standard checkerboard gives us corner points, but only if the full board is visible and unobstructed. A ChArUco board embeds ArUco markers inside the squares of the checkerboard. This gives two things at once:

- **Checkerboard corners** for sub-pixel accuracy
- **ArUco marker IDs** for robust identification even when part of the board is occluded

The valuable difference from a plain checkerboard is localisation precision with partial visibility. Each ArUco marker has a unique ID, so even if only part of the board is in frame, the solver knows exactly which corners it is seeing and where they sit in the full grid. A plain checkerboard cannot do this if part of it is hidden, we lose all corner correspondence.

For eye-in-hand calibration with my setup, this matters because the gripper often partially occludes the board as the robot moves through poses.

---

## How Calibration Solves the Camera-to-TCP Transform

The goal is to find the rigid transform `T_cam2tcp`, which describes how the camera sits relative to the robot's Tool Centre Point (TCP). Once known, any point the camera sees in its own frame can be converted into the robot's coordinate system and used for motion planning.

At each pose during data collection:

1. The robot reports its current **TCP pose in the robot base frame** (`T_tcp2base`), read from RTDE.
2. The camera detects the ChArUco board and estimates the **board pose in the camera frame** (`T_board2cam`) using PnP solving.

After collecting N poses, the solver receives:
- A list of end-effector transforms `{A_i}` — how the robot moved between poses
- A list of camera transforms `{B_i}` — how the board appeared to move between poses

It then solves for `X` in the equation `AX = XB`, where `X` is the unknown camera-to-TCP transform.

---

## What the ChArUco Board Parameters Mean

The solver does not need to know which specific board was printed, but it does need the right dimensions. If any of these are wrong, the calibration will be wrong.

> Find the calibration solver file at [Calibration/eye_in_hand/solve_calibration.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/Calibration/eye_in_hand/solve_calibration.py).
{: .prompt-info }

  <figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/charuco_board_7x5_dict6x6.png" alt="ChArUco Board" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>ChArUco Board</em></figcaption>
</figure>

| Parameter | What it controls |
|-----------|-----------------|
| `square_m` | Physical square size in metres. Used to build the 3D object points (Wrong size = all distance estimates scale wrong). |
| `cols, rows` | Grid dimensions. Tells the solver which corners exist and their layout. |
| `marker_m` | ArUco marker size. Affects PnP estimation. |
| `aruco_dict` | Dictionary used to generate the board (e.g. `DICT_6X6_250`). Must match exactly (wrong dictionary means no markers detected). |

The marker IDs themselves do not matter as long as the dictionary matches. Each detected ID maps to a fixed position in the grid, so the solver always knows exactly which corner it is looking at.

**What breaks if parameters are wrong:**

| Wrong parameter | Effect |
|----------------|--------|
| `square_m` too large or small | All depth estimates scale incorrectly, bad translation in `T_cam2tcp` |
| Wrong `cols / rows` | Corner indices mismatch |
| Wrong ArUco dictionary | No markers detected |
| `marker_m` too large or small | Detection works but PnP estimates are less stable |

---

## Board Consistency

Board consistency is how stable the detected board pose is across frames at different positions from the robot base frame when the robot base and board are both stationary. A consistent board means the corner detection is reliable and repeatable because both the board and robot base are not moving.

Low consistency (high pixel variance in reprojected corners across frames) means the image quality or board detection is noisy, and poses collected from that view will add error into the solver.

Monitoring board consistency before accepting a pose gives the solver cleaner input data. For this session, the consistency value is shown on screen. The target is to only accept poses where the board is stable.

---

## Reprojection Error

Reprojection error measures how well the solved transform matches the observed data. After solving for `T_cam2tcp`, the solver takes the known 3D board corners, projects them back into the image using the estimated camera matrix and transform, and measures the pixel distance between where they land and where they were actually detected.

A low reprojection error means the transform is internally consistent with the data. A high error means the poses or the board detections were noisy.

It does not guarantee the calibration is physically correct, it only tells you how well the solver fitted the data it was given.

---

## Results: Five Solver Comparison

Five hand-eye calibration solvers were tested on the same dataset. The visualisation displays a coordinate frame showing the estimated TCP location and orientation overlaid on the live camera view. The Z axis should point straight out of the TCP (vertically upward from the gripper face) and the origin should sit at the midpoint between the two fingers.

The five images below show the result of each solver:

<div style="display:flex; justify-content:center; align-items:flex-start; gap:1rem; flex-wrap:wrap; margin:1.5rem 0;">
  <figure style="margin:0; text-align:center; width:45%;">
    <img src="/assets/img/calib_tsai.png" alt="TSAI method" style="width:100%;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>[1/5] TSAI</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center; width:45%;">
    <img src="/assets/img/calib_park.png" alt="PARK method" style="width:100%;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>[2/5] PARK</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center; width:45%;">
    <img src="/assets/img/calib_horaud.png" alt="HORAUD method" style="width:100%;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>[3/5] HORAUD</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center; width:45%;">
    <img src="/assets/img/calib_andreff.png" alt="ANDREFF method" style="width:100%;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>[4/5] ANDREFF</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center; width:45%;">
    <img src="/assets/img/calib_daniilidis.png" alt="DANIILIDIS method" style="width:100%;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>[5/5] DANIILIDIS</em></figcaption>
  </figure>
</div>

DANIILIDIS [5/5] gave the closest result, the Z axis is most vertical and the origin closest to the finger midpoint. However, all five still show a slight offset from the true TCP.

The previous approach averaged across all five solvers to produce a "mean" result. This was the wrong call. Averaging across solvers that produce different quality estimates does not improve accuracy, it dilutes the best result with worse ones and introduces systematic error.

---

## Replay Calibration

Before running a new calibration session, the existing data collection process was replayed to inspect the quality of the captured poses. The video below shows the replay.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/9eIAboRm5fE"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Replay of calibration data collection</em></figcaption>
</figure>

The replay reveals a clear problem: the wrist orientation is not varied enough across collected poses. Throughout the sequence, the robot's wrist remains at nearly the same angle. Because the hand-eye solver solves `AX = XB` using the relative motion between poses, it needs diverse rotations to constrain the rotation component of `X`. When all poses share the same wrist orientation, the rotational input to the solver is nearly degenerate, the rotation matrices `{A_i}` are all similar, giving the solver almost only a few useful signal to separate the camera's rotational offset from the TCP. The result is a poorly conditioned rotation estimate, which is exactly why the calibrated axis directions are still visibly wrong even after the solve.

This is one of the primary reasons the calibration is inaccurate, and it must be corrected before the next session.

---

## What Needs to Change

The issues are:

- The TCP origin is visibly offset from the midpoint between the fingers
- The Z axis is not perfectly vertical out of the TCP face
- The mean of multiple solvers adds error instead of reducing it
- Wrist orientation is not varied enough across poses, making the solver's rotation estimate degenerate

For the next calibration session:

1. **Use Tsai-Lenz only.** Tsai-Lenz is the most rigorously validated closed-form AX = XB solver and remains the standard reference in robotics calibration literature. Unlike the other solvers tested, it decouples the rotation and translation solves, which makes it more numerically stable when the dataset has limited diversity. For an eye-in-hand setup with constrained motion like this one, it consistently outperforms stochastic or iterative alternatives. Running multiple solvers and averaging them served no purpose, it only dilutes the best estimate with worse ones.
2. **Require at least 15 poses.** The current dataset was too small for a reliable solve.
3. **Require at least 6 ArUco markers detected per frame.** Frames with fewer detected markers produce noisy PnP estimates and should be rejected before the pose is accepted.
4. **Monitor board consistency live.** Only accept poses where the board detection is stable across consecutive frames.
5. **Collect poses with more varied wrist orientations.** Degenerate pose sets (all similar angles) cause the solver to underfit the rotation component. The wrist must be rotated significantly between poses to give the solver enough rotational signal to constrain `X`.

These changes will go into the calibration script before the next session.
