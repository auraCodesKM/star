# Data Pipeline — PingSight

> **Visual Reference:** Key pipeline stage outputs are shown inline below using real system outputs from WTT broadcast footage (Frankfurt, Incheon, Fukuoka).

## 1. Input Requirements

| Parameter | Requirement |
|---|---|
| Format | MP4 (H.264/H.265) or equivalent |
| Resolution | Any; normalized to 1920×1080 internally |
| Frame rate | 25–120 fps (higher = better trajectory resolution) |
| Camera | Fixed broadcast angle preferred; pan/zoom handled |
| Content | Must show the full table; partial occlusion tolerated |

---

## 2. Preprocessing Stage

```
Raw Video
  │
  ├── Frame extraction (OpenCV VideoCapture)
  │     └── [frame_t-1, frame_t, frame_t+1] triplet buffer
  │
  ├── Resolution normalization
  │     └── Resize to 1920×1080 (bicubic), preserve aspect ratio
  │
  └── Temporal segmentation
        └── Split into rally segments at scene cut detection
```

---

## 3. Per-Frame Processing

```
Triplet [F_{t-1}, F_t, F_{t+1}]
  │
  ├── Ball Detection Branch
  │     ├── Primary model → heatmap P_A(u,v)
  │     ├── Auxiliary model → heatmap P_B(u,v)
  │     ├── Ensemble: P = α·P_A + (1-α)·P_B
  │     └── Peak extraction → (u, v, confidence)
  │
  └── Table Detection Branch
        ├── Primary model → 13 keypoint candidates
        ├── Auxiliary model → 13 keypoint candidates
        ├── Per-keypoint clustering → best hypothesis
        └── 13 pixel coordinates
```

---

**Per-frame detection outputs:**

| Ball (raw) | Ball (filtered) | Table (raw) | Table (filtered) |
|:---:|:---:|:---:|:---:|
| ![rb](../assets/detection/raw_ball_detections.png) | ![fb](../assets/detection/filtered_ball_detections.png) | ![rt](../assets/detection/raw_table_keypoints.png) | ![ft](../assets/detection/filtered_table_keypoints.png) |

## 4. Per-Rally Processing

```
Rally segment: sequence of N frames
  │
  ├── Camera Calibration
  │     ├── Input: 13 pixel keypoints (per segment, not per frame)
  │     ├── DLT → initial P
  │     └── LM optimization → refined P (3×4 matrix)
  │
  ├── Temporal Ball Filtering
  │     ├── DBSCAN on (frame_id, u, v) point cloud
  │     └── Output: cleaned ball trajectory [(frame, u, v)]
  │
  └── 3D Uplifting
        ├── Input: cleaned 2D detections + P
        ├── Normalize + embed
        ├── Transformer forward pass
        └── Output: [(frame, x, y, z, spin_top, spin_side)]
```

---

**3D uplifting inputs and validation:**

| 2D scatter input | 3D reprojection validation | Recovered 3D trajectory |
|:---:|:---:|:---:|
| ![2d](../assets/uplifting/2d_input_to_uplifting.png) | ![rep](../assets/uplifting/2d_reprojection_from_3d.png) | ![3d](../assets/demo/3d_trajectory_visualization.png) |

## 5. Analytics Processing

```
3D trajectory sequence
  │
  ├── Bounce Detection
  │     ├── Z-height threshold filtering
  │     ├── Local minimum detection
  │     └── Output: bounce frame indices
  │
  ├── Zone Classification
  │     ├── Map (x, y) at each bounce → zone label
  │     └── Output: [(frame, zone, side)]
  │
  ├── Temporal Pattern Extraction
  │     ├── Build (prev, curr, next) triplets per bounce
  │     ├── Transition matrix construction
  │     └── Frequency heatmap computation
  │
  └── Rally Segmentation
        ├── Detect direction reversals (strokes)
        ├── Detect out-of-bounds / net events
        └── Output: structured rally list
```

---

**Zone classification reference:**

![Zones](../assets/demo/zones.png)
*Zone substitution table and 9-zone grid (per side). Each bounce landing coordinates map to one of 18 labelled zones.*

