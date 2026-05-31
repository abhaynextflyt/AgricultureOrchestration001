# Model 2 Low Level Design (LLD) - Reinforcement Learning Based Spray Optimization - V1.0 (Design Phase)

## 1. Objective

Model 2 is a reinforcement learning (RL) based spray optimization engine designed to improve crop treatment effectiveness over time. Unlike Model 1, which performs disease detection and produces an initial spray recommendation, Model 2 learns from historical field outcomes and determines improved spray quantities for future missions.

Primary objectives:

- Improve crop health after spraying.
- Reduce unnecessary chemical usage.
- Reduce repeat visits caused by under-spraying.
- Continuously improve spraying decisions using collected field data.

---

## 2. Scope and Assumptions

The following assumptions apply to Version 1:

- **A1:** Model 1 provides disease type, disease severity (0 to 1), and an initial spray quantity recommendation.
- **A2:** Field missions are periodically repeated and the same crop locations can be revisited.
- **A3:** A crop health metric can be computed during revisits (for example NDVI, leaf color index, or disease severity reduction).
- **A4:** Historical mission data is stored and accessible through the cloud training pipeline.
- **A5:** Model 2 training is performed in the cloud and not on edge devices.

---

## 3. Inputs

Model 2 receives structured crop state information generated from Model 1 outputs and mission logs.

Input fields:

| Field             | Type   | Description                                           |
| ----------------- | ------ | ----------------------------------------------------- |
| x                 | float  | GPS X coordinate                                      |
| y                 | float  | GPS Y coordinate                                      |
| disease_type      | string | Disease classification                                |
| severity          | float  | Disease severity score (0 to 1)                       |
| previous_spray_ml | float  | Previous spray quantity (optional future enhancement) |

Example:

```json
{
  "x": 101.3,
  "y": 202.4,
  "disease_type": "rust",
  "severity": 0.78
}
```

---

## 4. Outputs

Model 2 produces a single recommendation:

- `spray_qty_ml`: quantity of spray to apply in milliliters

Example:

```json
{
  "spray_qty_ml": 12.5
}
```

---

## 5. State Definition

State represents the agricultural condition observed at a waypoint.

Version 1 state:

$$s = (x, y, d, \sigma)$$

Where:

- $x, y$ = spatial location of the crop
- $d$ = disease type
- $\sigma$ = disease severity, normalized to $[0, 1]$

Future versions may include temperature, humidity, wind speed, soil moisture, crop age, and historical spray count.

---

## 6. Action Definition

Action represents the spray decision.

$$a = \text{spray\_qty\_ml}$$

Action space:

$$a \in [0, a_{\max}]$$

The action space is continuous, which makes SAC a suitable algorithm.

---

## 7. Reward Definition

Reward quantifies the effectiveness of a spraying decision.

$$r(s,a) = \Delta H - \alpha \cdot E - \beta \cdot P$$

Where:

- $\Delta H = H_{t2} - H_{t1}$ is crop health improvement after $N$ days
- $E$ is the excess spray penalty
- $P$ is the revisit penalty

Expected behavior:

- Higher crop recovery -> higher reward
- Higher chemical waste -> lower reward
- More repeat visits -> lower reward

---

## 8. Dataset Generation Pipeline

Training data originates from Model 1 operational logs.

1. Mission execution: drone captures crop images.
2. Model 1 inference generates `disease_type`, `severity`, and `spray_qty_ml`.
3. Data logger stores:

```json
{
  "x": 101.3,
  "y": 202.4,
  "timestamp": "2026-05-31T10:15:00Z",
  "disease_type": "rust",
  "severity": 0.78,
  "spray_qty_ml": 15,
  "mission_id": "mission_0142"
}
```

4. Revisit mission performed after $N$ days.
5. Crop health re-evaluated.
6. Revisit comparator computes $\Delta H$.
7. Reward generated.
8. RL dataset created.

Final training record:

$$(s, a, r, s')$$

---

## 9. Replay Buffer Schema

Replay buffer stores historical transitions used during RL training.

Structure:

```json
{
  "state": {
    "disease_type": "rust",
    "severity": 0.75
  },
  "action": 12,
  "reward": 4.2,
  "next_state": {
    "disease_type": "rust",
    "severity": 0.3
  }
}
```

Storage: cloud training environment

Purpose: efficient sampling during RL training

---

## 10. Training Flow

1. Collect operational field data.
2. Generate reward signals.
3. Construct RL dataset.
4. Load transitions into replay buffer.
5. Train RL agent.
6. Validate policy.
7. Export trained model.

Training algorithm:

- Soft Actor-Critic (SAC)

Alternatives for experimentation:

- PPO
- TD3

---

## 11. Inference Flow

At runtime:

Crop image -> Model 1 -> disease type + severity -> Model 2 -> optimized spray quantity -> spray actuator

Model 2 acts as a spray optimization layer above disease detection.

---

## 12. Model Export

After training, the RL policy model is exported to ONNX format.

Purpose:

- Lightweight inference
- Edge compatibility
- Faster deployment

Output:

- `model2_policy.onnx`

---

## 13. Edge Deployment

Deployment flow:

Cloud training -> ONNX export -> OTA update -> Raspberry Pi -> Mission execution

The edge device loads the latest deployed policy and uses it during spraying operations.

---

## 14. Failure Modes and Mitigations

| ID  | Failure mode                          | Impact                       | Mitigation                                 |
| --- | ------------------------------------- | ---------------------------- | ------------------------------------------ |
| F1  | Insufficient training data            | Poor RL policy quality       | Minimum data threshold before training     |
| F2  | Incorrect disease predictions from M1 | Inaccurate state information | Model 1 validation and periodic retraining |
| F3  | Noisy health measurements             | Incorrect rewards            | Health score smoothing and normalization   |
| F4  | Over-spraying learned policy          | Chemical wastage             | Reward penalties and action limits         |

---

## 15. Future Improvements

Future versions may include:

- Weather-aware state space
- Multi-disease optimization
- Multi-objective reward functions
- Dynamic revisit scheduling
- Multi-agent drone coordination
- Online reinforcement learning
- Geospatial trajectory optimization

---

## Conclusion

Model 2 is a cloud-trained reinforcement learning system that continuously learns optimal spray quantities from historical crop outcomes. It uses field observations, spraying actions, and revisit health measurements to improve future spraying decisions while minimizing chemical waste and repeat interventions.
