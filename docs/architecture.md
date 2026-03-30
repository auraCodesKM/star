# System Architecture — PingSight

> For visual pipeline diagrams see [`assets/system_architecture.md`](../assets/system_architecture.md) and [`assets/pipeline_flow.md`](../assets/pipeline_flow.md).

## 1. Overview

PingSight is organized as a **five-stage sequential pipeline** with a shared configuration layer and an optional coaching output layer. Each stage is independently testable and replaceable.

```
Video Input
    │
    ▼
┌───────────────────────┐
│  Stage 1: Perception  │  2D ball + table detection per frame
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Stage 2: Calibration │  Camera geometry estimation from table keypoints
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Stage 3: Uplifting   │  2D detections → 3D trajectory + spin
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Stage 4: Analytics   │  Zone classification, bounce detection, patterns
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│  Stage 5: Coaching    │  RAG agent → strategic report + visualizations
└───────────────────────┘
```

---

## 2. Module Breakdown

### 2.1 Ball Detection Module

**Role:** Locate the table tennis ball in each frame with pixel-level precision.

| Raw Detection | Ensemble-Filtered |
|:---:|:---:|
| ![Raw ball](../assets/detection/raw_ball_detections.png) | ![Filtered ball](../assets/detection/filtered_ball_detections.png) |
| Single-model output — noisy. | Post ensemble + DBSCAN — clean trajectory. |

**Design:**
- Dual-encoder ensemble: a primary transformer-based segmentation model and a secondary auxiliary detector
- Input: Three consecutive frames stacked as a temporal context window (t-1, t, t+1)
- Output: Probability heatmap → argmax → pixel coordinate (u, v) with confidence score
- Post-processing: DBSCAN clustering across a sliding temporal window to remove false positives from audience movement or frame artifacts

**Key trade-offs:**
- Using frame triplets (vs. single frame) adds ~1.5ms latency but reduces false detection rate significantly
- Dual-model ensemble increases compute by ~2× but provides fallback when one model fails under occlusion

---

### 2.2 Table Detection Module

**Role:** Locate 13 canonical keypoints of the table in each frame to establish the world coordinate reference.

| Raw Keypoints | Stable Filtered Keypoints |
|:---:|:---:|
| ![Raw table kp](../assets/detection/raw_table_keypoints.png) | ![Filtered table kp](../assets/detection/filtered_table_keypoints.png) |
| Single-model detections — some noise at occluded corners. | Dual-model ensemble output — stable, used for camera calibration. |

**Design:**
- Dual-model ensemble (two independent keypoint detectors)
- 13-point geometry covers all corners, mid-points, net endpoints, and table center
- Keypoint clustering resolves prediction disagreements
- Handles camera pan/zoom common in broadcast footage by re-calibrating per-segment

**Key trade-offs:**
- 13 points are over-determined for camera calibration (minimum 4 needed), which provides robustness against partially occluded tables
- Re-calibration per segment adds cost but is necessary for broadcast video where the camera operator moves

---

### 2.3 Camera Calibration Module

**Role:** Compute the projection matrix P mapping 3D world coordinates to 2D pixel coordinates.

**Design:**
```
Known:         13 table keypoints in world coordinates (meters, fixed geometry)
Observed:      13 table keypoints in pixel coordinates (from Stage 2)

Algorithm:     Direct Linear Transform (DLT)
Refinement:    Levenberg-Marquardt nonlinear optimization
Output:        3×4 projection matrix P (or intrinsic K + extrinsic [R|t])
```

**Why Levenberg-Marquardt:** Pure DLT is sensitive to pixel-level noise. LM minimizes reprojection error iteratively, achieving sub-pixel accuracy even under detector noise.

---

### 2.4 3D Uplifting Module

**Role:** Given 2D ball detections and camera parameters, recover the full 3D position and spin state.

| 2D Input to Transformer | 3D→2D Reprojection (Validation) |
|:---:|:---:|
| ![2D input](../assets/uplifting/2d_input_to_uplifting.png) | ![Reprojection](../assets/uplifting/2d_reprojection_from_3d.png) |
| All 2D detections fed into the uplifting model (green scatter). | Recovered 3D position reprojected back to 2D. Reprojection error < 1.2 px. |

![3D Trajectory Visualization](../assets/demo/3d_trajectory_visualization.png)
*3D world-coordinate view of a reconstructed rally — orange line = ball arc, green dots = table keypoints.*

**Architecture:**

```
Input:
  - Sequence of 2D pixel coordinates [(u₁,v₁), ..., (uₙ,vₙ)]
  - Table keypoint sequence
  - Camera matrix P

Preprocessing:
  - Normalize all coordinates to 1920×1080 canonical resolution
  - Compute partial depth cue via back-projection ray

Encoder:
  - Ball positions → BallEmbedding (linear projection + learned features)
  - Table geometry → TableEmbedding
  - Time index → RotaryPositionalEmbedding (frequency-based, time-aware)

Transformer Backbone:
  - Multi-head attention with rotary positional bias
  - Captures temporal dependencies across stroke segments

Multi-Stage Decoder:
  Stage 1: Coarse [x, y, z] estimate
  Stage 2: Residual correction (fine-grained)
  Stage 3: Spin vector [τ_topspin, σ_sidespin] estimation

Output:
  - [x, y, z] in meters, world coordinate frame
  - Spin magnitude + axis per timestep
```

**Why Rotary Positional Embeddings:** Standard learned positional embeddings don't generalize to variable-length sequences (rallies vary in stroke count). Rotary embeddings encode relative temporal distance as a rotation in attention space, generalizing naturally.

---

### 2.5 Analytics Module

**Role:** Interpret 3D trajectory data as sports-meaningful events and patterns.

