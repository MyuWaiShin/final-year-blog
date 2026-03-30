---
title: "Software Architecture: How the Robot, Gripper, and Sensors Are Controlled"
date: 2026-03-01 12:00:00 +0000
categories: [Project Updates, Software]
tags: [ur10, rg2, urcap, multithreading, rtde, dashboard, ports, architecture]
author: myuwaishin
pin: false
last_modified_at: false
---

This post documents the full software architecture used to control the robot arm and gripper. Before building the complete pipeline, I wanted to record and specify exactly how each part of the system communicates, why certain approaches failed, and how the current architecture is structured.

---

## Hardware Setup


<div style="display:flex; justify-content:center; align-items:flex-end; gap:1.5rem; flex-wrap:wrap; margin:1.5rem 0;">
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/UR10.webp" alt="UR10 CB-Series" style="height:200px; width:auto; object-fit:contain;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>UR10 CB-Series</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/RG2.webp" alt="OnRobot RG2" style="height:200px; width:auto; object-fit:contain;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>OnRobot RG2</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/oak_d_lite.webp" alt="OAK-D Lite" style="height:200px; width:auto; object-fit:contain;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>OAK-D Lite</em></figcaption>
  </figure>
</div>

- UR10 CB-Series, a six degree-of-freedom arm
- OnRobot RG2, an electrically actuated parallel-jaw gripper mounted on the flange
- OAK-D Lite, a stereo depth camera mounted eye-in-hand
- Laptop running Python, the only external computer in the loop

The laptop connects to the robot over a local network. All communication is over TCP sockets.

---

## Part 1: Moving the Arm — RTDE

For arm motion I use the `ur_rtde` library, which communicates over the RTDE protocol on port 30004. RTDE stands for Real-Time Data Exchange and is UR's high-frequency structured communication interface.

It handles all timing and data formatting internally. Calling `rtde_control.moveL()` or `rtde_control.moveJ()` sends motion commands at the right rate and blocks until the move completes. Joint positions, TCP pose, and speed are read back through `rtde_receive`.

```python
# arm motion with ur_rtde
rtde_control.moveL([x, y, z, rx, ry, rz], speed=0.1, acceleration=0.5)
```

For arm motion, RTDE is the right tool. It is stable, well-maintained, and handles the real-time timing requirements automatically.

The problem is it cannot control the RG2 gripper.

---

## Part 2: The URCap Wall — Why `rg_grip()` Does Nothing

My first approach for gripper control was to send a URScript command directly over port 30002. URScript is the robot's own scripting language and the RG2 documentation mentions a function `rg_grip()`. So I tried:

```python
# 00_test_grip_urscript.py — the approach that FAILED
sock = socket.socket()
sock.connect((ROBOT_IP, 30002))
sock.sendall(b"rg_grip(10, 0)\n")
```

The gripper did not move. I then wrapped it in a proper URScript function block:

```python
prog = (
    "def test_grip():\n"
    "  rg_grip(10, 0)\n"
    "end\n"
    "test_grip()\n"
)
sock.sendall(prog.encode())
```

Still nothing. The robot received the script, executed it, and `rg_grip()` silently did nothing. Received no error and no movement.

The reason: the OnRobot RG2 integrates through a URCap, a plugin installed on the teach pendant that extends the robot's execution environment. The function `rg_grip()` only exists inside that extended environment, which is only active when a teach pendant `.urp` program is running.

Connecting to port 30002 and sending raw URScript talks to the robot's base interpreter. The URCap is not loaded there. `rg_grip` is completely unknown to that environment.


The diagnostic script [`00_test_grip_urscript.py`](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/pipeline_dev/scripts/00_test_grip_urscript.py) was written to confirm this.

It outputs:

```text
NO     Q1: rg_grip one-liner (port 30002)
NO     Q2: rg_grip in def block (port 30002)
rg_grip is NOT available in port 30002 URScript (URCap restriction).
```

---

## Part 3: The Workaround — Dashboard Server (Port 29999)

Since `rg_grip()` only works inside `.urp` programs, I pre-loaded two short programs on the robot's teach pendant:

- `grip_close.urp` — a single `rg_grip(0, 20)` call to close the gripper.
- `grip_close_loop.urp` — contains `rg_grip(0, 20)` call in a loop, re-closing every 0.1 seconds at 20N force. The loop means the gripper keeps re-applying force if an object slides.
- `grip_open.urp` — a single `rg_grip(100, 20)` call to fully open the jaws.

These are triggered remotely via the Dashboard Server on port 29999, which accepts simple text commands to manage the pendant's program execution:

```python
# grip_control_with_slip_detection.py — the approach that WORKED
def _dashboard(self, cmd: str) -> str:
    s = socket.socket()
    s.connect((self.ip, 29999))
    s.recv(1024)                        # consume the welcome banner
    s.sendall((cmd + "\n").encode())
    response = s.recv(1024).decode()
    s.close()
    return response

# to close the gripper:
self._dashboard("stop")
self._dashboard("load /programs/myu/grip_close.urp")
self._dashboard("play")
```

