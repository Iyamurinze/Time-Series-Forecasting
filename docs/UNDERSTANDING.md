# Understanding this project — a plain-language guide

Written for someone starting from zero. Read it once end to end; it builds up, so don't skip.
By the end you should be able to explain any part of the report in your own words — which is the
only thing that survives a follow-up question.

---

## 1. What problem are we actually solving?

Imagine you run a mobile network in Milan. Phones use data, and you have to provide enough capacity
to carry it. Here's the catch: **you have to decide how much capacity to provide *before* people
use it.** If you provide too little, people's internet is slow. If you provide too much, you've
spent money on equipment sitting idle.

So you want to predict: *how much data traffic will there be in the next few minutes?*

That's it. That's the whole problem. **We are predicting the near future from the recent past.**

### How near is "near"?

The city is divided into a grid — 100 × 100 = **10,000 small squares**, each about 235 metres
across. Every **10 minutes**, we know how much internet activity happened in each square.

We predict **one step ahead** = **10 minutes ahead**.

So: *"Given everything up to 14:20, how much traffic will there be at 14:30?"*

That's called **one-step-ahead forecasting**. One step = one 10-minute interval.

> **Say it like this:** "We're forecasting mobile internet traffic ten minutes ahead, for individual
> city districts, so an operator can allocate capacity before demand arrives."

---

## 2. What the data looks like

Two months of data: **1 November 2013 to 1 January 2014**. That's 62 days.

- 62 days × 144 ten-minute intervals per day = **8,928 time points**
- 10,000 squares
- The raw files are **19.38 GB** — big enough that you can't just open it all at once

Each row in the raw file says something like: *"In square 5161, at this timestamp, from phones
registered in Italy, there was this much internet activity."*

### The complication that mattered

There's a column called **country code** — it splits the traffic by which country the phone is
registered in (Italy, Germany, etc.). There are **246 different country codes**.

We don't care about that. We just want *total* traffic per square. So the very first thing the
pipeline does is **add up across all country codes**.

Why this matters so much: that one step shrinks **4,842,625 rows per day** down to **1,440,000**
(which is just 10,000 squares × 144 intervals). Doing it *immediately*, before storing anything, is
what made the whole thing fit in memory.

> **Say it like this:** "The raw data is split by country code, which we don't need. Summing it away
> at load time is what makes the dataset manageable — it's the single biggest saving."

---

## 3. The memory problem (and why it's a real problem)

If you load the data the obvious way — `pandas.read_csv`, no options — one day takes **295.6 MB**
of memory.

You need all 62 days at once to do any analysis across days. So:

**295.6 MB × 62 = 17.9 GB**

Google Colab gives you about **13 GB**. So it doesn't fit. Not "it's slow" — **it does not fit.**

### What we did instead

Four changes, each simple:

| What we did | Why |
|---|---|
| Only read 3 of the 8 columns | We don't need SMS or call data |
| Use smaller number types | `square_id` only goes up to 10,000 — doesn't need a 64-bit number |
| Add up country codes immediately | 4.8M rows → 1.44M rows |
| Read the file in **chunks**, not all at once | Memory used stays small no matter how big the file is |

Result: **19.2 MB per day instead of 295.6 MB** — about **15× smaller**. And because we process one
chunk at a time and write the result to disk, the memory used **doesn't grow** as we add more days.
Across all 62 days, the peak was **339.6 MB**.

Final stored size: **406 MB instead of 19.38 GB** — about **49× smaller**.

### The honest trade-off

Once we've added up the country codes, **we can't get them back**. If someone later asked "how much
traffic came from German phones?", we'd have to re-download all 19.4 GB.

We accepted that because our question only needs total traffic. **Always state the trade-off** — it
shows you made a decision rather than just doing the easy thing.

> **Say it like this:** "The naive approach would need 17.9 GB against Colab's 13 GB — it simply
> doesn't fit. We stream the data in chunks and aggregate immediately, which keeps peak memory at
> 340 MB regardless of how many days we process. The cost is that country-level detail is gone
> permanently."

---

## 4. The single most important idea in the whole project

**This is the thing to understand. If you only understand one thing, make it this.**

Traffic 10 minutes from now is *almost the same* as traffic right now. Data traffic doesn't
teleport — it rises and falls smoothly.

So here's a "model" that requires no maths at all:

