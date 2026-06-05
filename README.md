# Energy Score Model

A college basketball prospect-evaluation pipeline built around a composite "energy" score. The same framework now powers **two boards**:

1. **2026 NBA Draft class** — ranked against ten years of historical college players.
2. **2026 Transfer Portal** — ranking both already-signed and still-available transfers, so a staff can see at a glance who's still gettable.

Built in R, powered by [`cbbdata`](https://cbbdata.aaronyu.org/) (Bart Torvik data), with 2026 combine measurements integrated from the NBA Draft Combine for the draft class.

---

## What is the Energy Score?

`energy` is a single composite metric designed to capture a prospect's production. It standardizes six inputs, weights them, and sums them — then rescales to the standard deviation of BPM so a 1-unit difference in energy is roughly comparable to a 1-point difference in college BPM. The same scoring function drives both the draft and the transfer-portal pipelines.

### Inputs and weights

| Feature | Weight | What it captures |
|---|---|---|
| `youth` (younger class → higher) | 30% | Younger producers project better |
| `conf_str` (10 if ACC/SEC/B12/B10/BE, else 5) | 20% | Strength-of-schedule adjustment |
| `ortg` (offensive rating) | 15% | Per-100 offensive efficiency |
| `usg` (usage rate) | 15% | Offensive responsibility — separates featured creators from finishers |
| `bpm` (box plus/minus) | 10% | Overall box-score impact |
| `ts` (true shooting %) | 10% | Scoring efficiency, FT- and 3PT-weighted |

All features are z-scored across the relevant pool before weighting, so a player's energy score is interpretable as "how many BPM-equivalent units above (or below) the pool average this player profiles." The weights sum to 1.00, so the composite is a balanced blend rather than any one feature running away with the score.

### Why these weights?

They're **hand-tuned priors.** The 50% combined weight on youth + conference reflects two observations from the historical pool: (1) age is the single most important predictor of NBA translation in essentially every public draft model, and (2) production at low-major schools is meaningfully discounted by NBA front offices. Production features (BPM, ORTG, USG, TS) split the remaining 50% roughly evenly to avoid double-counting overlapping signal.

---

## Data

### Sources
- **Current prospects** (`data/prospects_2026`) — 2026 draft class roster with team, conf, listed height, class year
- **Combine measurements** (`data/combine_measurements_2026.csv`) — 75 players from the 2026 NBA Draft Combine: height (no shoes), wingspan, standing reach, weight, hand size
- **Signed transfers** (`data/transfer`) — players who have **already committed** to a new school; carries both the new `team` and the `old_team` where the stats were earned
- **Unsigned portal** (`data/unsigned.csv`) — players **still in the portal / uncommitted**; carries `old_team` only (destination not yet known)
- **Historical pool** — Top 100 college players by BPM per season from 2015–2025, pulled via `cbd_torvik_player_season()`. Filtered to ≥15 games and ≥10 minutes per game. 2026 prospects are excluded from this pool so they don't comp against themselves.

### How conferences are assigned for transfers
Because a transfer's stats were earned at their **old** school, `conf_str` is keyed on `old_team`, not the destination. The team → conference map is built once from the historical Torvik pull (most-recent conference per team, so realignment is reflected) and joined onto each transfer set. Teams that never crack the top-100 pull are mid-majors by definition and correctly default to the non-power value of 5.

### NBA Draft pipeline order
1. Pull 2026 prospects from GitHub
2. Merge combine measurements (overrides listed height when available)
3. Pull historical seasons via `cbbdata`
4. Compute energy scores (separate z-scoring for each pool)
5. Bucket prospects into PG / SG / Wing / C by height
6. Format heights as `6' 3.5"` for display
7. Generate Excel workbook with per-position tabs + overall + historical sheets
8. Generate top-1 comp table


---

## Transfer Portal Rankings

The portal board reuses the exact draft energy metric — same six features, same weights — applied to the 2026 transfer market. A few details that help when reading the output:

- **Two views in one workbook.** The first sheet ranks transfers who've already signed (where they landed is shown in `team`); the second ranks the players still available. Sorting the second sheet by `energy` gives an immediate, prioritized "who's left worth chasing" board.
- **Already-signed players are filtered out of the available board automatically**, so the list a staff acts on never includes someone who's off the market.
- **`youth` is taken straight from class year** (Fr / So / Jr / Sr) for the portal, shown as the `class` column rather than a derived age — same ordering, just more readable for a basketball reader.
- **Conference strength reflects where the production happened** (the old school), which is the relevant context when projecting how a transfer's numbers will hold up at a new level.

---

## Position buckets

Done by combine height when available, listed height otherwise (transfer files already provide numeric `height_in`):

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

- **No defensive signal beyond DBPM.** DBPM is noisy and box-score driven. A defense-first wing is systematically undervalued here.
- **A proper next step would be training a gradient-boosted classifier on `made_nba`, `became_rotation`, or `career_bpm`** and comparing the learned feature importances against the priors here.
- **Age is estimated, not actual.** Class year (Fr/So/Jr/Sr/Gr) maps to 18/19/20/21/22 in the draft pipeline. A 19-year-old freshman and a 17-year-old freshman look identical to the model. International prospects with unusual paths are most affected.

---

## Repo layout

```
.
├── nba_draft_2026.Rmd                    # draft-class energy pipeline
├── transfer_portal_energy_rankings.Rmd   # transfer-portal energy pipeline
├── data/
│   ├── prospects_2026                     # 2026 draft class
│   ├── combine_measurements_2026.csv      # 2026 combine measurements
│   ├── transfer                           # already-signed transfers
│   └── unsigned.csv                       # still-available portal players
└── README.md
```

---

## Reproducing

```r
# Install deps
install.packages(c("cbbdata", "tidyverse", "janitor", "psych", "writexl"))

# Get a cbbdata account at https://cbbdata.aaronyu.org/, then:
library(cbbdata)
cbd_login()

# Draft board:
rmarkdown::render("nba_draft_2026.Rmd")

# Transfer-portal board (signed + available):
rmarkdown::render("transfer_portal_energy_rankings.Rmd")
```

Each script writes a multi-sheet Excel workbook at the path defined in its export chunk. Both run off the same `cbd_login()` session.

---

## Roadmap

In rough priority order:

1. **Historical validation** — join the historical pool against NBA career stats, compute correlation between college energy and career BPM, publish the scatter plot in this README
2. **Top-5 comps with similarity scores and NBA outcomes** — replace the top-1 comp with a richer comp table
3. **Defensive features** — STL%, BLK%, DREB%, height-adjusted block rate for bigs
4. **Shot diet features** — rim rate, 3PA rate, FTr
5. **Learned weights** — gradient-boosted model trained on NBA outcomes, compare feature importances against hand-tuned priors
6. **Better conference adjustment** — KenPom or Torvik adjusted conference strength rather than binary power/non-power
7. **Fit scoring for the portal** — layer team-need and stylistic fit on top of raw energy so the available board can be re-sorted per program

---

## Acknowledgments

- [Bart Torvik](https://barttorvik.com/) for the underlying college basketball data
- [Aaron Yu](https://cbbdata.aaronyu.org/) for `cbbdata`
- NBA.com for combine measurements
