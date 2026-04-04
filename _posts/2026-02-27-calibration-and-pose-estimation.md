---
title: "Pose Estimation and Eye-in-Hand Calibration I"
date: 2026-02-27 12:00:00 +0000
categories: [Project Updates, Perception]
tags: [calibration, pose-estimation, yolo, oak-d, ur10, robotics]
author: myuwaishin
pin: false
last_modified_at: false
---

I am now doing eye-in-hand calibration and pose estimation of the target and then transferring frames from the camera to the robot. Successfully moving the TCP to the target is part of a simple retry recovery and having this means I will have examples of real life failures that I can use for my failure detection. 

When we say recovery, failure detection is part of the recovery because without knowing that it has failed it cannot recover, and a simple retry is the most fundamental recovery.

I am now trying to build a fully autonomous pick and place pipeline, after getting all the small but necessary pieces done to get to this stage including training YOLO detection, adjusting detection, and depth testing.

## The Goal
The goal is simple. The robot needs to look at the workspace, find an object, and pick it up autonomously. The camera tells the robot where to go.

To make that work, three things need to line up perfectly. 
1. Camera intrinsics to map 3D world points to 2D pixels.
2. Object pose estimation to turn a 2D detection into a 3D physical position.
3. Eye in hand calibration to translate where the camera sees the object into where the robot actually needs to move.

This post covers the first attempt at all three, including what worked and what didn't, and what the code does.

## 1. Camera Intrinsics
Before I can do any 3D math with camera images I need the intrinsic matrix. This is a set of four numbers that describe the geometry of that specific camera lens. 

```
K matrix

K = |fx   0   cx |
    |0    fy  cy |
    |0    0   1  |
```

* `fx`, `fy` is the focal length in pixels, which is how zoomed in the lens is horizontally and vertically
* `cx`, `cy` is the principal point, which is the pixel coordinate of the optical centre usually near the image centre

From the camera's perspective, I see a pixel at u, v and I know the depth Z, so where is that point in physical 3D space?

```python
X = (u - cx) * Z / fx     # metres left/right
Y = (v - cy) * Z / fy     # metres up/down
Z = depth                 # metres away from camera
```

This process is called back-projection. It goes from a 2D pixel plus depth back to 3D space.

The OAK-D Lite stores factory-calibrated intrinsics in its internal flash memory, so we read them directly from the device.

> Find the calibration file at [Calibration/eye_in_hand/handeye_calibration.json](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/Calibration/eye_in_hand/handeye_calibration.json).
{: .prompt-info }

```python
# From pose_estimation.py — load_intrinsics()
calib   = device.readCalibration()
M, w, h = calib.getDefaultIntrinsics(dai.CameraBoardSocket.RGB)

# M = [[fx, 0, cx], [0, fy, cy], [0, 0, 1]]
# The realistic values from handeye_calibration.json look like this:
self.fx  = M[0][0]   # 1452.6 px
self.fy  = M[1][1]   # 1454.1 px
self.cx0 = M[0][2]   # 626.7 px
self.cy0 = M[1][2]   # 309.0 px
```

The camera handles all the stereo matching internally and gives us a depth map aligned to the RGB frame.

## 2. Object Pose Estimation
The detection pipeline runs a fine-tuned YOLOv8n model converted to ONNX (for CPU inference) on the OAK-D RGB frames. The camera pipeline is configured using the DepthAI library (depthai or dai).

> Find the pipeline at [pose_estimation.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/Pick_Target/pose_estimation.py).
{: .prompt-info }   

```python
# From pose_estimation.py — create_pipeline()
cam = pipeline.create(dai.node.ColorCamera)
cam.setPreviewSize(640, 480)
cam.setFps(30)

mono_l = pipeline.create(dai.node.MonoCamera)
mono_r = pipeline.create(dai.node.MonoCamera)

mono_l.setBoardSocket(dai.CameraBoardSocket.LEFT)  # Explicitly assigns the left lens
mono_r.setBoardSocket(dai.CameraBoardSocket.RIGHT) # Explicitly assigns the right lens

stereo  = pipeline.create(dai.node.StereoDepth)
stereo.setDefaultProfilePreset(dai.node.StereoDepth.PresetMode.HIGH_DENSITY)
stereo.setDepthAlign(dai.CameraBoardSocket.RGB)  # depth pixels aligned to RGB
```

