# Related work — sources and what each one decided

Working bibliography for the report's *Related Work* section. Each entry records
**the decision it informed**, so the review stays connected to the methodology
instead of being a detached literature summary.

> Verify every entry against the publisher's page before submission, and add
> anything found during your own reading. References are IEEE style.

---

## A. The dataset

**[1]** G. Barlacchi *et al.*, "A multi-source dataset of urban life in the city of
Milan and the Province of Trentino," *Scientific Data*, vol. 2, art. 150055, 2015,
doi: 10.1038/sdata.2015.55.

*Informed:* the grid semantics (10,000 squares, 235 m, 10-minute aggregation), the
meaning of the `country_code` column, and the fact that activity values are
normalised CDR counts rather than byte volumes — which is why all errors in this
report are in "activity units" and why MAPE is reported with its coverage.

---

## B. Cellular traffic forecasting — the problem context

**[2]** W. Jiang, "Cellular traffic prediction with machine learning: A survey,"
and the companion survey *A Survey on Deep Learning for Cellular Traffic
Prediction*, *Intelligent Computing*, 2023, doi: 10.34133/icomputing.0054.

*Informed:* confirms the Milan dataset is the field's standard benchmark, so
results here are comparable to published work; and that CNN/RNN/LSTM families
dominate the literature — which is what makes an LSTM the expected choice and a
TCN the informative contrast.

**[3]** *Deep Learning on Network Traffic Prediction: Recent Advances, Analysis,
and Future Directions*, *ACM Computing Surveys*, 2024, doi: 10.1145/3703447.

*Informed:* the observation that the field lacks fair, consistently-configured
comparisons — the gap this study addresses by holding the input representation,
optimiser and evaluation protocol identical across the neural models.

**[4]** C. Zhang and P. Patras, "Long-term mobile traffic forecasting using deep
spatio-temporal neural networks," in *Proc. ACM MobiHoc*, 2018, pp. 231–240,
doi: 10.1145/3209582.3209606. (arXiv:1712.08083)

*Informed:* the decision **not** to attempt a spatio-temporal model. Their gains
come from convolving over the spatial grid, which the brief's single-area,
univariate formulation excludes. Cited as a limitation and as future work.

**[5]** *Cellular Traffic Prediction and Classification: A Comparative Evaluation
of LSTM and ARIMA*, in *Discovery Science*, LNCS, 2019,
doi: 10.1007/978-3-030-33778-0_11.

*Informed:* the direct precedent for this comparison. Reports LSTM beating ARIMA
*when training data is long and granularity fine* — a conditional finding this
study can test, since 8,928 ten-minute intervals sits exactly in that regime.

---

## C. Model families

**[6]** S. Hochreiter and J. Schmidhuber, "Long short-term memory," *Neural
Computation*, vol. 9, no. 8, pp. 1735–1780, 1997, doi: 10.1162/neco.1997.9.8.1735.

*Informed:* the gating argument for spanning a 144-step window without vanishing
gradients.

**[7]** S. Bai, J. Z. Kolter, and V. Koltun, "An empirical evaluation of generic
convolutional and recurrent networks for sequence modeling," *arXiv:1803.01271*,
2018.

*Informed:* the TCN's selection and its design. Their finding — that a simple
dilated causal convolutional stack matches or beats LSTMs across sequence tasks
while training far faster and retaining longer effective memory — is the specific
claim this study tests on traffic data. The residual-block structure and the
kernel/dilation receptive-field formula in `src/models/tcn.py` follow this paper.

**[8]** A. van den Oord *et al.*, "WaveNet: A generative model for raw audio,"
*arXiv:1609.03499*, 2016.

*Informed:* the dilated causal convolution itself, and the causality constraint
that makes the architecture admissible for forecasting.

---

## D. Time-series methodology

**[9]** R. J. Hyndman and G. Athanasopoulos, *Forecasting: Principles and
Practice*, 3rd ed. Melbourne, Australia: OTexts, 2021. [Online].
Available: https://otexts.com/fpp3/

*Informed:* three concrete decisions — (i) for long seasonal periods, prefer
seasonal differencing or harmonic regression over seasonal AR/MA terms, which is
exactly the constraint that made SARIMA tractable at `s = 144`; (ii) the seasonal
strength measure `F_S` used in the decomposition table; (iii) the insistence on
reporting a naive benchmark so accuracy claims are relative rather than absolute.

**[10]** R. B. Cleveland, W. S. Cleveland, J. E. McRae, and I. Terpenning, "STL: A
seasonal-trend decomposition procedure based on loess," *Journal of Official
Statistics*, vol. 6, no. 1, pp. 3–73, 1990.

**[11]** K. Bandara, R. J. Hyndman, and C. Bergmeir, "MSTL: A seasonal-trend
decomposition algorithm for time series with multiple seasonal patterns,"
*arXiv:2107.13462*, 2021.

*Informed:* the use of MSTL rather than plain STL. Traffic carries a daily *and*
a weekly cycle; a single-period decomposition would push the weekly component into
the remainder and overstate how much of the signal is unpredictable.

**[12]** D. A. Dickey and W. A. Fuller, "Distribution of the estimators for
autoregressive time series with a unit root," *J. American Statistical
Association*, vol. 74, no. 366, pp. 427–431, 1979.

**[13]** D. Kwiatkowski, P. C. B. Phillips, P. Schmidt, and Y. Shin, "Testing the
null hypothesis of stationarity against the alternative of a unit root,"
*J. Econometrics*, vol. 54, no. 1–3, pp. 159–178, 1992.

*Informed:* pairing ADF with KPSS. The tests have opposite nulls, so agreement is
evidence and disagreement is diagnostic — a single test on a strongly seasonal
series of 8,928 points rejects too easily to be trusted alone.

**[14]** G. E. P. Box and G. M. Jenkins, *Time Series Analysis: Forecasting and
Control*. San Francisco, CA: Holden-Day, 1970.

*Informed:* the SARIMA specification and the identification procedure (ACF/PACF →
differencing → order selection) followed in experiment stage 1.

---

## E. Tools

**[15]** S. Seabold and J. Perktold, "statsmodels: Econometric and statistical
modeling with Python," in *Proc. 9th Python in Science Conf.*, 2010, pp. 92–96.

**[16]** M. Abadi *et al.*, "TensorFlow: Large-scale machine learning on
heterogeneous systems," 2015. [Online]. Available: https://www.tensorflow.org/

**[17]** C. R. Harris *et al.*, "Array programming with NumPy," *Nature*, vol. 585,
pp. 357–362, 2020, doi: 10.1038/s41586-020-2649-2.

---

## F. Deliberately not used — and why

Worth a sentence each in the report; the brief asks for *criticism* of model
choices, and defensible exclusions are part of that.

| Considered | Why excluded |
|---|---|
| Transformer / Informer | Attention needs far more data than ~6,300 univariate windows, and self-attention is O(L²) on a CPU-only machine. Likely to lose to simpler models while costing the most — a negative result that would say more about the budget than the architecture. |
| ConvLSTM / STN [4] | Requires the spatial grid; the brief's task is explicitly single-area and univariate. Named as future work. |
| Prophet | Built for daily-or-longer business series with holiday effects; at 10-minute resolution and one-step-ahead it has no mechanism the SARIMA does not already have. |
| Gradient boosting on lag features | Genuinely strong and fast, but tabularising the window discards the sequential inductive bias that is the object of study. |