> **Whatever it is now, predict the same thing in 10 minutes.**

That's called **persistence** (or the "naive" forecast). And it's *really good*. We measured it: it
explains about **99% of the variation** in the data.

### Why this changes everything

If a dumb rule gets 99% right, then a fancy neural network can only compete for the remaining 1%.
**So the question is never "is my model good?" — it's "is my model better than doing nothing?"**

That's why every result in the report is shown next to persistence, and why we report a number
called **skill**:

- **Skill = 0** → exactly as good as persistence (i.e. the model added nothing)
- **Skill = +0.10** → 10% better than persistence
- **Skill = negative** → *worse* than persistence (embarrassing)

Our three models scored between **+0.022 and +0.170**. So: between 2% and 17% better than doing
nothing.

**That is not a disappointing result — it's an honest one.** Many published papers don't report the
naive baseline at all, which makes their models look far more impressive than they are.

> **Say it like this:** "At a ten-minute horizon, just repeating the last observation already
> explains 99% of the variance. So we report every model against that baseline. Our models improve
> on it by 2 to 17 percent, and being upfront about that is the point."

---

## 5. Why can't we do much better? (The decomposition)

We took the busiest square's traffic and split it into parts, like separating a song into
instruments. The parts are:

| Part | What it means | How much of the variation |
|---|---|---|
| **Daily cycle** | Low at 4am, high in the afternoon — every day | **82.9%** |
| **Weekly cycle** | Weekends differ from weekdays | **10.0%** |
| **Trend** | Slow drift over weeks | 2.8% |
| **Remainder** | Everything unexplained — the genuinely random bit | **3.6%** |

Add up the daily + weekly + trend: **~96% of the variation is just the calendar.** It's the same
pattern repeating.

**Only 3.6% is genuinely unpredictable randomness.**

This explains the whole results section in one line: *there simply isn't much left for a clever
model to find.* Any model that gets the daily rhythm right has captured nearly everything.

> **Say it like this:** "The decomposition shows 92.9% of the variance is calendar-driven and only
> 3.6% is genuinely stochastic. That's the ceiling — it's why all three models land close together."

---

## 6. The three models, in plain terms

We picked three models that work in **fundamentally different ways**. The brief required them to be
genuinely different, not three flavours of the same thing.

### SARIMA — the statistics approach

**The idea:** describe the series with an equation. "Tomorrow at 2pm ≈ some weighted combination of
recent values, plus an adjustment for the fact that 2pm is usually busy."

- It's **linear** — it can only add and multiply, no complicated curves
- It has **14 numbers** (parameters) it learns. That's tiny.
- You can read those numbers and understand what it's doing
- Been around since the 1970s (Box–Jenkins)

**Analogy:** a recipe. Fixed ingredients, fixed proportions, written down and readable.

### LSTM — the memory approach

**LSTM = Long Short-Term Memory.** It's a type of neural network.

**The idea:** read the series one step at a time, keeping a running "memory" of what matters. At
each step it decides what to remember and what to forget.

- It's **non-linear** — it can learn complicated curved relationships
- It has **16,961 parameters** — about **1,200× more than SARIMA**
- You can't read them; it's a black box
- It reads **sequentially** — step 1, then 2, then 3… which makes it slow

**Analogy:** reading a book and keeping notes as you go, deciding what's worth writing down.

### TCN — the pattern-matching approach

**TCN = Temporal Convolutional Network.** Also a neural network, but built differently.

**The idea:** instead of reading step by step, slide pattern-detectors across the whole window at
once. Early layers spot short patterns; later layers spot longer ones by combining them.

- Also **non-linear**
- **5,601 parameters**
- It looks at everything **in parallel**, so training is faster than LSTM
- But it has many layers, so *making a prediction* is slower

**Analogy:** looking at a whole page at once and spotting shapes, rather than reading word by word.

### Plus two "do nothing" baselines

- **Persistence** — predict the last value (explained above)
- **Seasonal naive** — predict what happened at this same time yesterday

Seasonal naive turned out to be **terrible** — much worse than persistence. Why? Because 10 minutes
ago is far more similar to now than yesterday was. (The numbers back this: correlation with 10
minutes ago is **0.987**; with yesterday, **0.878**.)

> **Say it like this:** "We chose three models with genuinely different mechanisms — a linear
> statistical model, a recurrent network that reads sequentially, and a convolutional network that
> processes the window in parallel. That way any performance difference reflects the architecture,
> not the tuning effort."

