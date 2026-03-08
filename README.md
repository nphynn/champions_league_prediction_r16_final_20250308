# UEFA Champions League 2025/26 — Knockout Stage Prediction

A data-driven approach to predicting the 2025/26 UCL Round of 16 through to the Final using composite strength scoring, logistic win probabilities, and Monte Carlo bracket simulation across three independently constructed datasets.

## Objective

Predict the outcome of all 8 Round of 16 ties and propagate winners through Quarter-Finals, Semi-Finals, and the Final using a fixed knockout bracket. 

## Data Sources

**Domestic league stats** (as of matchday 24–30 depending on league): Goals Scored (GF), Goals Conceded (GA), Points, and Games Played for all 16 teams across 8 leagues (Premier League, La Liga, Serie A, Bundesliga, Ligue 1, Primeira Liga, Süper Lig, Eliteserien).

**Champions League group/league phase stats**: 8 matches per team — GF, GA, Points.

**UEFA Country Coefficients (2025/26)**: England (22.29), Germany (17.57), Spain (17.41), Italy (17.36), Portugal (16.60), France (14.96), Turkey (10.28), Norway (10.28 - boosted to account for team improvement in play).

## Methodology

### Phase 1 — Data Generation (Three Datasets)

Three datasets were constructed with each merging domestic and CL performance under different assumptions about domestic league strength. All three share the same output schema: GF per game, GA per game, and Points Percentage.

**Dataset A (Raw Merged)** (applies no league-strength adjustment): Domestic and CL stats are summed directly and normalized by total games played. Every goal counts equally regardless of league. This serves as the control.

**Dataset B (UEFA Coefficient-Weighted)** (scales domestic stats by normalized UEFA country coefficients with EPL = 1.0 baseline): Offensive metrics (GF, Points_Pct) are multiplied by the weight; GA is divided by it, penalizing conceding in weaker leagues. CL stats remain raw. The coefficient range spans 0.46 to 1.00 meaning a team in the weakest league retains less than half their domestic output.

**Dataset C (Compressed Prior)** applies the same coefficient logic as Dataset B but compresses the weight range to 0.70–1.00 using linear rescaling: `weight = 0.70 + 0.30 × (coeff − min) / (max − min)`. This preserves league-quality signal without letting it dominate ensuring every team retains at least 70% of their domestic output.

### Phase 2 — Visualization

Chart types were produced per dataset to validate the data before modelling:

### Phase 3 — Strength Scoring & Bracket Simulation

#### Normalization: Median−1σ Floor

