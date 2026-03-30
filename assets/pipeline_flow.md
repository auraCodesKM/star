# Pipeline Flow Diagrams — PingSight

## Full Pipeline: Video to Report

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          INPUT STAGE                                     │
│                                                                          │
│   training_session.mp4                                                   │
│   [Resolution: any]  [FPS: 25-120]  [Format: MP4]                       │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │  Frame Extraction  │
                          │  OpenCV decode     │
                          │  Resize → 1920×1080│
                          └─────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼──────────┐       ┌────────────▼──────────┐
          │  BALL DETECTION    │       │  TABLE DETECTION       │
          │                    │       │                        │
          │  Input: triplet    │       │  Input: triplet        │
          │  [t-1, t, t+1]     │       │  [t-1, t, t+1]         │
          │                    │       │                        │
          │  Model A (primary) │       │  Model A (primary)     │
          │  Model B (aux)     │       │  Model B (auxiliary)   │
          │  Ensemble fusion   │       │  Keypoint clustering   │
          │                    │       │                        │
          │  Output: (u,v,conf)│       │  Output: 13 keypoints  │
          └─────────┬──────────┘       └────────────┬──────────┘
                    │                               │
          ┌─────────▼──────────┐       ┌────────────▼──────────┐
          │  DBSCAN FILTERING  │       │  CAMERA CALIBRATION   │
          │  Temporal window   │       │                        │
          │  Outlier removal   │       │  DLT initialization   │
          │                    │       │  LM refinement         │
          │  Output: cleaned   │       │                        │
          │  (frame, u, v)     │       │  Output: P (3×4)       │
          └─────────┬──────────┘       └────────────┬──────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    │
                          ┌─────────▼──────────┐
                          │  3D UPLIFTING       │
                          │                    │
                          │  Inputs combined:  │
                          │  2D detections     │
                          │  + Camera matrix P │
                          │                    │
                          │  BallEmbedding     │
                          │  TableEmbedding    │
                          │  RotaryPosEmb      │
                          │                    │
                          │  Transformer (×4)  │
                          │                    │
                          │  Stage 1: XYZ      │
                          │  Stage 2: +delta   │
                          │  Stage 3: Spin     │
                          │                    │
                          │  Output:           │
                          │  (x,y,z,s_top,     │
                          │   s_side) / frame  │
                          └─────────┬──────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
          ┌─────────▼──────┐ ┌──────▼──────┐ ┌─────▼────────┐
          │BOUNCE DETECTOR │ │ ZONE CLASS. │ │RALLY SEGMENT.│
          │                │ │             │ │              │
          │ Z < TABLE_H+ε  │ │ (x,y) → 18  │ │ Stroke detect│
          │ Local minimum  │ │ zone labels │ │ Point ends   │
          │ Min gap check  │ │             │ │              │
          └─────────┬──────┘ └──────┬──────┘ └─────┬────────┘
                    │               │               │
                    └───────────────┼───────────────┘
                                    │
                          ┌─────────▼──────────┐
                          │  TEMPORAL PATTERN  │
                          │  AGGREGATOR        │
                          │                    │
                          │  Bounce triplets   │
                          │  Transition matrix │
                          │  Zone heatmap      │
                          │  Spin distribution │
                          └─────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
          ┌─────────▼──────────┐       ┌────────────▼──────────┐
          │  COACHING AGENT    │       │  VISUALIZER            │
          │                    │       │                        │
          │  Query construction│       │  Frame annotation      │
          │  Vector retrieval  │       │  3D subplot            │
          │  LLM synthesis     │       │  Zone overlay          │
          │                    │       │  Comet trail           │
          │  Output:           │       │                        │
          │  coaching_report   │       │  Output:               │
          └────────────────────┘       │  analysis.mp4          │
                                       │  analysis.gif          │
                                       │  heatmap.png           │
                                       └────────────────────────┘
```

---

## Synthetic Data Generation Pipeline

```
┌──────────────────────────────────────────────────────────────┐
│                   MUJOCO SIMULATION                           │
│                                                               │
│  Config parameters:                                           │
│  ┌────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │ Ball physics   │  │ Initial state   │  │ Camera config │  │
│  │ mass, radius   │  │ position (x,y,z)│  │ position, FOV │  │
│  │ restitution    │  │ velocity (3D)   │  │ orientation   │  │
│  │ friction       │  │ spin (3D)       │  │ focal length  │  │
│  └────────────────┘  └─────────────────┘  └───────────────┘  │
│                ↓                ↓                ↓            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Physics Simulation Loop                     │  │
│  │                                                          │  │
│  │  For each timestep (1/1000s):                           │  │
│  │    1. Apply gravity: F_grav = m × 9.81 downward         │  │
│  │    2. Apply Magnus: F_mag = C_L × (ω × v)              │  │
│  │    3. Integrate acceleration → velocity → position      │  │
│  │    4. Check bounce: if Z ≤ TABLE_H → apply restitution  │  │
│  │    5. Record state [x, y, z, ω_x, ω_y, ω_z]           │  │
│  │    6. Break if out-of-bounds                            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Camera Projection                           │  │
│  │  3D position → project via P → 2D pixel coord           │  │
│  │  Optional: + Gaussian noise σ ~ N(0, 1.5px)            │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↓                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Dataset Export                              │  │
│  │  [(2D_seq, 3D_ground_truth, spin_ground_truth)]         │  │
│  │  50,000+ trajectories per run                           │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              ↓
              Used for pretraining the uplifting transformer
              before fine-tuning on real annotated data
```

---

## Training Pipeline

```
                    MODEL TRAINING PIPELINE

  Real annotated data                  Synthetic data (MuJoCo)
  (limited, high quality)              (large scale, perfect labels)
         │                                       │
         │                             ┌─────────▼──────────┐
         │                             │   Pretrain on      │
         │                             │   Synthetic Data   │
         │                             │   (Stage 1 + 2)    │
         │                             └─────────┬──────────┘
         │                                       │
         └───────────────┬───────────────────────┘
                         │
               ┌─────────▼──────────┐
               │  Fine-tune on      │
               │  Real Data         │
               │  (All 3 stages)    │
               └─────────┬──────────┘
                         │
               ┌─────────▼──────────┐
               │  Evaluation on     │
               │  Held-out test set │
               │  (Hawk-Eye GT)     │
               └─────────┬──────────┘
                         │
               ┌─────────▼──────────┐
               │  TensorBoard       │
               │  Monitoring        │
               │  Per-stage losses  │
               │  Reprojection err  │
               └────────────────────┘
```