Setting depth align to RGB `setDepthAlign(RGB)` is the critical line. It warps the depth map so that pixel u, v in the depth frame corresponds to the exact same 3D point as pixel u, v in the RGB frame. Without this the depth and colour would be offset because the cameras are physically separated by around 75mm.

### YOLO Detection
YOLO outputs axis aligned bounding boxes. For each detection we compute the centre pixel and look up the depth there.

```python
# From pose_estimation.py — PoseEstimator.compute()

# Sample a 5-pixel radius patch for robustness against noise
r = 5
patch = depth_frame[py-r:py+r+1, px-r:px+r+1]
good = patch[patch > 0]               # ignore invalid zeros
Z_mm_raw = float(np.median(good))     # median to reject noise spikes

# Temporal smoothing across last 8 frames
self._z_hist[class_id].append(Z_mm_raw)
Z_mm = float(np.median(self._z_hist[class_id]))
Z = Z_mm / 1000.0                     # mm → metres
```

Two smoothing stages are applied.

1. **Spatial smoothing** uses a median over a 5px radius patch instead of a single pixel. Infrared stereo depth sensors often return zeros which are invalid pixels due to reflection or no texture, or sharp outlier spikes. Taking the median of all valid pixels in an 11x11 square (which means a 5px radius) around the centre ensures we get a solid and representative depth reading even if the exact centre pixel failed to calculate depth.

2. **Temporal smoothing** uses a rolling median over the last 8 frames which kills flickering from intermittent depth returns on shiny or dark surfaces.

### Deploying 'OBB' Oriented Bounding Box

Knowing X, Y, and Z tells the robot where to go but not how to orient the gripper. For elongated objects like cylinders, picking at the wrong angle means the gripper collides with the object instead of gripping it cleanly.

The `OrientedBoxAnalyzer.analyze()` method extracts the object mask and fits a minimum area rotated rectangle to find its orientation.

```python
# From pose_estimation.py — OrientedBoxAnalyzer.analyze()

# 1. Colour segmentation — find the dominant hue
hsv  = cv2.cvtColor(crop, cv2.COLOR_BGR2HSV)
mask = self._dominant_hue_mask(hsv)

# 2. Find the largest contour
contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
largest = max(contours, key=cv2.contourArea)

# 3. Fit minimum-area rotated rectangle
rect = cv2.minAreaRect(largest)
(rcx, rcy), (rw, rh), angle = rect

# 4. Normalise: always treat the long axis as width
if rw < rh:
    angle += 90
    rw, rh = rh, rw

gripper_angle = angle % 180   # degrees, 0 = horizontal
```

OpenCV `cv2.minAreaRect()` returns the smallest rectangle of any angle that fully contains the contour. The angle returned is the rotation angle in degrees of the rectangle. However, OpenCV can sometimes arbitrarily swap what it calls the width and height of the rectangle depending on the angle. To fix this we normalize it. If the returned width `rw` is shorter than the height `rh`, we add 90 degrees to the angle and swap the dimensions. This guarantees that the angle always represents the orientation of the long axis of the object. A gripper angle of 0 means the object is lying horizontally across the image, and 90 means it's vertical. The gripper will then match this angle so its fingers approach the long sides of the object safely.

### Putting the 3D Pose Together

With both of these calculations complete, I have the final 3D object pose estimation.

```python
# Back-project pixel to 3D
X = (pixel_cx - cx0) * Z / fx
Y = (pixel_cy - cy0) * Z / fy
```

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/pose_estmation_cube.png" alt="Pose Estimation Cube" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Pose Estimation of a Detected Cube with OBB</em></figcaption>
</figure>

## 3. Hand Eye Calibration: Translating Camera Space to Robot Space
The camera knows where the object is, but how does the robot know where to move? The camera sees the object at `X_cam, Y_cam, Z_cam` in its coordinate frame. The robot needs the object's position in its own base frame to move towards it, which is a completely different coordinate system anchored at the robot's base.

My camera is mounted on the robot's wrist which is an eye-in-hand configuration.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/cam_to_robot_flowchart.png" alt="Eye-in-Hand: Camera space to Robot space Flowchart" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Eye-in-Hand: Camera space to Robot space Flowchart created with Miro (<a href="https://miro.com/welcomeonboard/UnozSVE5VzNicThFaXB6REpmb2hDNXcxMXBvOSt0WWtNNUdWNks4T2w3blhiT3hwTmFkc1I5V0MwVE1kVmdyNDFKVXh6SlVpUU9NMUs0d2hEQVV4WmswM1JPTjBYdDZqbEpsS01OTTZwejdmcm5TWTJYeWUwTTBRRGFCSEwrZER0R2lncW1vRmFBVnlLcVJzTmdFdlNRPT0hdjE=?share_link_id=106095975114">link</a>)</em></figcaption>
</figure>

