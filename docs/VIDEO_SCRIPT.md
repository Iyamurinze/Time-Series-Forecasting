# Video presentation — script and timings

**Target: 7–10 minutes.** The brief requires: the problem, your approach to the dataset, the models and why, the main findings, **one technical decision discussed in depth**, and **one limitation or failure case**.

These are talking points, not a script to read aloud. Every number is in the report and the notebooks, so you can point at the source for any of them.

---

## 0:00–0:45 · The problem

- Operators commit radio and backhaul capacity **before** the traffic arrives. Under-provision and service degrades; over-provision and you waste capital.
- Horizon here is **one step = ten minutes ahead** — what short-term scheduling actually runs on.
- Data: Telecom Italia Milan grid. 10,000 cells, ten-minute sampling, two months, **19.38 GiB**.
- State the research question: how do sequential models compare, **and does the answer change across areas with different traffic characteristics?**

> Flag early that the second half of the question turns out to be the interesting one.

## 0:45–2:00 · The dataset and the memory problem

Show notebook 1.

- The feed is keyed by `(square_id, time_ms, country_code)` — **246 country codes**, roughly half the Internet cells empty at that granularity.
- Country code is **nuisance structure** for this question. Summing it away turns 4,842,625 rows/day into the dense 1,440,000-cell grid, **99.996% occupied**.
- Show the measured table: naive `read_csv` is **295.6 MB/day → 17.9 GB projected** for 62 days, against ~13 GB in a free Colab instance. **It does not fit** — categorical, not a speed problem.
- Optimised: **19.2 MB/day, 15.4× smaller**, and because it streams, peak RSS is **flat in day count — 339.6 MB across the whole run**.
- Final: **406 MB Parquet from 19.38 GiB — 48.9× compression**, in 6.1 minutes.
- Name the trade-off aloud: country detail is **gone**, and recovering it means re-downloading 19.4 GiB. Acceptable because the question needs only total traffic per area.

## 2:00–3:15 · What the data told us, and what it decided

Show notebook 2 figures.

- **Log-normal distribution.** Raw skewness 4.27; on a log scale it collapses to **0.023**. Gini **0.608** — the busiest 1% of areas hold 11% of traffic.
  → *Consequence:* absolute errors aren't comparable across areas, so report a **scale-free skill score**.
- **The five areas behave differently** — this is what makes the research question answerable:
  - 5161: peak/trough **19.7**, weekend/weekday **1.38** → leisure/tourist
  - 5259: weekend/weekday **0.43** → office, empties at weekends
  - 4556: peak at **22:00**, cv **0.48** → residential, flattest and noisiest
- **MSTL:** daily **82.9%** of variance, weekly **10.0%**, remainder **3.6%**.
  → *Consequence:* **92.9% is calendar-driven.** Say plainly: this is why the models finish close together. It's the ceiling the data imposes.
- **ACF at lag 1 = 0.987.** Persistence will be hard to beat — which is why naive baselines are reported throughout.

## 3:15–4:15 · Models and why

- **SARIMA** — linear, interpretable, the literature's reference point.
- **LSTM** — the default in this literature; gating spans a long window.
- **TCN** — dilated causal convolution. Same context **without recurrence**, so it parallelises across time. Bai et al. (2018) predict faster training — a testable claim.
- **Plus persistence and seasonal naive.** Not among the three, but without them an improvement can't be interpreted.
- Protocol: **rolling one-step-ahead with observed history** — errors never compound. Chronological splits. Hyperparameters chosen on a tuning split **shifted one week earlier**, so the reported week informs no decision.
- Mention the leakage check: perturb the series after time *t*, assert forecasts before *t* don't change. A model that peeks scores beautifully and forecasts nothing.

## 4:15–5:30 · ⭐ The technical decision (discuss in depth)

**This is the required deep-dive. Take your time.**

The seasonal period is **s = 144** — a day of ten-minute intervals.

1. **The naive approach fails.** Putting that seasonal difference in `statsmodels`' state space costs **146 state dimensions**. The Kalman covariance update is **cubic in the state** — a single fit **did not finish in ten minutes**.
2. **The obvious fix is a trap.** `simple_differencing=True` avoids the blow-up, but returns predictions on the **differenced scale** and mis-aligns the boundary when observations are appended. It produced *silently wrong* results — plausible numbers that were wrong.
3. **The resolution:** apply the seasonal difference **explicitly** — `z(t) = x(t) − x(t−s)` — fit a non-seasonal ARIMA to *z*, invert by adding `x(t−s)` back. Same model, **two state dimensions instead of 146**, every step visible and testable.
4. **Then the data overruled the design.** Experiment 1.1 showed seasonal differencing made things **worse** — RMSE 227.51 versus 175.44 without it. The EDA had predicted this: both stationarity tests accept the raw series, and first differencing shrinks the spread far more than seasonal differencing. The final model uses **Fourier harmonics** instead (152.40 vs 163.39) — what Hyndman & Athanasopoulos recommend for long seasonal periods.

> The point to land: the engineering problem and the statistical answer pointed in different directions, and the evidence settled it.

**Alternative if you prefer:** the seed-variance experiment (see 6:45) is an equally strong deep-dive.