## 6. Output Generation

```
Analytics results
  │
  ├── Visualizer
  │     ├── Frame annotation (zone label, bounce marker, comet trail)
  │     ├── 3D trajectory subplot (rotating view)
  │     ├── Zone grid overlay (color-coded)
  │     ├── Export: analysis_normal.mp4
  │     └── Export: analysis_slow.gif (0.3× speed)
  │
  ├── Statistics Module
  │     ├── Zone frequency heatmap image
  │     ├── Spin distribution histogram
  │     └── Rally length distribution
  │
  └── Coaching Agent
        ├── Query construction from stats
        ├── RAG retrieval from knowledge base
        └── Report generation → coaching_report.txt
```

---

**Final pipeline outputs:**

| Full pipeline output | Debug combined overlay | Dual-view reconstruction |
|:---:|:---:|:---:|
| ![fp](../assets/demo/full_pipeline_output.png) | ![dbg](../assets/demo/debug_overlay.png) | ![r3d](../assets/demo/rally_3d_reconstruction.png) |

| Animated trajectory | Split-view animation |
|:---:|:---:|
| ![anim](../assets/demo/3d_trajectory_animation.gif) | ![split](../assets/demo/3d_trajectory_split_animation.gif) |

## 7. Data Formats

### Intermediate: Ball Detection Output
```json
{
  "frame_id": 142,
  "u": 823.4,
  "v": 441.2,
  "confidence": 0.94
}
```

### Intermediate: Camera Calibration Output
```json
{
  "segment_id": 3,
  "start_frame": 120,
  "end_frame": 280,
  "reprojection_error_px": 1.23,
  "camera_matrix": [[...], [...], [...]]
}
```

### Intermediate: Uplifting Output
```json
{
  "frame_id": 142,
  "x_m": 0.34,
  "y_m": -0.82,
  "z_m": 1.12,
  "spin_topspin_rps": 87.3,
  "spin_sidespin_rps": 12.1
}
```

### Final: Bounce Event
```json
{
  "bounce_id": 7,
  "frame_id": 142,
  "x_m": 0.34,
  "y_m": -0.82,
  "zone": "b2",
  "side": "near",
  "prev_zone": "c1'",
  "next_zone": "a3",
  "spin_type": "topspin",
  "spin_magnitude_rps": 87.3
}
```

### Final: Coaching Report (text)
```
TACTICAL REPORT
Generated: 2025-11-14  |  Video: training_session_12.mp4  |  Rallies analyzed: 47

== PATTERN ANALYSIS ==
...
```

---

## 8. Performance Characteristics

| Stage | Approximate Processing Time (per second of video) |
|---|---|
| Frame extraction | ~10ms |
| Ball detection (dual model) | ~85ms |
| Table detection (dual model) | ~110ms |
| Camera calibration (per segment) | ~15ms |
| DBSCAN filtering | ~2ms |
| 3D uplifting (transformer) | ~45ms |
| Analytics (zone, bounce, pattern) | ~5ms |
| Visualization rendering | ~60ms |
| **Total** | **~330ms / second of video** |

*Benchmarked on NVIDIA A100 GPU, 1920×1080 input at 60fps source.*

---

## 9. Synthetic Training Data Pipeline

```
MuJoCo Configuration
  │
  ├── Ball parameters: radius, mass, restitution, friction
  ├── Spin: initial angular velocity (3-vector)
  ├── Initial conditions: position, velocity (randomized ranges)
  └── Camera: position, orientation, FOV (broadcast-realistic ranges)
  │
  ▼
Physics Simulation Loop (per timestep 1/1000s)
  │
  ├── Apply gravity (9.81 m/s²)
  ├── Apply Magnus force (spin × velocity cross product)
  ├── Detect bounce (Z = TABLE_H), apply dissipation
  └── Record [x, y, z, spin] state
  │
  ▼
Camera Projection
  │
  └── Project 3D → 2D pixel coordinates (+ optional noise)
  │
  ▼
Dataset Export
  └── [(2D sequence, 3D ground truth, spin ground truth)]
```
