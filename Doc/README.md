

> This doc explains the **pipeline** — how data flows from "look at a plant" to "the AI learns from it."  
> Written like a human. No fluff. Just the flow.

---

## The Pipeline in One Sentence

```
Camera sees plant -> AI figures out what's wrong -> Sensors check weather -> 
GPS knows where -> Everything becomes a "state" -> SAC brain decides spray amount -> 
Pump sprays -> We wait -> Check if plant got better -> Give AI a score -> 
Store memory -> AI learns -> Repeat
```

---

## Stage 1: SEE (Perception Layer)

### 1.1 Model Output — "What disease is this?"

```
Input:  Camera image of a leaf
Output: Disease Type + Severity Score
```

**What happens:**
- A CNN (like ResNet) looks at the image
- Says: "This is Leaf Blight, severity 0.73" (73% bad)
- This info travels downstream to the State Builder

**Side note:** The small box below says "Detects plant disease from image. Outputs disease type and severity score."

---

### 1.2 Sensor Manager — "What's the weather like?"

```
Output: Temperature, Humidity
```

**What happens:**
- DHT22 or similar sensor reads temp and humidity every second
- Hot + humid = disease spreads faster
- The AI needs this to decide "spray more or less?"

---

### 1.3 GPS Module — "Where are we?"

```
Output: X Coordinate, Y Coordinate
```
**What happens:**
- GPS module gives lat/long or grid coordinates
- Prevents spraying the same spot twice
- Lets you map exactly where each treatment happened

---

## Stage 2: THINK (State Building)

### 2.1 State Builder Module — "Pack everything into a box"

```
Input:  Disease info + Sensor data + GPS + Past spray history
Output: State Vector S(t) — a list of 10 numbers
```

**The 10 numbers:**

| # | What | Example |
|---|------|---------|
| 1 | disease_id | 3 (Leaf Blight) |
| 2 | severity | 0.73 |
| 3 | soil_moisture | 45% |
| 4 | temperature | 28C |
| 5 | humidity | 65% |
| 6 | pest_count | 12 |
| 7 | plant_health_score | 0.4 (was already struggling) |
| 8 | previous_spray | 0 ml (never sprayed here before) |
| 9 | x_coordinate | 34.05 |
| 10 | y_coordinate | -118.24 |

**The small box below says:** "Combines disease, sensor, GPS, and historical spray information. Creates state vector S(t)."

---

### 2.2 Sensor Database — "What do we know about this disease?"

```
Input:  disease_id
Output: Recommended pesticide, base spray rate
```

**What happens:**
- Looks up disease #3 in the database
- Finds: "Leaf Blight -> Fungicide-X -> base rate 150ml"
- This feeds into the Plant Condition Analyzer as reference

---

### 2.3 Plant Condition Analyzer — "Double-check everything"

```
Input:  Image features + Database lookup
Output: Confirmed Disease ID + Severity + Confidence
```

**What happens:**
- Cross-checks the CNN's guess against the database
- Adds confidence score: "I'm 94% sure this is Leaf Blight"
- **Policy Manager** (small box below) decides: "Should I trust this or explore?"

---

## Stage 3: DECIDE (SAC Agent)

### 3.1 RL Agent (SAC) — "The Brain"

```
Input:  State Vector S(t) [10 numbers]
Output: Action A(t) [spray amount] + Q-Value [how good is this?]
```

**Inside the brain:**

```
Actor Network (Policy)          Critic Network (Q-Function)
----------------------          ---------------------------
Input:  S(t) [10]               Input:  S(t) + A(t) [11]
Layer 1: 256 neurons            Layer 1: 256 neurons
Layer 2: 128 neurons            Layer 2: 128 neurons
Output: Mean + StdDev [2]       Output: Q-Value [1]
        |
        
Sample action using
reparameterization trick
```

**The small boxes around it:**
- **Critic Network** (left): "Evaluates selected action and estimates future reward value"
- **Actor Network** (bottom left): "Decides action vector. Predicts spray quantity for current condition."
- **RL Agent** (bottom): "Uses SAC algorithm to learn optimal spray quantity for each plant."

**Why SAC (not PPO)?**
- SAC learns from OLD data (off-policy) — efficient, doesn't waste sprays
- SAC handles continuous spray amounts naturally
- SAC explores automatically via entropy — tries different amounts intelligently

