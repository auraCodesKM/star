# Sample Outputs — PingSight

> Real outputs from the system running on WTT broadcast footage: Frankfurt (Lebrun vs Moregard), Incheon, and Fukuoka (Wang Chuqin vs Harimoto).

## 1. Video Annotation Output

![Annotated Frame](../assets/demo/annotated_frame.png)
*Zone-aware live frame — Wang Chuqin vs Tomokazu Harimoto, Fukuoka. Left: broadcast with bounce tracking overlay. Right: 3D physics model with active zone highlighted.*

![Full Pipeline Output](../assets/demo/full_pipeline_output.png)
*3D-uplifted trajectory reprojected onto WTT Incheon broadcast footage. Dotted green arc = recovered ball path across the rally.*

### Frame Annotation Elements
Each output frame contains:

```
┌──────────────────────────────────────────────────────────────┐
│  [Original video frame]                                       │
│                                                               │
│  ○ ○ ○ ○ ●  ← Ball trail (12-frame comet tail)              │
│            ↑                                                  │
│         Current ball position                                 │
│                                                               │
│  ┌─────────────────────────────────┐                         │
│  │  TABLE ZONE GRID OVERLAY        │                         │
│  │  c3│c2│c1 ║ c1'│c2'│c3'        │                         │
│  │  b3│b2│b1 ║ b1'│b2'│b3'        │                         │
│  │  a3│a2│a1 ║ a1'│a2'│a3'        │  ← Active zone (orange) │
│  └─────────────────────────────────┘                         │
│                                                               │
│  Zone: b2  |  Prev: c1'  |  Next: a3                         │
│  Spin: topspin 89rps                                          │
│  Z: 0.76m (BOUNCE)                                           │
└──────────────────────────────────────────────────────────────┘
```

### 3D Subplot (side panel)
```
     Z (m)
  1.5 │    .
      │   . .
  1.0 │  .   .
      │ .     .●  ← current
  0.76│─────────────── TABLE SURFACE
      │
  0.0 └──────────────── Y (m)
     -1.37          0          1.37
```

---

## 2. Zone System Reference

![Zone Grid](../assets/demo/zones.png)
*Tactical zone substitution table and 9-zone grid layout (per side). Out-of-bounds = code 0.*

## 3. Zone Frequency Heatmap

### Sample Output: 10-Rally Analysis

```
ZONE FREQUENCY HEATMAP — NEAR SIDE (bounce count / total)

         ← BACKHAND    CENTER    FOREHAND →
         COL 3         COL 2     COL 1
         ┌────────────┬─────────┬────────────┐
NET  c   │    8%      │   12%   │    6%      │
         ├────────────┼─────────┼────────────┤
MID  b   │   11%      │   18%   │    9%      │
         ├────────────┼─────────┼────────────┤
BASE a   │    7%      │   22%   │    7%      │
         └────────────┴─────────┴────────────┘

KEY INSIGHT: 40% of opponent attacks target center column (b2/a2).
             Cross-court to a3 severely underutilized (7%).
```

---

## 4. Spin Distribution Summary

```
====================================================
  SPIN ANALYSIS — Session: training_day_07.mp4
  Rallies: 47  |  Total strokes: 284
====================================================

  TOPSPIN
  ████████████████████████████████████░░░░   61%  avg 89 rps
  Range: [42, 134 rps]

  BACKSPIN
  ██████████████░░░░░░░░░░░░░░░░░░░░░░░░░░   28%  avg 47 rps
  Range: [18, 89 rps]

  SIDESPIN
  █████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░   11%  avg 58 rps
  Range: [22, 97 rps]

====================================================
  NOTE: High-speed topspin (>85 rps) detected in 34% of
        third-ball attacks. Backspin defense consistency
        drops 23% against these shots.
====================================================
```

---

## 5. Trajectory 3D Reconstruction

![Rally 3D Reconstruction](../assets/demo/rally_3d_reconstruction.png)
*WTT Frankfurt — Felix Lebrun vs Truls Moregard. Left: original broadcast. Right: recovered 3D ball positions in world coordinates.*

![Rally 3D Reconstruction GIF](../assets/demo/rally_3d_reconstruction.gif)
*Animated 3D reconstruction of a complete rally — watch the trajectory build frame by frame.*

| Rotating trajectory view | Split-view animation |
|:---:|:---:|
| ![3D anim](../assets/demo/3d_trajectory_animation.gif) | ![split](../assets/demo/3d_trajectory_split_animation.gif) |
| *Isolated ball trajectory in world coordinates.* | *Table surface + full 3D flight path, side by side.* |

### Sample Rally — 8 Stroke Exchange