This chain allows the robot to translate high-level visual data into low-level motor commands.

**Why do we calculate everything relative to the robot's base frame?**
The UR10 kinematics solver, using movej or movel, plans point to point trajectories based on absolute coordinates, i.e. the robot's base frame. If we tell the robot to move 5cm forward relative to the TCP, the mathematical definition of forward changes constantly as the wrist joint rotates during the movement.

By translating the camera's detection immediately into the static Base Frame, we give the robot a fixed XYZ coordinate in the real world. This provides a stable and mathematically absolute target for the trajectory planner to reach regardless of the arm's starting pose or how the joints turn during the journey.

### Camera to TCP calibration procedure
The approach used was a 3 point centre and touch method.

>Find the calibration script here: [calibrate_handeye_3pt.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/Pick_Target/calibrate_handeye_3pt.py)
{: .prompt-info }   

The procedure is as follows:

1. Move the robot so a known object appears centred at a crosshair in the camera view.
2. Records the robot TCP pose at that moment which is the "detect" pose. It also records the X, Y, Z depth of the object from the camera frame.
3. Physically jog the robot gripper tip downwards until it touches the centre of the same object.
4. Records the new TCP pose which is the "touch" pose.

The difference between the two TCP poses gives the exact position of the object relative to the robot base frame. Combined with the camera's 3D observation of the object using the X, Y, Z from the prior step, we get a direct camera observation to TCP frame transformation.

This is repeated for 3 objects at different positions, and then solved for the rotation and translation that maps camera coordinates to TCP coordinates.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/handeyecalibrate.png" alt="Hand Eye Calibration" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Hand Eye Calibration</em></figcaption>
</figure>

### Solving the transform with Procrustes SVD
With 3 pairs of corresponding points we solve for the rigid transform `R` and `t` such that:

```python
P_tcp = R @ P_cam + t
```

This equation states that any point in the camera frame `P_cam`, when rotated by matrix `R` and translated by vector `t`, will equal its position in the TCP frame `P_tcp`. We need this because the camera is strapped to the wrist at a weird diagonal angle and offset. R fixes the angle, and t fixes the offset gap.

It's solved analytically using Singular Value Decomposition, or SVD. SVD is a linear algebra tool that breaks a complex matrix down into simpler rotation and scaling matrices. Unlike machine learning algorithms that guess and check over thousands of epochs, SVD calculates the mathematically perfect least squares rotation in exactly one step.

```python
# From calibrate_handeye.py — solve_cam_to_tcp()

# For each pair, compute where the object sits in TCP frame
for p in pairs:
    R_det      = rotvec_to_matrix(*p['pose_detect'][3:])   # TCP rotation at detect time
    P_tcp_det  = np.array(p['pose_detect'][:3])
    P_tcp_touch = np.array(p['pose_touch'][:3])
    
    # Object position in TCP frame = how far the TCP moved to touch it
    P_obj_in_tcp = R_det.T @ (P_tcp_touch - P_tcp_det)

# 1. Gather all points
A = np.array(P_cam_list)   # camera observations, shape (3, 3)
B = np.array(P_tcp_list)   # TCP-frame positions,  shape (3, 3)

# 2. Centre both point clouds around the origin
A_c = A - A.mean(axis=0)   
B_c = B - B.mean(axis=0)

# 3. Calculate the Cross-Covariance Matrix (how the two point clouds relate geometrically)
H = A_c.T @ B_c            

# 4. Use SVD to extract the optimal rotation matrix between the two point clouds
U, _, Vt = np.linalg.svd(H)
R = Vt.T @ U.T             

# Guard against reflection (det must be +1)
if np.linalg.det(R) < 0:
    Vt[-1, :] *= -1
    R = Vt.T @ U.T

# 5. Calculate translation (the remaining offset once rotation is perfectly aligned)
t = B.mean(axis=0) - R @ A.mean(axis=0)
```

The result structure is `R` as a 3x3 rotation matrix and `t` as a 3D translation vector. This is directly saved to `handeye_calibration.json`. 

