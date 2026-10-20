# Gesture-Controlled-Robotic-Arm-using-Inverse-Kinematics
Developed a gesture-controlled 6-DOF robotic arm that enables intuitive, real-time human-robot interaction using computer vision and inverse kinematics. (MediaPipe and OpenCV integration)
A software-driven system that controls a **6-DOF robotic arm with hand gestures**. A webcam and MediaPipe track the user's hand, the gestures are turned into joint commands, and a 3D simulation of the arm in the browser shows the motion in real time. The architecture is modular, so the same commands can later drive a physical arm.

> **Course project:** VIT (Vellore Institute of Technology)

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Existing Methods and Limitations](#existing-methods-and-limitations)
- [System Architecture](#system-architecture)
- [Gesture Controls](#gesture-controls)
- [6-DOF Arm Model](#6-dof-arm-model)
- [Running the Simulator](#running-the-simulator)
- [Hardware Plan](#hardware-plan)
- [Results](#results)
- [Novelty](#novelty)
- [Limitations and Future Work](#limitations-and-future-work)
- [Repository Contents](#repository-contents)
- [Team](#team)
- [References](#references)
- [License](#license)

## Overview

Robotic arms are usually controlled with joysticks, buttons, teach pendants, pre-programmed paths, or marker-based tracking. These need physical controllers or fixed programming, and a human is needed whenever something changes.

This project replaces them with **computer vision**. A webcam captures the user's hand, MediaPipe extracts 21 hand landmarks, and the system maps the gestures to joint angles of a 6-DOF arm. The report also describes a Damped Least Squares (DLS) inverse kinematics solver with Jacobian-based control for stable motion near singular configurations, and a finite state machine for task management. A 3D simulation shows the arm moving in real time, so the control logic can be tested without hardware.

## Objectives

- Design a gesture-based control system
- Implement inverse kinematics
- Develop a modular robotics architecture
- Simulate robotic arm motion
- Enable intuitive interaction

## Existing Methods and Limitations

| Existing method | Limitation |
|---|---|
| Joysticks, buttons, teach pendants | Not real-time or adaptive; rely on pre-programmed inputs; constant human intervention; hard to trace errors |
| Marker-based tracking (e.g., ArUco tags) | Needs physical markers and high-spec cameras; sensitive to lighting |
| Basic inverse kinematics | Unstable near singularities |

Overall limitations: lack of intuitive control, hardware dependency, instability in basic IK, and limited integration.

## System Architecture

```
Gesture → Hand Tracking → Target Mapping → IK Solver → Joint Angles → Output
```

The implementation is split into five layers that communicate over a WebSocket:

| Layer | What it does |
|---|---|
| **Input** | Webcam via OpenCV (800 × 600, 30 fps) |
| **MediaPipe** | BGR→RGB, flip, HandLandmarker (`hand_landmarker.task`), async callback returning 21 landmarks with z-depth |
| **Gesture processing** (`gesture_server.py`) | Palm orientation (cross product of wrist→MCP vectors), finger extension (tip.z vs MCP.z, threshold 0.02; thumb on x-axis), gesture classifier |
| **Signal smoothing** | Exponential moving averages: tilt (α = 0.25, dead zone ±0.08), angle (α = 0.30, wrap-around guard ±90°), joint output (α = 0.35, six smoothers clamped to joint limits) |
| **WebSocket transport** | Python `websockets` + `asyncio` server at `ws://localhost:8765`, broadcasting JSON `{type, theta[6], gesture, joint, reset}` to all clients |
| **Browser** | `robotic_arm_6dof.html` with an auto-reconnecting WebSocket client, a forward-kinematics engine (DH parameters), and a Three.js renderer |
| **Output** | 6-DOF arm, trajectory trail, joint readouts (HUD), end-effector position |

## Gesture Controls

From the block diagram:

| Gesture | Action |
|---|---|
| 1 to 5 fingers extended | Select joint J1 to J5 |
| Fist | Select joint J6 |
| Palm orientation / tilt | Sets the angle of the selected joint |
| Two hands | Reset |

## 6-DOF Arm Model

Denavit–Hartenberg parameters used by the simulator (lengths in model units):

| Joint | Name | a | d | α | Limits (°) |
|---|---|---|---|---|---|
| 1 | Base | 0 | 230 | 90° | ±170 |
| 2 | Shoulder | 250 | 0 | 0° | ±90 |
| 3 | Elbow | 220 | 0 | 0° | ±135 |
| 4 | F-Roll | 0 | 180 | 90° | ±180 |
| 5 | Wrist | 0 | 0 | −90° | ±90 |
| 6 | EE-Yaw | 0 | 80 | 0° | ±180 |

Home pose: `[0, −20, 60, 0, 30, 0]` degrees.

## Running the Simulator

**File:** [`robotic_arm_6dof.html`](robotic_arm_6dof.html). It is a single page using Three.js r128.

1. Open the file in **Chrome or Edge**. No installation is needed.
2. Make sure you're online, since Three.js and the fonts load from a CDN.
3. Drag to orbit and scroll to zoom.

**Keyboard controls:**

| Keys | Action |
|---|---|
| `Q` / `A` | Joint 1 (Base) + / − |
| `W` / `S` | Joint 2 (Shoulder) + / − |
| `E` / `D` | Joint 3 (Elbow) + / − |
| `R` / `F` | Joint 4 (F-Roll) + / − |
| `T` / `G` | Joint 5 (Wrist) + / − |
| `Y` / `H` | Joint 6 (EE-Yaw) + / − |
| `+` / `−` | Change step size (1°, 5°, 10°) |
| `Space` | Reset to home pose |

### Gesture control

> **TODO:** add the Python gesture server and its setup instructions here.

The gesture side needs a webcam, Python, and these libraries:

```bash
pip install opencv-python mediapipe websockets
python gesture_server.py
```

It also needs the MediaPipe `hand_landmarker.task` model file. The server then listens on `ws://localhost:8765` and the browser page connects to it.

## Hardware Plan

Full hardware integration was not completed. The software and simulation are the working parts; the hardware below is the planned extension.

| Component | Specification | Purpose |
|---|---|---|
| Robotic arm chassis | 6 DOF | Main body |
| ESP32 | 5 V + Wi-Fi | Joint control and real-time arm programming |
| Servo motors | MG995S / MG996R, about 15 kg·cm | Joint rotation |
| Webcam | Laptop camera or ESP32-CAM | Gesture detection |
| Power supply | Battery bank or SMPS (5 V, 5 to 10 A) | Powering 5 to 6 servos without drawing from the microcontroller |

## Results

- Real-time hand tracking and gesture recognition worked, and the arm responded immediately to input.
- The IK model showed stable convergence to the desired positions.
- The simulation produced smooth motion with minimal jerk.

<!-- Add screenshots or a demo GIF: -->
<!-- ![Simulator](docs/images/simulator.png) -->

## Novelty

- Gesture control integrated with inverse kinematics
- Stable DLS approach, reliable near singular configurations
- Modular design, with vision, logic and visualization as separate layers

## Limitations and Future Work

- Hardware integration is not complete.
- Gesture recognition depends on camera quality and visibility.
- No dataset was used.
- Planned: drive physical servos through an ESP32, add gesture sequences with a finite state machine, and explore patent filing or publication.

## Repository Contents

```
.
├── README.md
├── robotic_arm_6dof.html        # 3D simulator (Three.js)
├── gesture_server.py            # Gesture recognition + WebSocket server (add)
├── hand_landmarker.task         # MediaPipe model (add or download)
├── docs/
│   ├── Gesture-Controlled_Robotic_Arm_using_Inverse_Kinematics.pdf
│   ├── robotic_arm_report.docx
│   ├── PPT.pptx
│   └── images/
└── LICENSE
```

Links: [Simulator](robotic_arm_6dof.html) · [Project report (PDF)](docs/Gesture-Controlled_Robotic_Arm_using_Inverse_Kinematics.pdf) · [Presentation](docs/PPT.pptx)

## Team

| Name | Registration No. |
|---|---|
| Ritvik Anish | 23BEE1018 |
| Aneesh Manu | 23BEL1022 |

## References

- Siciliano, B., *Robotics: Modelling, Planning and Control*
- Craig, J. J., *Introduction to Robotics*
- Buss, S. R., *Introduction to Inverse Kinematics*
- MediaPipe Hands documentation
- Angelidis and Bampis (2025), gesture-controlled robotic arm for small assembly lines
- Qi et al. (2023), review of vision-based hand gesture recognition for human–robot interaction
- Paterson and Aldabbagh (2021), vision-controlled robotic arm using OpenCV