---

### 3.2 Action Generator — "Convert brain output to real numbers"

```
Input:  Raw action from SAC [0.0 - 1.0]
Output: Spray Quantity [0 - 500 ml]
```

**What happens:**
```python
raw_action = 0.36          # SAC outputs this
spray_ml = 0.36 * 500      # scale to ml
spray_ml = clamp(180, 0, 500)  # safety: never below 0, never above 500
```

**The small box says:** "Converts RL output into actual spray quantity to hardware."

---

## Stage 4: ACT (Hardware Control)

### 4.1 Spray Controller — "Tell the pump what to do"

```
Input:  Spray quantity in ml
Output: PWM signal to pump
```

**What happens:**
- Software says: "Spray 180 ml"
- Controller calculates: "At my flow rate, that's 3.6 seconds at 80% power"
- Sends PWM signal to the pump

**The small box says:** "Converts PWM signal and controls motor for precise spraying."

---

### 4.2 Crop Treatment — "Actually spray the plant"

```
Input:  PWM signal
Output: Physical spray on the plant
```

**What happens:**
- Nozzle opens, pesticide comes out
- Flow sensor measures actual amount sprayed
- Reports back: "I sprayed 178 ml" (slightly off due to pressure)

**The small box says:** "Activates spray and applies pesticide to infected crop area."

---

## Stage 5: LEARN (Feedback Loop)

### 5.1 Data Logger — "Write everything down"

```
Input:  State, Action, GPS, Disease info, Spray amount, Timestamp
Output: SQLite database + Action logs
```

**What gets logged:**
```
Time: 2026-06-03 14:23:05
Where: (34.05, -118.24)
What: Leaf Blight, severity 0.73
Weather: 28C, 65% humidity
Decision: Spray 180 ml
Actual: 178 ml sprayed
```

**The small box says:** "Stores disease type, GPS, spray quantity, and reward history."

---

### 5.2 N-Day Result — "Did it work?"

```
Input:  Drone image / NDVI scan after N days
Output: New Health Score
```

**What happens:**
- Wait 7 days (or whatever N is configured to)
- Fly drone again, take NDVI image
- Calculate new health score: 0.85 (much better!)

**The small box says:** "Agent visits that plant/area N days and then again checks condition of the plant."

### 5.3 Reward Calculator — "How did we do?"

```
Input:  Health before, Health after, Spray cost, Health penalty
Output: Reward score
```

**The math:**
```python
health_improvement = 0.85 - 0.40 = +0.45   # Great improvement!
spray_cost = 180 / 500 = 0.36               # Used 36% of max tank
health_penalty = 0                           # No penalty, plant got better

reward = 0.45 * 1.0 - 0.36 * 0.5 - 0 * 0.8
reward = 0.45 - 0.18 = +0.27                # Positive! Good job, AI.
```

**The small box says:** "Calculates reward using improvement, spray cost, and environmental efficiency."

---

### 5.4 Replay Buffer — "Store the memory"

```
Input:  (State, Action, Reward, Next State, Done)
Output: Stored in buffer for training
```

**What gets stored:**
```python
experience = {
    "state":      [3, 0.73, 45, 28, 65, 12, 0.40, 0, 34.05, -118.24],
    "action":     [0.36],                          # 180 ml
    "reward":     0.27,                            # Good score
    "next_state": [3, 0.10, 42, 29, 60, 2, 0.85, 180, 34.05, -118.24],
    "done":       False                            # Keep going
}
```

**The small box says:** "Stores past experiences for training."

---

### 5.5 Back to SAC — "Learn from experience"

```
Input:  Batch of experiences from Replay Buffer
Output: Updated Actor + Critic networks
```

**What happens:**
1. Sample 256 random experiences from buffer
2. Update Critic: "Was my prediction of reward close to reality?"
3. Update Actor: "How can I choose better actions?"
4. Update Temperature (alpha): "Am I exploring enough?"
5. Soft-update target networks (slow copies for stability)

**Then the loop repeats.** Next plant, next decision, smarter AI.

---

## Full Pipeline Visual