---

## 7. How we made the comparison fair

This part matters a lot for marks, and it's easy to explain.

### Split the data by time, never randomly

| Part | Dates | Used for |
|---|---|---|
| **Training** | 1 Nov – 8 Dec | Teaching the model |
| **Validation** | 9 – 15 Dec | Deciding when to stop training |
| **Test** | **16 – 22 Dec** | **The reported result — used once** |

**Why by time and not randomly?** If you shuffled randomly, a model could train on Wednesday
afternoon and be tested on Wednesday morning — it would have effectively seen the answer. Time
series must always be split chronologically.

### Choosing settings without cheating

To pick the model settings, we used an **even earlier week** (9–15 Dec) and shifted everything back.
That way the reported week (16–22 Dec) **never influenced any decision** — we looked at it exactly
once, at the very end.

### Checking the models don't cheat

This is the neat bit. A model that secretly peeks at future values would score brilliantly and be
completely useless. So we **tested for it**:

1. Make a prediction for every point in the test week.
2. Now go back and **corrupt the data in the second half** of that week (add a huge number).
3. Re-run the predictions.
4. **The first half must be identical.** If it changed, the model was looking ahead.

All five models passed.

> **Say it like this:** "We split chronologically, chose hyperparameters on an earlier week so the
> reported week informed no decision, and explicitly tested every model for look-ahead by perturbing
> the future and confirming past predictions didn't change."

---

## 8. What we found

### Finding 1 — No model wins

| Model | Average rank across 5 areas |
|---|---|
| LSTM | 1.8 |
| SARIMA | 2.0 |
| TCN | 2.2 |

Those are basically tied. Each model won on some areas and lost on others.

### Finding 2 — Some "wins" are too small to be real

On square 5161: TCN scored **119.48**, LSTM scored **119.54**. The TCN "won" by **0.06**.

But here's the thing — **we measured how much a model's score wobbles just from random chance.**
Neural networks start with random numbers, so training the *exact same model* twice gives slightly
different results. We trained each one **5 times** with different random starts:

- LSTM scores varied by **8.69**
- TCN scores varied by **22.00**

So a gap of 0.06 is **nothing**. It's like saying someone is taller because they measured 0.06 mm
more — the measurement error is bigger than the difference.

**This is one of the strongest things in the report.** Most student projects (and plenty of
published papers) would declare "the TCN is best!" We measured the noise and found the difference
was inside it.

### Finding 3 — The differences that ARE real depend on the area

Remember, the five squares behave differently:

| Square | What it's like | Who won |
|---|---|---|
| 5161 | City centre, big daily swings, busier at weekends → **tourist/leisure** | Neural models (by a lot) |
| 4556 | Flat, quiet, peaks at 10pm → **residential** | **SARIMA** (by 11%) |

**The pattern:** neural networks help when the daily pattern is **dramatic and curvy** (they're good
at complicated shapes). They stop helping when the series is **flat and noisy** — there's no
complicated shape to learn, so the simple model does just as well with far less effort.

**This is the answer to the research question.** Not "model X is best", but "*it depends on the
area, and here's specifically what it depends on*."

### Finding 4 — SARIMA is the practical choice

| | SARIMA | LSTM |
|---|---|---|
| Parameters | **14** | 16,961 |
| Training time | **12 seconds** | 50 seconds |
| Prediction speed | **fastest** | 3.5× slower |
| Accuracy rank | 2nd | 1st |

It's essentially tied on accuracy with **1,200× fewer parameters**. If you were actually deploying
this across 10,000 city squares, you'd pick SARIMA.

---

## 9. The two deep-dives you'll be asked about

### Deep-dive A — The technical problem we hit

