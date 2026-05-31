# Agriculture Drone Intelligence Platform - High-Level Design (HLD) - V1.0 (Design Phase)

## 1. Objective

Build an intelligent crop spraying platform that uses computer vision and reinforcement learning to optimize chemical usage while improving crop health. The system aims to:

- Detect crop disease from drone imagery.
- Estimate disease severity.
- Recommend an initial spray quantity.
- Learn from historical outcomes.
- Continuously improve spraying policies through reinforcement learning.

---

## 2. System Overview

The platform is organized into three layers: Edge (on-drone), Data Collection, and Cloud.

### 2.1 Edge Layer (Drone)

Responsibilities: real-time sensing, inference, and actuation.

Components:

1. RGB / multispectral camera
2. RTK GPS
3. Flow sensor
4. ArduPilot flight controller
5. Spray actuator
6. Disease detection model (Model 1)

### 2.2 Data Collection Layer

Responsibilities: store operational mission data for learning.

Per waypoint, store:

- GPS coordinate
- Timestamp
- Disease type
- Disease severity
- Spray quantity
- Image hash
- Mission ID

Storage:

- SQLite (Raspberry Pi)
- Local mission logs

### 2.3 Cloud Layer

Responsibilities: training, analytics, and deployment.

Components:

1. Spatial crop state database
2. Revisit comparator
3. RL training pipeline
4. Model registry
5. OTA deployment service

---

## 3. Model 1 - Disease Detection

Purpose: convert crop imagery into structured agricultural information.

Input:

- Leaf image

Output:

- Disease type
- Disease severity
- Initial spray quantity recommendation

Example:

```json
{
  "disease_type": "rust",
  "severity": 0.78,
  "spray_qty_ml": 15
}
```

---

## 4. Crop State Database

Purpose: maintain historical crop health information.

Stores:

- Coordinate
- Timestamp
- Disease history
- Spray history
- Health progression

Technology:

- PostgreSQL + PostGIS
- TimescaleDB

---

## 5. Revisit Comparator

Purpose: measure crop recovery after spraying.

Inputs:

- Health at time t1
- Health at time t2

Output:

$$\Delta H = H_{t2} - H_{t1}$$

$\Delta H$ becomes the primary reinforcement learning signal.

---

## 6. Model 2 - Reinforcement Learning Agent

Purpose: learn optimal spray quantities using historical outcomes.

State (s):

- Location
- Disease type
- Disease severity

Action (a):

- Spray quantity in ml

Reward:

$$r(s,a) = \Delta H - \alpha \cdot E - \beta \cdot P$$

Where:

- $E$ = excess spray penalty
- $P$ = revisit penalty

Output:

- Optimized spray quantity

---

## 7. Training Pipeline

Data sources:

- Model 1 logs
- Crop state database
- Revisit comparator outputs

Generated dataset:

$(s, a, r, s')$

Training method:

- Soft Actor-Critic (SAC)

Output:

- Optimized spraying policy

---

## 8. Deployment Pipeline

Training occurs in the cloud. The policy is exported to:

- ONNX
- TensorFlow Lite

Deployment:

Cloud -> Edge device -> Drone

---

## 9. Continuous Learning Loop

Mission -> Data Collection -> Cloud Storage -> Reward Generation -> SAC Training -> Policy Update -> Deployment -> Next Mission

The system continuously improves spraying decisions over time.