This is what [`00_test_urp.py`](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/pipeline_dev/scripts/00_test_urp.py) was built to verify, and it worked immediately. The gripper closed, force was applied, and jaw width feedback started changing.

---

## Part 4: Reading Sensor Data — Port 30002 as a State Stream

With the gripper closing, I needed to know when it had made contact and how wide the jaws were. The RG2 exposes two feedback signals through the robot's tool flange I/O:

- `AI2` (Analog Input 2), a 0 to 3.7V voltage proportional to jaw width. 0V is closed, 3.7V is roughly 110mm open.
- `DI8` (Digital Input 8), a single bit that goes HIGH when the gripper jaws hit something (force limit reached).

These values are embedded in the real-time state packets that the robot streams continuously over port 30002. Every few milliseconds, the robot sends a binary packet containing nested sub-packets. Inside the Tool Data sub-packet (type 2) we can find both `AI2` and `DI8`.

Reading them requires parsing the binary format manually:

```python
def _parse_packet(self, pkt: bytes):
    offset = 5
    while offset + 5 <= len(pkt):
        sp_len  = struct.unpack("!I", pkt[offset:offset+4])[0]
        sp_type = pkt[offset+4]
        if sp_type == 2 and sp_len >= 15:   # Tool Data sub-packet
            # AI2 voltage is an 8-byte double at byte 7
            self.latest_analog_in2 = struct.unpack("!d", pkt[offset+7:offset+15])[0]
            # DI8 is packed into the last byte
            self.latest_digital_in = pkt[offset + sp_len - 1]
        offset += sp_len
```

Jaw width in millimetres is then calculated using a two-point linear calibration:

```python
def get_width_mm(self) -> float:
    voltage = max(self.latest_analog_in2, 0.0)
    raw_mm  = (voltage / 3.7) * 110.0
    slope   = (91.0 - 10.5) / (65.8 - 8.5)   # ~1.405
    offset  = 10.5 - (8.5 * slope)            # ~-1.44
    return round(raw_mm * slope + offset, 1)
```

Calibration was done by measuring the actual jaw separation with a calliper at two known positions, fully closed and holding a known-width object, and fitting a linear correction.

---

## Part 5: Why Multithreading is Necessary

Port 30002 streams state data continuously and indefinitely. The robot sends packets whether you ask for them or not. If nothing is reading from the socket, the OS receive buffer fills up and the connection eventually stalls.

At the same time:

- The main thread is moving the arm via `rtde_control.moveL()`
- Arm motion takes several seconds
- During that time, the slip monitor needs to keep watching the jaw width
- If the gripper drops the object mid-transit, the system needs to know immediately

If everything ran sequentially in a single loop, reading a state packet would block the arm from moving, and sending a dashboard command would interrupt the state stream. The system would miss critical events.

The solution is to give each job its own thread:

```python
# Thread 1: reads port 30002 continuously, updates shared sensor variables
self._recv_thread = threading.Thread(target=self._update_loop, daemon=True)
self._recv_thread.start()

# Thread 2: watches those values, detects slip events
self._monitor_thread = threading.Thread(target=self._slip_monitor, daemon=True)
self._monitor_thread.start()
```

Both threads are marked `daemon=True`, so they shut down automatically when the main program exits.

A `threading.Lock()` on the command socket ensures that if two parts of the code try to send URScript at the same time, they queue correctly instead of corrupting each other's data:

```python
self._cmd_lock = threading.Lock()

def send_urscript(self, script: str):
    with self._cmd_lock:
        self._cmd_sock.sendall((script.strip() + "\n").encode())
```

---

## Part 6: Full Architecture

Putting it all together, the pipeline uses three communication channels simultaneously:

```
Laptop (Python)
│
├── ur_rtde (RTDE, port 30004)
│     └── moveL / moveJ / get_actual_tcp_pose
│         Used for: all arm motion, joint state readback
│
├── Dashboard Server (port 29999)  ← one-shot connections, on demand
│     └── stop / load .urp / play
│         Used for: triggering gripper open and close via URCap programs
│
└── Secondary Client (port 30002)  ← persistent connection, always open
      ├── READ  → binary state packets (AI2 + DI8)
      │           Used for: jaw width, contact detection, slip monitoring
      └── WRITE → raw URScript (movel, movej)
                  Used for: lightweight motion commands
```

---

## Summary

What looked like a single unified API turned out to be three separate protocols solving three different problems.

| Channel | Port | Job |
|---------|------|-----|
| RTDE via `ur_rtde` | 30004 | Arm motion and joint state readback |
| Dashboard Server | 29999 | Triggering URCap gripper programs |
| Secondary Client | 30002 | Continuous sensor state stream (width and contact) |

Multithreading is what makes all three work at the same time. Without it, the slip monitor would go blind every time the arm moved, and the arm would stall every time the gripper was checked.