```
 SEE                    THINK                   DECIDE
-----                 ---------                --------
  |                       |                       |
  v                       v                       v
+---------+          +-----------+          +------------+
| Camera  |          |  State    |          |   SAC      |
| Image   |--------->|  Builder  |--------->|   Agent    |
+---------+          |  S(t)     |          |   A(t)     |
                     +-----------+          +------------+
  |                       |                       |
  v                       v                       v
+---------+          +-----------+          +------------+
| Sensor  |          |  Plant    |          |  Action    |
| Temp/   |--------->|  Analyzer |          |  Generator |
| Humid   |          |  Verify   |          |  Convert   |
+---------+          +-----------+          +------------+
  |                                               |
  v                                               v
+---------+                                 +------------+
|  GPS    |                                 |  Spray     |
| X,Y     |                                 |  Controller|
+---------+                                 +------------+
                                                  |
                                                  v
                                             +------------+
                                             |  Crop      |
                                             |  Treatment |
                                             |  (Spray)   |
                                             +------------+
                                                  |
 LEARN <------------------------------------------+
------
  |
  v
+------------+     +------------+     +------------+     +------------+
|  Data      |     |  N-Day     |     |  Reward    |     |  Replay    |
|  Logger    |---->|  Result    |---->|  Calculator|---->|  Buffer    |
|  Store all |     |  Check     |     |  Score it  |     |  Remember  |
+------------+     +------------+     +------------+     +------------+
                                                                |
                                                                v
                                                         +------------+
                                                         |  Train     |
                                                         |  SAC       |
                                                         |  (Update   |
                                                         |  networks) |
                                                         +------------+
```

---

## Data Flow Summary Table

| Step | Module | Input | Output | Goes To |
|------|--------|-------|--------|---------|
| 1 | Camera | Physical plant | Image | Disease Model |
| 2 | Disease Model | Image | disease_id, severity | State Builder |
| 3 | Sensor Manager | Physical env | temp, humidity | State Builder |
| 4 | GPS Module | Satellite | X, Y | State Builder |
| 5 | State Builder | All above | S(t) [10 nums] | SAC Agent |
| 6 | Sensor DB | disease_id | pesticide info | Plant Analyzer |
| 7 | Plant Analyzer | Image + DB | confirmed disease | State Builder |
| 8 | SAC Agent | S(t) | action, Q-value | Action Generator |
| 9 | Action Generator | raw action | spray_ml | Spray Controller |
| 10 | Spray Controller | spray_ml | PWM signal | Crop Treatment |
| 11 | Crop Treatment | PWM | physical spray | Data Logger |
| 12 | Data Logger | everything | SQLite records | N-Day Result |
| 13 | N-Day Result | drone image | new health | Reward Calculator |
| 14 | Reward Calculator | before/after | reward score | Replay Buffer |
| 15 | Replay Buffer | experience tuple | stored memory | SAC Training |
| 16 | SAC Training | batch of memories | updated weights | Back to step 8 |

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| **SAC instead of PPO** | Off-policy = learn from old data. Continuous actions = any spray amount. Entropy = smart exploration. |
| **Wait N days for reward** | Pesticides don't work instantly. Need real results before judging. |
| **Twin Critic networks** | Prevents Q-value overestimation. More stable training. |
| **Replay Buffer** | SAC can reuse experiences. Efficient for expensive real-world data. |
| **SQLite local DB** | Works offline. No cloud dependency in the field. |
| **GPS tagging** | Prevents double-spray. Enables zone analytics. Compliance logging. |

---

## Quick Numbers Reference

| Parameter | Value | Why |
|-----------|-------|-----|
| State size | 10 | All relevant info in one vector |
| Action size | 1 | Just spray quantity (extendable) |
| Max spray | 500 ml | Safety cap |
| Actor hidden | [256, 128] | Enough capacity, not too slow |
| Critic hidden | [256, 128] | Match actor for stability |
| Buffer size | 1,000,000 | Store lots of field experience |
| Batch size | 256 | Standard for SAC |
| Learning rate | 3e-4 | Safe default |
| Gamma | 0.99 | Care about long-term plant health |
| Tau | 0.005 | Slow target network updates |
| N-day wait | 7 days | Typical pesticide effect timeline |

---

> **That's the pipeline.**  
> See -> Think -> Decide -> Act -> Learn -> Repeat.  
> Every spray makes the AI a little smarter.
"""
