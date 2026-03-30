# Zone Grid Diagram — PingSight

## Zone Taxonomy

```
TABLE TENNIS STRATEGIC ZONE SYSTEM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Physical dimensions:
  Table length: 2.74m   (±1.37m from net)
  Table width:  1.525m  (±0.7625m from center)
  Table height: 0.76m   (world Z-coordinate of surface)

Zone grid division:
  3 rows × 3 columns × 2 sides = 18 zones + 1 out-of-bounds

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

     PHYSICAL COORDINATES (meters, top-down view)
     X-axis: left-right  (negative = left, positive = right)
     Y-axis: depth       (negative = far, positive = near)

     x: -0.762  -0.508  -0.254   0    0.254  0.508  0.762
        ╔══════╤═══════╤═══════╦═══════╤═══════╤═══════╗
y: -1.37║  a3' │  a2'  │  a1'  ║  a1   │  a2   │  a3   ║ y: 1.37
        ║      │       │       ║       │       │       ║
   -0.91╠══════╪═══════╪═══════╬═══════╪═══════╪═══════╣ 0.91
        ║  b3' │  b2'  │  b1'  ║  b1   │  b2   │  b3   ║
   -0.46╠══════╪═══════╪═══════╬═══════╪═══════╪═══════╣ 0.46
        ║  c3' │  c2'  │  c1'  ║  c1   │  c2   │  c3   ║
      0 ╚══════╧═══════╧═══════╬═══════╧═══════╧═══════╝ 0
                              NET (y=0)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ZONE LABELS
  Row 'a' = Baseline     (|y| ∈ [0.91, 1.37])
  Row 'b' = Middle       (|y| ∈ [0.46, 0.91])
  Row 'c' = Net          (|y| ∈ [0.00, 0.46])

  Col '1' = Center       (|x| ∈ [0.00, 0.254])
  Col '2' = Mid          (|x| ∈ [0.254, 0.508])
  Col '3' = Wide         (|x| ∈ [0.508, 0.762])

  Side ''  = Near player (y > 0)
  Side ''' = Far player  (y < 0)

  Special: '0' = Out of bounds (any coordinate outside table)
```

## Zone-Strategy Mapping

```
TACTICAL SIGNIFICANCE OF EACH ZONE

         FAR SIDE                             NEAR SIDE

  a3' ← Far wide baseline                a3 → Near wide baseline
        High-risk, high-reward                  Opponent's wide corner
        Crosscourt winner target                Classic crosscourt attack

  a2' ← Far mid baseline                 a2 → Near mid baseline
        Safe deep ball                          Most common attack zone
        Hard to attack from here               Benchmark 22% of shots

  a1' ← Far center baseline              a1 → Near center baseline
        Opponent strong position               Strong attack position
        Use to set up wide move               Set up crosscourt threat

  b3' ← Far wide middle                  b3 → Near wide middle
        Quick pivot zone for far player        Effective attack angle
        Angular attack available               Creates running opportunity

  b2' ← Far center middle                b2 → Near center middle
        High traffic zone                      High traffic zone
        Exchange zone in long rallies          Central attack/rally zone

  b1' ← Far center-mid                   b1 → Near center-mid
        Fast exchange zone                     Fast exchange zone
        Quick transition hub                   Quick response position

  c3' ← Far wide net                     c3 → Near wide net
        Wide serve target (sidespin)           Opponent wide serve lands here
        Forces difficult flip                  Difficult return position

  c2' ← Far center-short                 c2 → Near center-short
        Short serve target (backspin)          Short returns land here
        Forces push or flip                    Flip attack opportunity

  c1' ← Far center net                   c1 → Near center net
        Short center serve                     Net kill zone
        After opponent drops short             Fast flick position
```

## Visualization Color Scheme

```
Zone heat intensity scale:

  Cold  ░░░░  (0–5% frequency)
        ▒▒▒▒  (5–10%)
        ████  (10–20%)
  Hot   ████  (>20%)

Zone color by tactical significance:
  Net zones (c-row):      ORANGE  — high-pressure short balls
  Middle zones (b-row):   GREEN   — exchange / positioning
  Baseline zones (a-row): BLUE    — deep attack / baseline play

Active zone highlight: YELLOW border + brighter fill
```

## Sample Heatmap Legend

```
                ZONE HEATMAP EXAMPLE
                10-rally session analysis

    FAR SIDE              NEAR SIDE
    ┌────┬────┬────┐   ┌────┬────┬────┐
    │  5%│  8%│  4%│   │  7%│ 22%│  7%│  ← Baseline (a)
    ├────┼────┼────┤   ├────┼────┼────┤
    │  7%│ 12%│  9%│   │  9%│ 18%│ 11%│  ← Middle (b)
    ├────┼────┼────┤   ├────┼────┼────┤
    │  3%│  4%│  5%│   │  6%│ 12%│  8%│  ← Net (c)
    └────┴────┴────┘   └────┴────┴────┘
    left←  →right       left←  →right

    Total = 100% across all 18 zones

    Key insight: 40% concentration in near-side center column
```
