# CONCURSOS DE ROBÓTICA DE LA UCLM

# WRO 2026 Future Engineers - Autonomous Vehicle Documentation - I.E.S. Santa María de Alarcos (Ciudad Real, Spain)

This repository contains the official documentation, source code, and mechanical design overview of our autonomous vehicle engineered for the **Future Engineers** category of the World Robot Olympiad (WRO). 

Our engineering approach focuses on a lightweight mechanical design, an interference-free hardware architecture, and a dynamic Proportional-Derivative (PD) control algorithm written in TypeScript via the MakeCode platform.

---

## Table of Contents

1. [Mechanical Design and Mobility](#1-mechanical-design-and-mobility)
2. [Power Architecture, Electronics, and Sensors](#2-power-architecture-electronics-and-sensors)
3. [Software Architecture and Strategy](#3-software-architecture-and-strategy)
4. [Systemic Thinking and Engineering Decisions](#4-systemic-thinking-and-engineering-decisions)

---

## 1. Mechanical Design and Mobility

The chassis is built upon a custom-cut plywood board. This material choice provides structural simplicity, a low overall weight, and high agility on the track. Furthermore, the modular nature of the design allows for rapid physical repairs in the event of hardware failure or collisions during a race.

* **Rear-Wheel Drive Transmission:** We utilize a continuous rotation servo coupled to the rear axle via two custom-designed 19-tooth (modulus 2) gears. These gears were modeled in Tinkercad and 3D printed. This direct-gear transmission solved early traction issues, eliminating the wheel slip experienced in our initial friction-based prototypes.
* **Front Steering Mechanism:** The steering system is based on **Ackermann steering geometry**. It operates using a standard positional servo connected via plastic tie rods, nuts, and bolts to two movable steering knuckles with independent axles for each front wheel.
* **Supplementary Materials:** Plastic sheets for steering linkages, structural silicone, zip ties, and standard mounting hardware.

---
All pieces have been cut with a SculpFun S30 with a 5W diode laser at 100% power and a cutting speed of 80mm/min and air assistance. 
Folder V4

|         RIGHT         |           FRONT            |             LEFT            |          REAR             |
| --------------------- | -------------------------- | --------------------------- | ------------------------- |
| <img width="696" height="313" alt="image" src="https://github.com/user-attachments/assets/36aa9d9f-ce6f-4e2a-96e9-37297832e079" /> | <img width="694" height="308" alt="image" src="https://github.com/user-attachments/assets/c9345c40-2c37-4908-aa45-18e16d6e4385" /> | <img width="713" height="337" alt="image" src="https://github.com/user-attachments/assets/f50e7488-1b95-419f-be6b-38a48f1f42dd" /> | <img width="693" height="305" alt="image" src="https://github.com/user-attachments/assets/6660fb8d-c9b7-47fd-a4cf-a276fb64ff4c" /> |

|                       |            UP              |                             |                           |
| --------------------- | -------------------------- | --------------------------- | ------------------------- |
| <img width="861" height="587" alt="image" src="https://github.com/user-attachments/assets/591e3c8a-b745-43c4-b298-b927579b61e6" /> | <img width="701" height="674" alt="image" src="https://github.com/user-attachments/assets/f91d8bd7-84f2-43ce-a6fd-f03cd3563ed3" /> | <img width="834" height="600" alt="image" src="https://github.com/user-attachments/assets/10d917b0-e0a8-4d45-93bf-9031306664d5" /> | <img width="792" height="493" alt="image" src="https://github.com/user-attachments/assets/3e2dad86-ad0c-467b-990c-d5be0958f331" /> |

## 2. Power Architecture, Electronics, and Sensors

The central processing unit of the robot is a **BBC micro:bit** mounted on a Microlog expansion board. The entire system, including the microcontroller, motors, and sensors, is powered by an independent AA and AAA battery bank or alternatively a power bank to ensure stable current draw during motor acceleration.

For environmental perception, the vehicle utilizes:
* **Ultrasonic Sensors (x2):** Mounted on the left and right sides to measure wall distances (HC-SR04 module).
* **HuskyLens AI Vision Sensor:** Communicating via I2C to handle real-time color and object recognition (pillars and parking zones).

### Optimized Hardware Pinout
During hardware integration, we identified severe signal interference. The micro:bit's internal LED matrix and the onboard buttons share physical pins with external GPIOs. Using these shared pins for sensors caused "ghost starts" (unprompted hardware interrupts) and erratic ultrasonic readings. We re-routed the hardware to guarantee zero conflicts:

| Component | Assigned Pin | Engineering Notes |
| :--- | :--- | :--- |
| Traction Motor (Servo) | P0 | Primary speed control |
| Steering Motor (Servo) | P2 | Angle control (Ackermann linkage) |
| Right Ultrasonic (Echo) | P1 | Re-routed to avoid P5 (Button A interference) |
| Right Ultrasonic (Trig) | P16 | Dedicated trigger line |
| Left Ultrasonic (Echo) | P8 | Completely isolated from internal LED matrix |
| Left Ultrasonic (Trig) | P12 | Completely isolated from internal LED matrix |
| HuskyLens Camera | P19 / P20 | Standard I2C communication lines |

---

## 3. Software Architecture and Strategy

The main execution loop is developed in **JavaScript/TypeScript** using MakeCode. Text-based programming was selected over block-based programming due to its superior handling of arrays (used for median filtering of sensor data), better error control, and the ability to maintain a clean, linear, and highly readable codebase.

### State Machine Overview
The vehicle's behavior is organized as a finite state machine with four modes, traversed sequentially during a run. The flowchart below shows the full control flow, from the safe start sequence to the final parking maneuver. Most "No" branches return to the sensor-reading step, which is the beginning of each iteration of the main loop.

```mermaid
flowchart TD
    A([Power on]) --> B[Init HuskyLens and compass]
    B --> C{Button A<br/>pressed?}
    C -->|No| C
    C -->|Yes| D[Smooth motor ramp-up]
    D --> E[/Read sensors:<br/>2 sonars, camera, compass/]

    E --> M1[Mode 1: center between walls<br/>until first corner]
    M1 --> Q1{Corner detected?<br/>wall lost or rotation}
    Q1 -->|No| E
    Q1 -->|Yes| S[Deduce direction<br/>from turn sign: CW or CCW]

    S --> M2[Mode 2: drive laps<br/>PD centering control]
    M2 --> Q2{Pillar visible<br/>and large?}
    Q2 -->|Red| ER[Steer right to avoid]
    Q2 -->|Green| EV[Steer left to avoid]
    Q2 -->|No| CNT[Count corners]
    ER --> CNT
    EV --> CNT
    CNT --> Q3{12 corners<br/>completed?}
    Q3 -->|No| E
    Q3 -->|Yes| Q4{Obstacle<br/>round?}

    Q4 -->|Yes| P0[Mode 3: hug outer wall<br/>and search for magenta]
    P0 --> Q5{Magenta<br/>detected?}
    Q5 -->|No| Q6{Search<br/>timeout?}
    Q6 -->|No| P0
    Q6 -->|Yes| FIN
    Q5 -->|Yes| P1[Phase 1: pull nose in]
    P1 --> P2[Phase 2: straighten parallel]
    P2 --> FIN

    Q4 -->|No| OP[Mode 3 Open:<br/>roll forward and stop]
    OP --> FIN

    FIN([Mode 4: stop motor<br/>and show target icon])
```

### Autonomous Pre-Start Logic
Upon powering the expansion board and pressing Button A, the vehicle does not move immediately. It executes a pre-flight environmental scan:
* **Track Direction Detection:** The robot measures the initial distance to the left and right walls. Based on the asymmetrical starting dimensions of the WRO track, the microcontroller mathematically deduces whether it must run the circuit clockwise or counter-clockwise.
* **Challenge Mode Detection:** The HuskyLens scans the immediate environment. If it detects active specific color IDs (1: Magenta, 2: Green, 3: Red), the software automatically enters *Obstacle Challenge* mode. If no colors are detected, it defaults to the *Open Challenge* mode.

### Centering Algorithm: PD Controller
In high-speed racing robotics, the "I" (Integral) component of a standard PID controller accumulates past errors, often causing delayed reactions and subsequent wall collisions (a phenomenon known as *Integral Windup*). Therefore, we implemented a highly tuned **Proportional-Derivative (PD) controller**.

* **Error Calculation:** The system determines its offset from the track's center by calculating the difference between the right and left ultrasonic sensors (`error3 = dist_der - dist_izq - offset_centro`).
* **Proportional (P):** The raw steering force. The further the robot is from the center line, the sharper the steering angle applied (`KP = 1.4`).
* **Derivative (D):** The dampening force. To prevent infinite oscillation (zig-zagging), the derivative calculates the rate of approach to the center. It counter-steers slightly before reaching the exact center to ensure smooth stabilization (`KD = 0.9`).

The final mathematical correction sent to the steering servo is constrained by mechanical safety limits:
`correccion = Math.constrain(error3 * KP + derivativo * KD, 0 - CORR_MAX, CORR_MAX)`

### Navigation Strategies
* **Cornering:** The algorithm detects a corner when the ultrasonic sensors report a sudden loss of the outer wall (distance exceeds a set threshold) AND the micro:bit's internal magnetometer registers an accumulated heading rotation exceeding a specific angular threshold.
* **Obstacle Avoidance:** The HuskyLens continuously reports the ID and the bounding box width of detected colors. If the bounding box of a red or green pillar exceeds a predetermined size (indicating proximity), the software temporarily injects an `offset_centro` value into the PD controller, forcing the robot to fluidly shift to the safe lane.
* **Braking and Parking:** After completing the required laps, the system actively searches for the Magenta color (ID 1). Once the bounding box width confirms it is within range, the vehicle triggers a sequence to steer into the parking zone and cuts power to the traction motor.
* **Trajectory Correction:** Sensor data is read, filtered through a mathematical median array to discard anomalous noise, and processed by the PD controller seamlessly at 50Hz.

---

## 4. Systemic Thinking and Engineering Decisions

The development of this autonomous vehicle required a systemic approach to balance technical requirements against real-world constraints:

* **Time and Budget Constraints:** The team was limited to 5 working hours per week and utilized standard materials provided by our institution. We adapted our mechanical design to maximize the structural integrity of readily available components (plywood, silicone).
* **Traction Dynamics:** Initial tests on the WRO tatami mat were unsuccessful due to poor speed and agility caused by direct-axle wheel slip. *Solution:* We adopted an iterative design process, upgrading to a custom 3D-printed gear transmission system to increase torque and grip.
* **Sensor Interference and Hardware Bugs:** We experienced critical failures where the car would start on its own or fail to read distances. *Solution:* Through deep hardware documentation research, we mapped the micro:bit's internal architecture and migrated the ultrasonic pins away from the shared LED matrix and Button A lines.
* **Machine Vision Calibration:** Reliable color recognition in motion proved highly unstable due to changing lighting conditions. *Solution:* We dedicated specific tuning sessions solely to train the HuskyLens algorithm under various angles, distances, and lighting environments to achieve a stable bounding-box read.