Standard z-score normalization was initially tested but produced excessively wide strength gaps (Δ ≈ 3.5) resulting in 95%+ win probabilities for favourites which is unrealistic for knockout football. Standard min-max (0 to 1) was also rejected because it maps the weakest team to exactly zero which is unfair (e.g., Tottenham's Points_Pct becoming 0.0).

Five normalization methods were evaluated: proportion of max, arbitrary floored min-max, mean-anchored floor, log-transform, and median−1σ floor. The selection criterion was realism: the most lopsided matchup should produce ≤75% win probability and mid-table matchups should stay around 55–60%, consistent with historical UCL knockout upset rates (underdogs win ~30–40%).

**Median−1σ floor was selected.** For each feature, the floor is computed as `floor = clamp((median − std) / max, 0.05, 0.5)`, then the feature is scaled to `[floor, 1.0]`. This is statistically grounded (adapts to each feature's distribution) producing a max win probability of ~66%, and preserves enough separation for genuinely stronger teams to have a meaningful edge.

#### Composite Strength Score

Features are combined as: `Strength = 0.30 × GF_norm + 0.35 × (1 − GA_norm) + 0.35 × Points_Pct_norm`. GA is flipped so that lower conceding yields a higher contribution. Defence and results receive slightly more weight (0.35 each) than pure attack (0.30), reflecting the historical importance of defensive solidity in knockout football.

#### Logistic Win Probability

For each tie: `P(A wins) = 1 / (1 + e^(−k × Δ))` where `Δ = Strength_A − Strength_B` and `k = 1.5`. At Δ = 0 it is 50/50; at the maximum observed gap (~0.44). 

#### Monte Carlo Bracket Simulation (10,000 runs)

Rather than deterministically advancing the stronger team (which was rejected as unrealistic), each tie is resolved probabilistically: a random draw weighted by the logistic win probability determines who advances. The full bracket is simulated 10,000 times per dataset, and separately for an ensemble (averaged strength across all three datasets). This produces tournament win probability distributions rather than a single fixed prediction.

### Phase 4 — Cross-Dataset Comparison & Final Prediction

**Agreement check**: For each tie, compare which team is favoured across all three datasets. Full agreement = high confidence; 2-1 split = moderate; divergence = assumption-sensitive.

**Weight sensitivity analysis**: Weights (w_gf, w_ga, w_pct) were varied across 36+ combinations (0.15 to 0.50, step 0.05, sum = 1.0), with 2,000 simulations per combo per dataset. Teams that win frequently across many weight configurations are robust predictions.

**Ensemble bracket**: Strength scores averaged across all three datasets, then a fresh 10,000-run simulation to produce consensus predictions.

**Final score**: `Final = 0.40 × Ensemble + 0.30 × Sensitivity + 0.30 × Avg(A,B,C)`.


## Predicted Bracket (Most Likely Path)

```
ROUND OF 16              QUARTER-FINALS        SEMI-FINALS          FINAL
═══════════════════════════════════════════════════════════════════════════

           PSG ─┐
                 ├─ PSG ──────────┐
       Chelsea ─┘                 │
                                  ├─ PSG ──────────┐
   Galatasaray ─┐                 │                 │
                 ├─ Galatasaray ──┘                 │
     Liverpool ─┘                                   │
                                                    ├─ Bayern ─────┐
   Real Madrid ─┐                                   │              │
                 ├─ Man City ─────┐                 │              │
      Man City ─┘                 │                 │              │
                                  ├─ Bayern ────────┘              │
      Atalanta ─┐                 │                                │
                 ├─ Bayern ───────┘                                │
  Bayern Munich ─┘                                                 │
                                                               🏆 ARSENAL
     Newcastle ─┐                                                  │
                 ├─ Barcelona ────┐                                │
    Barcelona ─┘                  │                                │
                                  ├─ Barcelona ─────┐              │
Atletico Madrid ─┐                │                 │              │
                  ├─ Atlético ────┘                 │              │
     Tottenham ─┘                                   │              │
                                                    ├─ Arsenal ────┘
  FK Bodø/Glimt ─┐                                  │
                   ├─ Sporting CP ─┐                │
    Sporting CP ─┘                 │                │
                                   ├─ Arsenal ──────┘
 Bayer Leverkusen ─┐               │
                    ├─ Arsenal ────┘
        Arsenal ─┘
```

## Results

### Tournament Win Probabilities (Final Composite Score)

| Rank | Team | DS A | DS B | DS C | Ensemble | Sensitivity | Final |
|------|------|------|------|------|----------|-------------|-------|
| 🥇 | Bayern Munich | 11.5% | 10.5% | 10.5% | 11.2% | 11.2% | **11.1%** |
| 🥈 | Arsenal | 9.5% | 12.0% | 11.9% | 10.9% | 10.9% | **11.0%** |
| 🥉 | Sporting CP | 9.2% | 8.8% | 9.9% | 9.5% | 9.3% | **9.4%** |
| 4 | Barcelona | 8.9% | 7.6% | 8.6% | 8.5% | 8.3% | **8.4%** |
| 5 | Man City | 5.7% | 7.6% | 6.9% | 6.9% | 6.9% | **6.8%** |
| 6 | Real Madrid | 6.7% | 6.6% | 6.1% | 6.5% | 6.4% | **6.4%** |
| 7 | PSG | 6.9% | 5.0% | 6.1% | 6.1% | 6.2% | **6.1%** |
| 8 | Atletico Madrid | 6.0% | 5.7% | 5.6% | 5.8% | 5.6% | **5.7%** |
| 9 | Chelsea | 4.6% | 6.9% | 5.4% | 5.7% | 5.5% | **5.6%** |
| 10 | Liverpool | 4.2% | 6.5% | 5.1% | 5.5% | 5.5% | **5.4%** |
| 11 | Galatasaray | 7.8% | 3.5% | 5.4% | 5.2% | 5.2% | **5.3%** |
| 12 | FK Bodø/Glimt | 6.1% | 2.6% | 4.4% | 4.1% | 4.2% | **4.2%** |
| 13 | Newcastle | 3.3% | 4.8% | 3.9% | 4.0% | 4.1% | **4.0%** |
| 14 | Atalanta | 3.4% | 3.9% | 3.7% | 3.3% | 3.7% | **3.5%** |
| 15 | Bayer Leverkusen | 3.5% | 3.7% | 3.2% | 3.6% | 3.5% | **3.5%** |
| 16 | Tottenham | 2.7% | 4.4% | 3.3% | 3.4% | 3.5% | **3.4%** |

The competition is genuinely open: the top 4 account for ~40% of outcomes, and even the lowest-ranked team (Tottenham, 3.4%) has a realistic path through the bracket.

### Why Bayern Leads the Table but Arsenal Wins the Bracket

These two outputs answer different questions and should not be confused.

The **Final Composite Score table** above aggregates 10,000+ simulations across all three datasets plus a weight sensitivity sweep. It answers: "across all possible bracket outcomes, who wins the tournament most often?" Bayern Munich leads at 11.1% because their high strength score gives them favourable odds in every possible matchup they could face — regardless of which specific path they take.

The **Predicted Bracket** shown further below answers a different question: "if we follow the single most likely winner at every tie, who emerges?" This is a chain of conditional outcomes where each round's matchup depends on who won the previous round. Arsenal wins this path because while their individual matchup margins are modest (62% vs Leverkusen, 51% vs Sporting CP in the QF, 55% vs Barcelona in the SF), their bracket half is slightly more favourable. Bayern, despite having a higher overall tournament win rate, faces a tighter semi-final and a lower probability of reaching the final via their specific bracket path.

In short: Bayern is the most likely champion across all simulated scenarios. Arsenal is the most likely champion if you follow the single most probable outcome at each fork. Both are valid — the 0.1% gap between them (11.1% vs 11.0%) confirms they are effectively co-favourites.

### R16 Upset Watch

| Tie | Margin | Closeness |
|-----|--------|-----------|
| Galatasaray vs Liverpool | 0.1% | 🔴 Coin flip |
| Real Madrid vs Man City | 1.8% | 🔴 Coin flip |
| PSG vs Chelsea | 2.5% | 🔴 Coin flip |
| Atletico Madrid vs Tottenham | 7.1% | 🔴 Coin flip |
| Newcastle vs Barcelona | 13.8% | 🟡 Upset possible |
| Bodø/Glimt vs Sporting CP | 17.0% | 🟡 Upset possible |
| Atalanta vs Bayern Munich | 23.3% | 🟢 Favourite clear |
| Leverkusen vs Arsenal | 24.4% | 🟢 Favourite clear |

Four of eight R16 ties are essentially coin flips (<10% margin), confirming that the Median−1σ normalization produces realistically competitive matchups.

## Key Technical Decisions

| Decision | Issue | Resolution |
|----------|-------|------------|
| Ridge regression | No labelled outcomes to train on; model would just fit features to themselves | Replaced with composite strength scoring |
| Z-score normalization | Produced 95%+ win probabilities; unrealistic | Replaced with Median−1σ floored min-max |
| Min-max [0, 1] | Worst team per feature maps to 0 (zero contribution) | Rejected; floored approach preserves baseline value |
| Deterministic bracket | Stronger team always advances; no upset mechanism | Replaced with probabilistic Monte Carlo sampling (10K runs) |
| Arbitrary floor (0.10) | Not data-driven; chosen without justification | Replaced with statistical floor: `(median − σ) / max`, clamped [0.05, 0.5] |
| k steepness (1.5) | Controls logistic curve sensitivity | Validated: max matchup P ≈ 66%, consistent with real knockout upset rates |


## Tech Stack

Python 3, pandas, NumPy, SciPy (zscore), matplotlib. No ML libraries required — the approach is entirely statistical and simulation-based.


## Limitations

- Domestic data is a snapshot at one point in the season; form fluctuates.
- The model uses only three features (GF/g, GA/g, Points_Pct). Factors like squad depth, injuries, head-to-head history, home/away legs, and managerial tactics are not captured.
- UEFA coefficients are country-level, not club-level. A mid-table EPL team gets the same coefficient as the league leader.
- The logistic k parameter (1.5) and feature weights (0.30/0.35/0.35) are reasonable defaults, not optimised against historical knockout outcomes.

## Author
[Nick Phynn](https://www.linkedin.com/in/nick-phynn-928354b4/) & [Romario Bennett](https://www.linkedin.com/in/romario-bennett-48b648220/)
