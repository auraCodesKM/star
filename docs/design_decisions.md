# Design Decisions — PingSight

This document explains the key architectural and algorithmic choices made in PingSight, including the trade-offs considered and the reasoning behind each decision.

---

## 1. Why Monocular (Not Stereo/Multi-Camera)?

**Decision:** Operate on single-camera broadcast footage.

**Alternatives considered:**
- Stereo camera rig → trivially solves depth estimation
- Multi-camera Hawk-Eye system → industry standard for professional tournaments
- Wearable IMU sensors → direct measurement

**Rationale:**
Professional setups are unavailable to 99% of training environments worldwide. A club coach with a smartphone recording should be the deployment target. Monocular video is the only format available at scale. The engineering challenge of monocular 3D reconstruction is hard, but solving it has massive reach.

---

## 2. Why Segmentation (Not Bounding Box Detection)?

**Decision:** Frame ball detection as heatmap segmentation, not object detection (YOLO-style bounding boxes).

**Rationale:**
- A 4–6 pixel ball produces a tiny bounding box with essentially no spatial information inside it
- Bounding box regression is unstable for objects smaller than the anchor sizes
- Heatmap approach produces a probability distribution over all pixels — the confidence is spatially graded, which is more informative
- Allows soft thresholding: low-confidence detections are preserved as weighted inputs rather than being hard-rejected

---

## 3. Why Temporal Triplets as Input?

**Decision:** Feed (t-1, t, t+1) to detection models rather than single frames.

**Rationale:**
- Single-frame ball appearance at high speed is ambiguous (could be small white object in background)
- Temporal context reveals motion direction, which is discriminative
- t+1 look-ahead is available during offline analysis (not real-time streaming)
- Adds only 2× frame read overhead, negligible compared to model forward pass cost

**Limitation:** Adds 1 frame latency. For offline analysis, this is irrelevant.

---

## 4. Why 13 Table Keypoints (Not 4 Corners)?

**Decision:** Detect 13 table keypoints for camera calibration.

**Rationale:**
- 4 points is the minimum for DLT, but provides zero robustness to detection noise
- With 13 points, up to 6 can be occluded or noisy and the calibration still converges
- Broadcast cameras frequently have table edges partially off-frame; extra interior keypoints ensure calibration succeeds
- Reprojection error is measurable and can trigger re-calibration when quality degrades

---

## 5. Why Levenberg-Marquardt for Calibration (Not Pure DLT)?

**Decision:** Use DLT as initialization, then refine with Levenberg-Marquardt nonlinear optimization.

**Rationale:**
- DLT assumes linear projection and is sensitive to keypoint localization errors
- Real cameras have lens distortion, and broadcast feeds are compressed (introducing compression artifacts at edges)
- LM minimizes reprojection error directly — the actual metric that matters
- Adds ~10ms per calibration event; calibration runs only at rally boundaries (every few seconds), so cost is negligible

---

## 6. Why Rotary Positional Embeddings?

**Decision:** Use Rotary Positional Embeddings (RoPE) in the uplifting transformer rather than learned absolute or sinusoidal positional embeddings.

**Rationale:**
- Rallies vary in length from 2 strokes to 30+ strokes
- Learned absolute positional embeddings overfit to seen sequence lengths
- Sinusoidal embeddings treat all temporal distances the same regardless of physics timescale
- RoPE encodes the **relative** time distance between frames as a rotation, which generalizes to unseen lengths
- The physics timescale (1/FPS) is explicitly encoded in the rotation frequency, giving the model a physics-grounded notion of time

---

## 7. Why Multi-Stage Prediction (Not Single-Stage)?

**Decision:** Three-stage decoder: coarse → refinement → spin.

**Rationale:**
- 3D position estimation and spin estimation have different difficulty profiles
- Position can be estimated from trajectory shape alone; spin requires subtle curvature analysis
- Multi-stage allows separate loss supervision at each stage, which stabilizes training
- Stage 2 refinement explicitly models residuals, which is easier to learn than the full position
- If Stage 1 produces a physically reasonable estimate, Stage 2 only needs to correct small errors — this is a well-understood inductive bias in residual networks

---

## 8. Why MuJoCo for Synthetic Data?

**Decision:** Use MuJoCo physics simulation to generate synthetic training trajectories.

**Rationale:**
- Annotated real-world table tennis videos with 3D ground truth are extremely scarce
- MuJoCo provides physically accurate rigid-body simulation including Magnus force (spin-induced deflection)
- Synthetic data with perfect ground truth labels can be generated at scale (50,000+ trajectories)
- Sim-to-real gap is mitigated by fine-tuning on real annotated data after synthetic pretraining

---

## 9. Why 18 Zones (3×3 per side)?

**Decision:** Partition the table into 18 discrete zones.

**Rationale:**
- Continuous (x, y) coordinates are hard to communicate to coaches and athletes
- Too few zones (4 quadrants) lose tactically important distinctions (near-net vs. deep)
- Too many zones produce sparse statistics with insufficient data per zone
- 3×3 per side (rows: net/middle/baseline, columns: forehand/center/backhand) maps directly to how coaches verbally describe placement
- This is the de facto vocabulary used in table tennis coaching literature

---

## 10. Why RAG Rather Than a Fine-Tuned Model?

**Decision:** Retrieval-Augmented Generation rather than a fully fine-tuned sports coaching LLM.

**Rationale:**
- A fine-tuned model would require large volumes of curated (data, coaching_report) pairs — this data does not exist at scale
- RAG allows the knowledge base to be updated independently of the model (add new tactic documents without retraining)
- Retrieval provides explicit citations — the system can show which strategies it is drawing from
- For a domain-specific application with a small but high-quality corpus, RAG outperforms general fine-tuning in factual accuracy
- Hallucination risk is lower when the model is anchored to retrieved documents

---

## 11. Ensemble Design: Why Two Models Rather Than One Large Model?

**Decision:** Use two independent models with ensemble fusion rather than one larger model.

**Rationale:**
- Two independently trained models with different architectures have uncorrelated failure modes
- When Model A fails (e.g., loses ball against white background), Model B (trained with different augmentation strategy) often succeeds
- Ensemble provides a natural confidence signal: high agreement = high confidence, disagreement = uncertain
- Total compute is 2× single model, but this is acceptable given the reliability improvement
- In deployment, the secondary model can be a smaller/faster architecture, keeping compute budget manageable

---

## Summary of Key Trade-Offs

| Decision | Benefit | Cost |
|---|---|---|
| Monocular | Universal applicability | Hard depth estimation problem |
| Segmentation | Spatially graded confidence | Slower than bounding box detection |
| Temporal triplets | Noise robustness | 1-frame look-ahead required |
| 13 keypoints | Robustness to occlusion | More detection parameters |
| LM calibration | Sub-pixel accuracy | 10ms refinement overhead |
| RoPE embeddings | Length generalization | Slightly more complex implementation |
| Multi-stage decoder | Stable training | More hyperparameters |
| MuJoCo synthetic | Scale up training data | Sim-to-real gap risk |
| 18 zones | Coaching-aligned granularity | Zone boundary edge cases |
| RAG coaching | Updatable, citable, low hallucination | Requires curated knowledge base |
