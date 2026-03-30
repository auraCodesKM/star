# Performance Benchmarks — PingSight

## 1. Detection Accuracy

### Ball Detection

| Metric | Single Model | Ensemble (Dual) | Notes |
|---|---|---|---|
| Precision@0.5px | 87.3% | 93.1% | Within 0.5px of ground truth |
| Recall | 84.6% | 91.4% | Detection rate on labeled frames |
| Miss rate | 15.4% | 8.6% | Frames where ball is not detected |
| False positive rate | 2.1% | 1.4% | Non-ball detections |

Key failure modes: extreme motion blur at >100 km/h, partial occlusion behind player limbs, ball near white edge lines.

### Table Keypoint Detection

| Metric | Value | Notes |
|---|---|---|
| Mean keypoint error | 2.8 px | Average distance from ground truth |
| Worst keypoint | 6.1 px | Far end corner (frequent occlusion) |
| Calibration success rate | 97.4% | Segments where calibration converges |
| Camera matrix reprojection error | 1.2 px | After LM refinement |

---

## 2. 3D Reconstruction Accuracy

Evaluated against ground truth from multi-camera Hawk-Eye reference system on held-out broadcast segments:

| Metric | Value | Notes |
|---|---|---|
| Mean 3D position error | 4.2 cm | Euclidean distance from ground truth |
| Z (height) error | 2.8 cm | Most critical for bounce detection |
| X (lateral) error | 3.1 cm | Left-right position |
| Y (depth) error | 5.1 cm | Hardest: monocular depth |
| Spin cosine similarity | 0.87 | Alignment with true spin vector |
| Spin magnitude error | 11.3 rps | Mean absolute error |

---

## 3. Analytics Accuracy

### Bounce Detection

| Metric | Value |
|---|---|
| Precision | 96.2% |
| Recall | 94.8% |
| F1 Score | 95.5% |

False detections mainly occur when ball grazes net or makes near-miss with table edge.

### Zone Classification

| Metric | Value | Notes |
|---|---|---|
| Zone accuracy | 88.4% | Correct zone assignment |
| Adjacent zone error | 9.8% | Off by one zone (boundary) |
| Large zone error | 1.8% | Off by two or more zones |

Zone boundary errors are the primary failure mode — physically a 5cm error near a zone boundary flips the classification.

### Rally Segmentation

| Metric | Value |
|---|---|
| Point boundary accuracy | 97.1% |
| Stroke count accuracy | 92.3% |
| Rally duration error | ±0.08s mean |

---

## 4. Processing Speed

### Offline Analysis (GPU)

| Hardware | Frames/second | ×Real-time |
|---|---|---|
| NVIDIA A100 (80GB) | 48 fps | 0.8× (near real-time at 60fps source) |
| NVIDIA RTX 4090 | 31 fps | 0.52× |
| NVIDIA RTX 3080 | 19 fps | 0.32× |
| NVIDIA T4 (cloud) | 14 fps | 0.23× |

*For a 60fps source video, A100 processes 1 hour of footage in ~1.25 hours.*

### Per-Stage Breakdown (A100, 60fps input)

| Stage | ms/frame | % of total |
|---|---|---|
| Frame decode | 0.2ms | 1.6% |
| Ball detection | 1.4ms | 11.3% |
| Table detection | 1.8ms | 14.5% |
| Camera calibration | 0.25ms | 2.0% |
| DBSCAN filtering | 0.03ms | 0.2% |
| 3D uplifting | 0.75ms | 6.0% |
| Analytics | 0.08ms | 0.6% |
| Visualization | 1.0ms | 8.1% |
| I/O (disk, encoding) | 7.0ms | 56.5% |

**I/O dominates**: disk read/write and video encoding account for 56% of total time. With NVMe SSD and hardware encoder, end-to-end can reach 55+ fps.

---

## 5. Model Size and Memory

| Module | Parameters | GPU Memory (inference) |
|---|---|---|
| Ball Detector Primary | 25M | 1.8 GB |
| Ball Detector Auxiliary | 12M | 0.9 GB |
| Table Detector Primary | 22M | 1.6 GB |
| Table Detector Auxiliary | 8M | 0.6 GB |
| Uplifting Transformer | 18M | 1.3 GB |
| **Total** | **85M** | **~6.2 GB** |

Minimum GPU memory: 8 GB (with model streaming). Comfortable: 12 GB. All models on GPU simultaneously: 24 GB.

---

## 6. Dataset Statistics

| Dataset | Clips | Frames | Annotations |
|---|---|---|---|
| Ball detection training | 1,240 | 186,000 | Pixel coordinates |
| Table detection training | 890 | 133,500 | 13 keypoints per frame |
| 3D uplifting training | 2,100 | 315,000 | 3D positions + spin |
| Synthetic (MuJoCo) | 50,000 | 7,500,000 | Perfect ground truth |
| Validation | 180 | 27,000 | Multi-source annotations |
| Test (Hawk-Eye reference) | 42 | 6,300 | Hawk-Eye ground truth |