The daily pattern repeats every **144** time steps (that's 24 hours ÷ 10 minutes).

Standard statistics software handles repeating patterns by storing information about all 144 steps.
That means tracking **146 numbers**, and the maths involved scales with the **cube** of that
(146³ ≈ 3.1 million operations per time step, for ~7,000 time steps).

**Result: one attempt to fit the model didn't finish in 10 minutes.** Unusable.

**What we did:** instead of asking the software to handle the repetition, we did it by hand.
Subtract each value from the value exactly one day earlier:

```
z(today 2pm) = traffic(today 2pm) − traffic(yesterday 2pm)
```

That removes the daily pattern up front. Now the software only tracks **2 numbers instead of 146**,
and it fits in seconds.

**The twist:** when we actually tested it, this turned out to be the *wrong* thing to do
statistically. Subtracting yesterday makes the result *noisier* (you're combining two noisy
measurements). The experiments showed it made accuracy **worse** — 227.51 versus 175.44.

So the final model uses a different technique (**Fourier terms** — describing the daily pattern with
sine waves instead of subtraction), which scored **152.40**.

> **Why this is a good story to tell:** you hit a real engineering wall, solved it, then discovered
> the data disagreed with your solution, and followed the evidence. That's exactly what "research"
> means.

### Deep-dive B — When the models failed

The worst period was **Sunday 22 December**. Every model got noticeably worse there.

**Why?** It's the last weekend before Christmas. The city is winding down — traffic drops about
**45%** between 9 and 27 December.

But our models were **trained on data ending 15 December**. They have never seen Christmas. They
have no way of knowing a holiday is coming — there's no "it's nearly Christmas" input.

Interestingly:
- **SARIMA got worst** — it has a fixed idea of what a day looks like and can't adapt
- **TCN coped best** — it only looks at the last 6 hours, so it adjusts quickly to a changing level

**The fix** would be to tell the model about holidays (a "calendar feature"), or retrain it
regularly. We didn't, because the assignment specified using only the traffic series itself.

---

## 10. Glossary — every term you might be asked

| Term | Plain meaning |
|---|---|
| **One-step-ahead** | Predicting the very next time point (10 minutes ahead) |
| **Baseline** | A deliberately simple method you must beat to have achieved anything |
| **Persistence** | The baseline: predict that the next value equals the current value |
| **Skill score** | How much better than the baseline. 0 = no better; 0.1 = 10% better |
| **RMSE** | Average error, but big mistakes count extra. Lower is better |
| **MAE** | Average error, all mistakes count equally. Lower is better |
| **MAPE** | Average error as a *percentage*. Useful when scales differ |
| **R²** | Fraction of the variation explained. 1.0 is perfect |
| **Autocorrelation (ACF)** | How similar the series is to itself N steps ago. 0.987 at lag 1 = almost identical |
| **Stationary** | The statistical behaviour doesn't drift over time |
| **Seasonality** | A pattern that repeats on a fixed cycle (daily, weekly) |
| **Decomposition** | Splitting a series into trend + repeating patterns + leftover noise |
| **Lookback / window** | How many past points the model is shown. LSTM used 144 (a day), TCN used 36 (6 hours) |
| **Hyperparameter** | A setting you choose, not something the model learns (e.g. how many layers) |
| **Overfitting** | The model memorises the training data and does worse on new data |
| **Epoch** | One full pass through the training data |
| **Early stopping** | Stop training when performance stops improving, to avoid overfitting |
| **Parameters** | The numbers the model learns. SARIMA: 14. LSTM: 16,961 |
| **Seed** | The random starting point. Different seeds → slightly different results |
| **Chunking** | Reading a big file a piece at a time so memory stays low |
| **Parquet** | A compressed file format for tables — much smaller than text |
| **Look-ahead / leakage** | Accidentally letting the model see the future. Makes results meaningless |

---

## 11. Honest advice before you present

**You will be asked follow-up questions.** Reading a script won't get you through that — but you
don't need to memorise anything either. Almost every question comes back to a handful of ideas:

1. **The data is 19.4 GB, so we had to be careful with memory** → we aggregated early and streamed
2. **Predicting 10 minutes ahead is easy** → so the baseline is strong and we report against it
3. **96% of the pattern is just the calendar** → so there's little room for clever models
4. **Three models, one fair protocol** → so differences reflect architecture, not effort
5. **We measured the noise** → so we know which differences are real
6. **The answer depends on the area** → that's the actual research finding

If you genuinely understand those six lines, you can answer almost anything by reasoning from them.

**If you get asked something you don't know:** say so, then reason out loud from what you do know.
"I'd have to check the exact number, but the reason we did X was Y" is a *good* answer. Bluffing is
the only bad one.

**One rule:** never claim a model "won" when the difference is inside the noise band. It's the
single easiest way to lose credibility — and being the person who *knows* that distinction is
exactly what makes this project strong.
