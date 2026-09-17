# Report outline — structure, evidence, and what each section must argue

Working scaffold. The **Evidence** column names the artefact that supports each
claim, so nothing in the report is asserted without something to point at.
Findings are filled in once the experiments have run.

Target: concise. Aim for 8–12 pages. The rubric rewards interpretation, not volume —
"do not include outputs simply because they were generated."

---

## 1. Introduction  *(~½ page)*

- The problem: mobile operators must allocate radio and backhaul capacity ahead of
  demand; under-provisioning degrades service, over-provisioning wastes capital.
- Why one-step-ahead at 10 minutes is the operationally relevant horizon.
- State the research question verbatim from the brief.
- State the contribution: a controlled comparison of three model families on a
  common protocol, with the naive baseline reported — which the ACM Computing
  Surveys review [3] identifies as missing from much of the literature.

## 2. Related Work  *(~1 page)*

Source: `docs/RELATED_WORK.md` — 17 annotated references.

Do **not** write a list of paper summaries. Organise by the decision each body of
work informed:

| Sub-argument | Sources | Decision it justifies |
|---|---|---|
| The Milan grid is the field's benchmark | [1], [2] | Comparability of our results |
| CNN/RNN families dominate | [2], [5] | LSTM as the expected choice |
| TCNs match RNNs while training faster | [7], [8] | TCN as the informative contrast |
| Long seasonal periods need differencing or harmonics | [9], [14] | SARIMA at s=144 |
| Spatial models need the grid | [4] | Why we do *not* attempt one — and future work |
| Fair benchmarking is scarce | [3] | Identical protocol across models |

Include the "deliberately not used" table — the brief asks for criticism of model
choices, and defensible exclusions are part of that.

## 3. Dataset and Data Preparation  *(~1 page)*

| Claim | Evidence |
|---|---|
| Scale and schema | `README.md` §1; 62 files, 19.38 GiB, 4,842,625 rows/day |
| Country code is a nuisance dimension | 4,842,625 → 1,439,982 rows; 99.999% grid occupancy |
| Naive ingestion does not fit | notebook 1, section 4 output — 17.9 GB projected vs 16 GB RAM |
| Optimisation is effective | Same table: 295.6 → 27.5 MB/day, and flat in day count |
| The chunk size was chosen, not guessed | notebook 1, section 5b output — 18 MB at 250k vs 605 MB at 2M, throughput flat |
| Timestamps are handled correctly | Verified: `T0 = 1383260400000` = 2013-11-01 00:00 CET; no DST transition in window |

State the trade-off explicitly: country-level detail is discarded and unrecoverable
without re-downloading. Justify it.

## 4. Exploratory Analysis  *(~2 pages — the largest rubric block at 13 + 12 pts)*

**4.1 Spatial distribution (Task 2.1)** — `figures/task2_1_traffic_distribution.png`,
`figures/task2_1_spatial_map.png`, notebook 2 summary table.
Argue from the Gini coefficient and the log-scale panel: is the distribution
log-normal-like? What does that imply about generalising a model across areas?

**4.2 The five areas (Task 2.2)** — `figures/task2_2_first_two_weeks.png` and the shape-stats table.
Compare *shape*, not just level. Peak/trough ratio, weekend/weekday ratio, peak
hour. Connect differences to plausible land use (business district vs residential
vs transport hub). Flag anything anomalous.

**4.3 Temporal structure (Task 2.3)** — decomposition and autocorrelation figures.
This is where the modelling decisions get their justification:

| Observation | Consequence for the models |
|---|---|
| ACF at lag 1 | How strong is persistence? Sets the bar. |
| ACF at lag 144 vs lag 1 | Does a model need a full day of history, or is this local? |
| MSTL variance shares | How much is *not* calendar-driven — i.e. what is left to learn |
| ADF vs KPSS across differencing schemes | Which differencing SARIMA needs |
| Weekday/weekend profiles | Whether one model can serve both day types |

## 5. Methodology  *(~1½ pages)*

- Forecasting protocol: rolling one-step-ahead with observed history. Say
  explicitly that errors do not compound, and that this is *why* persistence is
  competitive.
- Splits, and the tuning split that keeps the reported week untouched.
- Input representation per model — the brief asks for this explicitly (4.VI):
  sequence length, preprocessing, normalisation, and that the scaler is fitted on
  training targets only.
- The three models and their justification (pull from `docs/RELATED_WORK.md`).
- SARIMA's s=144 problem and its resolution — this is a strong, concrete
  methodological decision to discuss in the viva: 146 state dimensions, cubic
  Kalman update, >10 min per fit; explicit differencing brings it to 2 states.
- The tuning strategy: what was swept, in what order, and why that order.

## 6. Results and Discussion  *(~2½ pages)*

- **4.II** 9 forecast plots → `MyDrive/milan_traffic_project/figures/`
- **4.III** per-area metric tables → `results/final_metrics_sq*.csv` on Drive
- **4.IV** timings + hardware → `results/final_timings_summary.csv` on Drive
- **4.VII** cross-area comparison → `results/final_rmse_by_area.csv` on Drive

Discussion must go beyond ranking. Address:

1. Does the ranking hold across all three areas, or does it change? Relate any
   change to the traffic characteristics found in §4.
2. Accuracy vs cost. If the TCN trains slower than the LSTM on this CPU — contrary
   to [7] — explain why: parameter count, depth, and the fact that CPU-bound small
   batches do not expose the parallelism the TCN's advantage depends on.
3. Skill vs persistence. A model beating persistence by a few percent has learned
   little; say so if that is what the numbers show.
4. **Failure analysis** (required) → `results/failure_analysis_sq*.csv` and `figures/failure_sq*.png` on Drive.
   Identify the hardest window, and explain it — a holiday, an anomaly, a level
   shift no model conditioned on recent history could anticipate.

## 7. Conclusion and Future Work  *(~½ page)*

- Answer the research question directly.
- Limitations: single-area univariate, one city, two months, CPU-only budget,
  hyperparameters tuned on one area only.
- Future work: spatio-temporal models [4]; multi-step horizons; cross-area transfer.

## 8. References

IEEE style, from `docs/RELATED_WORK.md`. **Must also include** the GitHub repo
link and the demo video link.

## 9. Use of AI  *(required disclosure)*

Be specific and honest about what assistance was used and where. Vague disclosure
is worse than none. You must be able to explain every design decision, the
memory-management strategy, the model choices, and the results.

---

## Figure budget

Roughly 10–12 figures. Candidates, in priority order:

1. Traffic distribution (4 panels) — Task 2.1, required
2. Spatial map — supports the "areas differ structurally" argument
3. Five-area time series — Task 2.2, required
4. MSTL decomposition — Task 2.3
5. ACF/PACF — Task 2.3, drives the lookback choice
6–8. Three forecast plots for the busiest area (one per model) — 4.II, required
9. Failure-analysis zoom — required
10. Optional: weekly profile heatmap, if §4 needs it

The other six forecast plots belong in an appendix, not the body.
