# NBA Draft 2026 — Energy Score Model

A college-to-NBA prospect evaluation pipeline that ranks the 2026 draft class against ten years of historical college players using a composite "energy" score.

Built in R, powered by [`cbbdata`](https://cbbdata.aaronyu.org/) (Bart Torvik data) with 2026 combine measurements integrated from the NBA Draft Combine.

---

## What is the Energy Score?

`energy` is a single composite metric designed to capture **how much a prospect's production is actually translatable to the next level**. It standardizes six inputs, weights them, and sums them — then rescales to the standard deviation of BPM so a 1-unit difference in energy is roughly comparable to a 1-point difference in college BPM.

### Inputs and weights

| Feature | Weight | What it captures |
|---|---|---|
| `youth` (negative est. age) | 30% | Younger producers project better; the dominant signal in any draft model |
| `conf_str` (10 if SEC/B12/B10/BE/Duke, else 5) | 20% | Crude strength-of-schedule adjustment |
| `ortg` (offensive rating) | 15% | Per-100 offensive efficiency |
| `usg` (usage rate) | 15% | Offensive responsibility — separates featured creators from finishers |
| `bpm` (box plus/minus) | 10% | Overall box-score impact |
| `ts` (true shooting %) | 10% | Scoring efficiency, FT- and 3PT-weighted |

All features are z-scored across the relevant pool before weighting, so a player's energy score is interpretable as "how many BPM-equivalent units above (or below) the pool average this player profiles."

### Why these weights?

They're **hand-tuned priors**, not derived from a regression on NBA outcomes. The 50% combined weight on youth + conference reflects two observations from the historical pool: (1) age is the single most important predictor of NBA translation in essentially every public draft model, and (2) production at low-major schools is meaningfully discounted by NBA front offices. Production features (BPM, ORTG, USG, TS) split the remaining 50% roughly evenly to avoid double-counting overlapping signal — BPM already contains efficiency and usage information, so loading it heavily risks counting the same skill three times.

This is the model's most legitimate critique. See **Limitations** below.

---

## How comps work

For each 2026 prospect, the pipeline finds the closest historical comparable from the 2015–2025 pool by:

1. Filtering historical players within ±1 year of the prospect's estimated age
2. Selecting the player with the smallest absolute energy-score gap

Comps are intentionally rough — they're a sanity check on the energy score, not a prediction. "Player X has an energy score of 4.2, which is roughly where Player Y (also 19, also a B12 wing) was in 2021" is a more honest framing than implying career outcomes.

---

## Data

### Sources
- **Current prospects** (`data/prospects_2026`) — 2026 draft class roster with team, conf, listed height, class year
- **Combine measurements** (`data/combine_measurements_2026.csv`) — 75 players from the 2026 NBA Draft Combine: height (no shoes), wingspan, standing reach, weight, hand size
- **Historical pool** — Top 100 college players by BPM per season from 2015–2025, pulled via `cbd_torvik_player_season()`. Filtered to ≥15 games and ≥10 minutes per game. 2026 prospects are excluded from this pool so they don't comp against themselves.

### Pipeline order
1. Pull 2026 prospects from GitHub
2. Merge combine measurements (overrides listed height when available)
3. Pull historical seasons via `cbbdata`
4. Compute energy scores (separate z-scoring for each pool)
5. Bucket prospects into PG / SG / Wing / C by height
6. Format heights as `6' 3.5"` for display
7. Generate Excel workbook with per-position tabs + overall + historical sheets
8. Generate top-1 comp table

---

## Position buckets

Done by combine height when available, listed height otherwise:

| Group | Height (no shoes, in.) |
|---|---|
| PG | ≤ 75 |
| SG | 75.25 – 78 |
| Wing | 78.25 – 82 |
| C | ≥ 83 |

Note that combine heights are without shoes and run roughly 0.75–1.5" shorter than listed heights with shoes. Players who only have a listed height may end up in a different bucket than they would have post-combine.

---

## Limitations (read these)

Any draft model trying to compete with NBA front offices is starting from a deficit. Being honest about what this one can and can't do:

- **No defensive signal beyond DBPM.** DBPM is noisy and box-score driven. No stocks-adjusted rate, no on/off, no opponent quality of competition. A defense-first wing is systematically undervalued here.
- **No shot diet.** Rim rate, 3PA rate, free throw rate, off-the-dribble vs. catch-and-shoot — all missing. A high-TS spot-up shooter looks identical to a high-TS shot creator in this model. They are not the same NBA bet.
- **Weights are hand-tuned**, not learned from NBA outcomes. A proper next step would be training a gradient-boosted classifier on `made_nba`, `became_rotation`, or `career_bpm` and comparing the learned feature importances against the priors here.
- **No historical validation yet.** The model has not been backtested against actual NBA outcomes. The 814-player historical pool is sitting right there — pairing it with hoopR career data and computing correlation between college energy and NBA BPM is the obvious next step.
- **Age is estimated, not actual.** Class year (Fr/So/Jr/Sr/Gr) maps to 18/19/20/21/22. A 19-year-old freshman and a 17-year-old freshman look identical to the model. International prospects with unusual paths are most affected.
- **Top-1 nearest neighbor is a thin comp.** Top-5 with similarity scores would be more honest. Showing whether each comp made the NBA, became a rotation player, or busted would be more useful still.
- **Conference strength is binary.** SEC/B12/B10/BE/Duke = 10, everything else = 5. KenPom or Torvik conference adjustments would be a clear upgrade.

---

## Repo layout
