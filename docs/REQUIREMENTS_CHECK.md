# Requirements cross-check

Audit of the submission against the brief and the 100-point rubric. Verified against
`reports/report.pdf` (22 pages) and the three executed notebooks.

---

## A. Task requirements

### Task 1 — Data handling and memory management

| Required | Status | Where |
|---|---|---|
| Strategy for loading/processing within available resources | ✅ | Report §3.2; notebook 1 §5 |
| Description of approach and decisions | ✅ | §3.2 |
| **Evidence of memory before and after** | ✅ | 295.6 MB → 19.2 MB/day; 17.9 GB → flat. Notebook 1 §4 and §5 measure both |
| Discussion of improvement achieved | ✅ | 15.4× per day, 48.9× on disk, peak 339.6 MB |
| **Limitations and trade-offs** | ✅ | §3.2 final paragraph — country detail irreversible; 6.2 GB if loaded whole |
| Supported by sources where appropriate | ✅ | Refs [8] |

### Task 2 — Exploratory analysis

| Required | Status | Where |
|---|---|---|
| Figure: distribution of total traffic across areas | ✅ | Figure 1 (4 panels) |
| Discussion of the distribution's characteristics | ✅ | §4.1 — log-normal, Gini 0.608 |
| **Three areas with highest total traffic identified** | ✅ | 5161, 5059, 5259 |
| Figures: first two weeks for the 5 areas (top-3 + 4159 + 4556) | ✅ | Figure 3, five panels |
| Discussion of similarities/differences, notable patterns | ✅ | §4.2 — four behavioural signatures with land-use interpretation |
| **Two additional analyses on the highest-traffic area** | ✅ | MSTL decomposition (§4.3 Analysis 1), autocorrelation (Analysis 2) |
| Each with output + what it reveals + how it informs forecasting | ✅ | Both close with an explicit "consequence" |

*Three supporting analyses also included: stationarity (ADF/KPSS), weekly profile, anomaly screening.*

### Task 3 — Related work and model selection

| Required | Status | Where |
|---|---|---|
| Focused review of relevant research | ✅ | §2, 13 sources |
| Three models selected, sufficiently different | ✅ | Linear statistical / recurrent / convolutional |
| Justification per model | ✅ | §2 + §5.3 |
| Considers EDA characteristics | ✅ | e.g. harmonics chosen from the std-dev comparison |
| Considers previous research | ✅ | Refs [4], [7], [8] each tied to a decision |
| **Understanding/criticism of strengths and limitations** | ✅ | §6.6 per model; §2 "deliberately excluded" |

### Task 4 — Forecasting experiments

| # | Required | Status | Where |
|---|---|---|---|
| I | Self-contained description: structure, input representation, preprocessing, training | ✅ | **§5.2 + §5.3** — all three specified in full, incl. shared training table |
| II | **9 plots** (3 models × 3 areas), observed vs predicted | ✅ | Figures 8–10 (body) + A1–A6 (appendix) = 9 |
| III | **Three tables** with MAE, MAPE, RMSE per area | ✅ | 5 tables — §6.1 (two) + Appendix B (three) |
| IV | Training/execution time + method + **hardware** | ✅ | §6.3 — NVIDIA T4, TF 2.20, wall-clock, averaged over 5 areas |
| V | **Personal considerations: design, performance, margins for improvement** | ✅ | **§6.6**, per model plus a cross-cutting paragraph |
| VI | Input representation: sequence length, preprocessing, normalisation | ✅ | §5.2 table |
| VII | Comparative analysis: accuracy, training time, suitability; best model supported by quantitative + qualitative | ✅ | §6.2, §6.3, §6.4 |
| — | MAE and RMSE plus other suitable metrics | ✅ | MAE, RMSE, MAPE, sMAPE, R², skill |
| — | Table comparing the three models on the highest-traffic area | ✅ | §6.1 first table |
| — | **At least one period where models perform poorly, with reasons** | ✅ | §6.5 — Sunday 22 Dec, 56–85% degradation, Christmas regime shift |
| — | Performance across areas related to Task 2 characteristics | ✅ | §6.2 — neural models fail on the flat/noisy residential area |

### Report structure

| Required section | Status |
|---|---|
| Introduction | ✅ §1 |
| Related Work | ✅ §2 |
| Dataset and Data Preparation | ✅ §3 |
| Exploratory Analysis | ✅ §4 |
| Methodology | ✅ §5 |
| Results and Discussion | ✅ §6 |
| Conclusion and Future Work | ✅ §7 |
| References (IEEE) | ✅ §9 |
| **AI use disclosed** | ✅ §8 |
| **GitHub repo link in references** | ✅ [14] |
| **Demo video link in references** | ⚠️ **[15] — placeholder, fill before submitting** |

---

## B. Rubric coverage

| Criterion | Pts | Evidence |
|---|---|---|
| Data handling & memory | 10 | Measured before/after, chunk-size sensitivity, trade-off stated, 48.9× reduction |
| EDA & characterisation | 13 | 7 figures, log-normal + Gini, five distinct behavioural profiles tied to land use |
| Time-series analysis | 12 | MSTL variance partition, ACF/PACF at the decision-relevant lags, ADF+KPSS with the caveat that KPSS p is a bound |
| Model design & methodology | 14 | Three families, identical protocol, full structural specs, causality test |
| Experimentation & tuning | 10 | 32 configurations, staged with hypotheses, **plus the seed-variance check** |
| Evaluation & comparison | 13 | 5 areas × 5 models, 6 metrics, cost analysis, cross-area ranking |
| Failure analysis & reflection | 8 | §6.5 regime shift + §5.6 invalidated tuning + §6.6 per-model margins |
| Code quality | 5 | Three notebooks, documented, ETL verified to 1e-9 against an independent reference |
| Report quality & originality | 5 | 22 pp, structured, argues from evidence; reports its own falsified predictions |
| Reproducibility & repo | 2 | README with runtimes, notebooks run in order, manifest with MD5s |
| Video | 8 | ⚠️ **not yet recorded** — script in `docs/VIDEO_SCRIPT.md` |

---

## C. What still needs doing

1. **Record the video** (7–10 min). Script: `docs/VIDEO_SCRIPT.md`.
2. **Insert the video link** in two places in `reports/report.md` — the byline (line 5) and reference [15] — then rebuild the PDF.
3. **Read §8 (Use of AI)** and confirm it matches how you would describe the process. That disclosure is yours to stand behind.
4. **Read `docs/UNDERSTANDING.md`** before presenting.

## D. Distinguishing strengths

Three things here go beyond a typical submission and are worth pointing at in the video:

1. **The naive baseline is reported throughout.** Persistence reaches R² ≈ 0.99, so the models improve on it by only 2–17%. Many published papers omit this, which makes their results look far stronger than they are.
2. **Seed variance was measured.** Three of four tuning conclusions turned out to sit inside run-to-run noise, and the report says so rather than claiming four findings.
3. **Two predictions were falsified and recorded as such** — that the TCN would train more slowly, and that chunk size showed a clean memory/speed trade-off. Reporting what didn't hold is the part that reads as real research.