## 5:30–6:45 · Results

Show the RMSE-by-area table.

| Model | 4159 | 4556 | 5059 | 5161 | 5259 | Mean rank |
|---|---|---|---|---|---|---|
| LSTM | 19.24 | 38.48 | **97.62** | 119.54 | 91.33 | 1.8 |
| SARIMA | **19.01** | **34.17** | 97.64 | 127.95 | 94.40 | 2.0 |
| TCN | 20.85 | 38.75 | 100.63 | **119.48** | **90.94** | 2.2 |
| Persistence | 21.54 | 39.62 | 114.38 | 134.88 | 109.58 | 4.0 |

- **No model dominates.** Ranks 1.8 / 2.0 / 2.2. Each wins somewhere.
- **Be honest about the ties:** TCN "beats" LSTM on 5161 by **0.06 RMSE**, against a measured TCN seed noise band of **22.0**. That's a tie. Calling it a win is reading noise as signal.
- **Where it separates, it separates by area character:**
  - **4556** (residential, flattest, noisiest): SARIMA wins by 11%, and both neural models **barely clear persistence** (+0.029, +0.022). They fail where the series is least structured.
  - **5161** (most cyclical, peak/trough 19.7): neural models **double** SARIMA's skill. The nonlinear shape is what they're for.
- **Cost:** SARIMA trains in **12.1 s with 14 parameters**; the LSTM takes 50.1 s with **16,961** — 1,212× more. SARIMA is the cost-adjusted winner.
- **On Bai et al.:** the TCN does train **1.3× faster** — confirmed. **But at inference it's 6.2× slower**, because twelve conv layers run per prediction. For a system predicting every ten minutes across 10,000 cells, inference is what matters.

## 6:45–7:45 · ⭐ The limitation / failure case

**Two strong options — pick one, or do both briefly.**

**Option A — the regime change (best failure case).**
- Hardest window: **Sunday 22 December, 11:50–23:40**. Every model degrades **56–85%**.
- Why: the MSTL trend falls **45% between 9 and 27 December** as the city winds down for Christmas. **Training data ends 15 December.** Every model is forecasting a regime it has never seen.
- **SARIMA degrades most (1.85×)** — its harmonics encode a *fixed* shape that can't adapt to a shifting level. **TCN degrades least (1.56×)** — its short 36-step window makes it most locally adaptive, the same property that made it overfit at longer windows. A coherent trade-off, not a contradiction.
- Fix: exogenous calendar features, or periodic refitting. Both excluded by the univariate spec.

**Option B — the tuning that wasn't real.**
- Retrained each fixed architecture under **5 random seeds**. Seed range: LSTM **8.69**, TCN **22.00** RMSE.
- Against each sweep's spread: TCN lookback **36.12** → real. LSTM lookback **11.09** → marginal. **LSTM capacity 7.72 and TCN width 4.77 → inside the noise band.**
- So "64 units was selected" and "16 filters was selected" are **arbitrary picks inside noise**. Three of four tuning conclusions don't survive.
- Say it plainly: without this check the report would have claimed four findings where the evidence supports one.

## 7:45–8:30 · Conclusions and close

- **Answer the question directly:** no model dominates; the ranking is not stable across areas; where it separates, it separates by **how structured the area's traffic is**.
- **Deployable answer:** SARIMA — competitive everywhere, 1,212× fewer parameters, fastest inference.
- **What the baseline revealed:** persistence achieves R² ≈ 0.99. Improvements of 2–17% are the ceiling the data imposes, not a disappointment.
- **Limitations:** univariate and single-area by spec; one city; two months; tuned on one area and transferred; evaluation week sits on a regime change.
- **Future work:** calendar features; spatio-temporal models exploiting the grid; ensembling across seeds, since seed noise is comparable to the between-model differences.

---

## Delivery checklist

- [ ] Screen-record with the notebooks open — show **real output**, not slides of numbers
- [ ] Have the report open for the tables
- [ ] Say the numbers out loud; don't just point at them
- [ ] **Never claim a model "won"** where the gap is inside the noise band — the easiest way to lose credibility in a viva
- [ ] Mention AI assistance briefly and honestly (disclosed in §8 of the report)
- [ ] Add the video link to the report's reference [15] and to the README before submitting

## Likely viva questions

| Question | Where the answer is |
|---|---|
| Why drop the country code? | §3.2 — nuisance dimension, 99.996% dense after collapse, trade-off stated |
| Why not just use more RAM, or Dask? | Naive path is 17.9 GB vs ~13 GB available; streaming makes peak flat in day count |
| Why is seasonal naive so bad? | ACF: lag-1 is 0.987, lag-144 is 0.878. Ten minutes ago beats yesterday |
| Why did SARIMA beat the deep models anywhere? | 92.9% of variance is calendar-driven; on flat, noisy areas there's no nonlinearity to exploit |
| How do you know there's no leakage? | Explicit test: perturb after *t*, assert forecasts before *t* unchanged. All five models pass |
| Why only a 2–17% improvement? | That's the ceiling: remainder is 3.6% of variance; persistence is already R² ≈ 0.99 |
| Would more tuning have helped? | Seed variance says no — two sweeps already sit inside the noise band |