![Zone Grid](../assets/demo/zones.png)
*The 9-zone tactical grid (per side, 18 total). Each bounce landing maps to a named zone used in coaching reports.*

![Annotated Frame](../assets/demo/annotated_frame.png)
*Live zone-aware broadcast frame (left) paired with the 3D physics model showing the active landing zone (right).*

#### 2.5.1 Bounce Detection

```
Input:  Z(t) time series (vertical position in meters)
Logic:
  TABLE_H = 0.76m  (standard table height)
  bounce = (Z(t) < TABLE_H + ε) AND (Z(t-1) > TABLE_H + ε)
  with multi-frame sliding window to reject noise spikes
Output: Bounce frame indices + world (x,y) landing coordinates
```

#### 2.5.2 Zone Classification

```
Table is divided into a 3×3 grid per side (18 zones total):

  Coordinate ranges (meters, centered at net):
    Row a (baseline): y ∈ [±0.92, ±1.37]
    Row b (middle):   y ∈ [±0.46, ±0.92]
    Row c (net):      y ∈ [0,     ±0.46]

    Col 1 (center):   x ∈ [-0.25, +0.25]
    Col 2 (mid):      x ∈ [±0.25, ±0.50]
    Col 3 (wide):     x ∈ [±0.50, ±0.76]

Zone label = row + col + side_prime  (e.g., "a1", "b3'", "c2")
Zone "0" = out of bounds
```

#### 2.5.3 Temporal Pattern Analysis

```
Per-rally triplet extraction:
  (prev_zone, curr_zone, next_zone) for each bounce

Aggregation across N rallies:
  - Frequency matrix F[i][j] = P(transition i→j)
  - Marginal distribution = zone usage heatmap
  - Pattern sequences = common multi-bounce chains
```

#### 2.5.4 Rally Segmentation

```
Events detected:
  - Stroke: ball reverses x-direction velocity
  - Point end: ball leaves table bounds or hits net
  - Serve: first bounce after serve position

Output: List of [rally_id, stroke_sequence, winner_side, duration]
```

---

### 2.6 Coaching Agent Module

**Role:** Transform structured analytics data into natural-language strategic advice.

**Architecture:**

```
Vector Store:
  - Pre-indexed corpus of table tennis tactics
  - Player profiles (if historical data available)
  - Zone-strategy associations (e.g., "wide serve to c3 → backhand weakness")

RAG Pipeline:
  Query = encode(current_player_stats + zone_heatmap + spin_profile)
  Retrieved = top-K similar strategy documents
  Context = retrieved documents + current stats summary

LLM:
  Input:  Context + structured metrics
  Output: 3-section coaching report:
    1. Pattern Analysis (what the data shows)
    2. Opponent Exploitation Opportunities
    3. Training Recommendations
```

---

## 3. End-to-End Output

![Full Pipeline Output](../assets/demo/full_pipeline_output.png)
*Complete pipeline result: 3D-uplifted ball trajectory reprojected onto WTT broadcast footage. The dotted arc represents the recovered trajectory from monocular video alone.*

![Debug Overlay](../assets/demo/debug_overlay.png)
*Debug overlay combining all pipeline stages: table geometry (magenta border), 2D ball detections (green dots), and the reconstructed 3D trajectory.*

![Rally 3D Reconstruction](../assets/demo/rally_3d_reconstruction.png)
*Dual-view output — original broadcast (left) and recovered 3D ball positions in world coordinates (right). WTT Frankfurt: Felix Lebrun vs Truls Moregard.*

## 4. Data Flow Diagram

```
MP4 File
   │
   ├──[Decode]──▶  Frame Buffer (3-frame sliding window)
   │
   ├──▶  Ball Detector A ──┐
   │                       ├──▶ Ensemble Fusion ──▶ (u,v) + confidence
   └──▶  Ball Detector B ──┘
   │
   ├──▶  Table Detector A ──┐
   │                        ├──▶ Ensemble Fusion ──▶ 13 keypoints
   └──▶  Table Detector B ──┘
                │
                ▼
         Camera Calibration
                │ P (3×4 matrix)
                ▼
         Uplifting Transformer
                │
       [x,y,z,spin] per frame
                │
         Bounce Detector
                │
         Zone Classifier ──▶ (prev, curr, next) triplets
                │
         Rally Segmenter ──▶ Structured rally list
                │
         Pattern Aggregator ──▶ Zone heatmap + transition matrix
                │
         RAG Coaching Agent ──▶ Text coaching report
                │
         Visualizer ──▶ Annotated MP4 + GIF + PNG exports
```

---

## 5. Deployment Considerations

| Concern | Approach |
|---|---|
| **GPU Memory** | Models loaded lazily; ball/table detectors share backbone if possible |
| **Inference Speed** | Frame-parallel ball/table detection; batched transformer inference |
| **Video Length** | Chunked rally processing; memory-bounded frame buffer |
| **Model Weights** | PyTorch Hub integration for on-demand weight download |
| **Configuration** | OmegaConf YAML for all hyperparameters and paths |

---

## 6. Inter-Module Contracts

Each module communicates through well-defined data structures:

```
BallDetection Output:
  {frame_id: int, u: float, v: float, confidence: float}

TableDetection Output:
  {frame_id: int, keypoints: List[(u,v)], camera_matrix: np.ndarray}

UpliftingOutput:
  {frame_id: int, x: float, y: float, z: float,
   spin_top: float, spin_side: float}

BounceEvent:
  {frame_id: int, x: float, y: float, zone: str,
   prev_zone: str, next_zone: str}

RallySegment:
  {rally_id: int, start_frame: int, end_frame: int,
   bounces: List[BounceEvent], winner: str}
```