## Picking the Object
The script `pick_detected_object.py` does the full chain.

>Find the script here: [pick_detected_object.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/Pick_Target/pick_detected_object.py)
{: .prompt-info }   

The script takes the camera observation of the object's X, Y, Z position and the robot's TCP pose as inputs. It then applies the camera to TCP transform to calculate the object's position in the robot's base frame.

```python
# From pick_detected_object.py — cam_to_base()
P_tcp_base = np.array(tcp_pose[:3])        # where the TCP is right now in base frame
R_tcp      = rotvec_to_matrix(*tcp_pose[3:]) # TCP rotation in base frame
P_obj_cam  = np.array([X_cam, Y_cam, Z_cam]) # where camera sees the object

# Step 1: camera frame → TCP frame (fixed, from calibration)
P_obj_tcp  = R_cam_in_tcp @ P_obj_cam + t_cam_in_tcp

# Step 2: TCP frame → robot base frame (changes every frame as robot moves)
P_obj_base = R_tcp @ P_obj_tcp + P_tcp_base
```

These are two transforms that are chained. First, Camera to TCP applies the calibrated `R` and `t` which is fixed geometry and the same for all time. Second, TCP to Base applies the current robot pose which changes as the robot moves.

Once `P_obj_base` is known, the pick sequence is straightforward. Absolute base coordinates are used to command the robot.

```python
# From pick_detected_object.py — pick_object()
mover.grip_open()

# Approach from 10cm above
mover.movel(P_obj[0], P_obj[1], P_obj[2] + APPROACH_HEIGHT, rx, ry, rz)

# Descend slowly to the object
mover.movel(P_obj[0], P_obj[1], P_obj[2], rx, ry, rz, vel=0.05, acc=0.05)
mover.grip_close()

# Lift back up
mover.movel(P_obj[0], P_obj[1], P_obj[2] + APPROACH_HEIGHT, rx, ry, rz)
```

## Validating the Calibration Results
The first autonomous pick attempt didn't work. The resulting calibration calculated by SVD was inaccurate. When instructed to pick the object, the robot moved to a position noticeably offset from where the object actually was, frequently missing the object by up to 100mm.
This is definitely due to inaccurate calibration.

Here's why I speculated the calibration was wrong:

1. **The centre and touch calibration is sensitive to human error.**
   The "centring" step involves manually jogging the robot until the camera crosshair is visually aligned with the object. Even 20px of alignment error at a working depth of ~600mm translates to several centimetres of position error and I may have made a few jogging errors.
   
   ```text
   Position error = alignment_error_px × Z / fx
                  = 20px × 0.600m / 691px
                  ≈ 17mm per 20px
   ```
   
   With errors in human guided alignment, the calibration is only as accurate as the jogging patience.

2. **The camera is significantly offset and angled from the TCP.**
   The OAK-D is mounted off centre on the wrist bracket and not directly above the TCP. This means the `t_cam_in_tcp` translation vector is large, and any error in the rotation `R` gets violently amplified over the camera to TCP distance.

3. **Three points is not enough to constrain the 6DOF transform reliably.**
   3 points is the mathematical absolute bare minimum required to calculate a 3D plane using SVD. Because there is no redundancy, one slightly wrong touch measurement directly corrupts `R` and `t` with no way to detect or average the error out.

4. **Depth noise at the object surface.**
   The OAK-D's stereo depth is less accurate on small, flat, textureless objects than on large surfaces. Because the camera is mounted at an angle on the robot wrist, its Z-axis (the depth it measures) is tilted relative to the robot's base. When the rotation matrix `R` in transform chain is applied, it takes that 20mm Z-depth error and "rotates" it into the robot's X and Y coordinates. Essentially, a small error in how far away the camera thinks the object is turns into a horizontal miss when the robot tries to reach for it.

## Next Steps
The coordinate transform chain is correct and the code structure is solid. But the manual calibration process was fundamentally flawed.

Moving forward, I am dropping the Procrustes SVD solver and researching more accurate hand eye calibration methods.

---

**Scripts referenced in this post**
- `pose_estimation.py` — YOLO detection and depth back-projection and OBB
- `calibrate_handeye_3pt.py` — 3 point centre and touch calibration
- `pick_detected_objects.py` — full detect to transform to pick pipeline

You can find them all on my git here [https://github.com/MyuWaiShin/Final_Year_Project_2026](https://github.com/MyuWaiShin/Final_Year_Project_2026).