```
Time  Event           Zone     X(m)   Y(m)   Z(m)   Spin
──────────────────────────────────────────────────────────
0.00  Serve            –       0.00   0.80   0.93   –
0.28  Bounce (far)     c2'     0.15  -0.38   0.76   top 87rps
0.52  Bounce (near)    a1      0.08   1.05   0.76   back 42rps
0.78  Bounce (far)     b3'     0.52  -0.72   0.76   side 65rps
1.02  Bounce (near)    b1      0.04   0.58   0.76   top 91rps
1.28  Bounce (far)     c1'     0.09  -0.25   0.76   top 103rps
1.51  Bounce (near)    a2      0.28   0.92   0.76   back 38rps
1.73  Net error        –       0.00   0.00   0.84   –
──────────────────────────────────────────────────────────
Rally duration: 1.73s  |  Strokes: 8  |  Winner: far player
```

---

## 6. Transition Pattern Analysis

### Top 5 Observed Zone Triplets (prev → curr → next)

```
Rank  Pattern              Frequency  Interpretation
────────────────────────────────────────────────────
 1    c1' → a1 → c2'          18%    Fast exchange, center
 2    b2' → b2 → b2'          14%    Cross-table rally
 3    c3' → a3 → c1'          11%    Wide to center pivot
 4    a1' → a1 → a1'           9%    Deep baseline exchange
 5    b1' → c1 → a2'           7%    Net attack recovery
────────────────────────────────────────────────────

EXPLOITATION INSIGHT:
Pattern #3 (c3' → a3 → c1') shows the opponent consistently
responds to wide balls by returning to center (c1').
Tactic: After forcing to a3, expect center return —
position early for forehand attack.
```

---

## 7. Detection Pipeline Comparison

| Stage | Raw | Filtered |
|:---:|:---:|:---:|
| **Ball** | ![rb](../assets/detection/raw_ball_detections.png) | ![fb](../assets/detection/filtered_ball_detections.png) |
| **Table** | ![rt](../assets/detection/raw_table_keypoints.png) | ![ft](../assets/detection/filtered_table_keypoints.png) |

**2D→3D uplifting stage:**

| Input to uplifting | Reprojection validation | Debug overlay |
|:---:|:---:|:---:|
| ![2d](../assets/uplifting/2d_input_to_uplifting.png) | ![rep](../assets/uplifting/2d_reprojection_from_3d.png) | ![dbg](../assets/demo/debug_overlay.png) |

## 8. Sample Coaching Report

```
══════════════════════════════════════════════════════════════
  PINGSIGHT TACTICAL REPORT
  Session: training_day_07.mp4  |  Date: 2025-11-14
  Rallies analyzed: 47  |  Total duration: 38 minutes
══════════════════════════════════════════════════════════════

PATTERN ANALYSIS
─────────────────
Your shot placement is heavily center-weighted: 52% of all
shots target the center two columns (b2, a2, c2). This
predictability makes you readable to experienced opponents.

Your third-ball attack success rate is high (71%) when serving
to c3' (wide backhand), but you only use this serve 12% of
the time — well below optimal strategic frequency.

Spin variety is limited: 61% topspin, 28% backspin, 11%
sidespin. Opponents adapting to heavy topspin after the
second ball account for 40% of your point losses.

EXPLOITATION OPPORTUNITIES
────────────────────────────
1. WIDE FOREHAND GAP: Opponent leaves forehand corner (a3)
   open 73% of the time after your diagonal serve. A cross-
   court attack to a3 after c1 serve could yield high points.

2. BACKHAND OVERCOMMITMENT: After your topspin to b2,
   opponent commits backhand 81% of the time. A pivot loop
   to their forehand (b3/a3) is available but unused.

3. SHORT GAME: Only 6% of your serves land in c-row zones.
   Short serves to c1/c2 force opponent into difficult
   positions — consider 25% short serve frequency.

TRAINING RECOMMENDATIONS
─────────────────────────
1. PLACEMENT DIVERSIFICATION DRILL
   Practice 20-minute multiball sessions targeting a3 and
   b3 zones exclusively. Goal: ≥20% shot placement outside
   center column by session 4.

2. SIDESPIN SERVE DEVELOPMENT
   Introduce pendulum sidespin serves targeting c3' with
   follow-up to wide forehand. Drill at 3 sets × 30 balls.

3. COUNTER-TOPSPIN DEFENSE
   Your backspin block success rate drops 23% against >85rps
   topspin. Add dedicated loop-block exchange drills at
   increasingly high spin rates (60 → 80 → 100 rps targets).

══════════════════════════════════════════════════════════════
  Confidence: HIGH (n=47 rallies, good data quality)
  Next session: Focus on Recommendations #1 and #3
══════════════════════════════════════════════════════════════
```

---

## 9. Output File Summary

| File | Description |
|---|---|
| `analysis_normal.mp4` | Full annotated video at original speed |
| `analysis_slow.gif` | 0.3× slow-motion highlight GIF |
| `zone_heatmap.png` | Per-side bounce frequency heatmap |
| `spin_distribution.png` | Spin type breakdown histogram |
| `trajectory_3d.gif` | Rotating 3D trajectory animation |
| `transition_matrix.png` | Zone-to-zone probability heatmap |
| `coaching_report.txt` | Full text coaching report |
| `rally_data.json` | Structured rally data (all bounces, zones, spin) |
