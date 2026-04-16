---
title: Prophet for Time-Series Anomaly Detection
summary: How to use Meta's Prophet library for time-series anomaly detection in Python, covering two approaches (uncertainty-factor and dynamic-threshold), key parameters, and model evaluation.
updated: 2026-04-16
sources: Reza Rajabi, 2023-03-27; Ersinesen, 2025-01-14
raw: [outlier-anomaly-detection-facebook-prophet-python](../../raw/anomaly-detection/2023-03-27-outlier-anomaly-detection-facebook-prophet-python.md); [prediction-based-time-series-anomaly-detection-prophet](../../raw/anomaly-detection/2025-01-14-prediction-based-time-series-anomaly-detection-prophet.md)
---

# Prophet for Time-Series Anomaly Detection

[Prophet](https://facebook.github.io/prophet/) is Meta's open-source library for time-series forecasting (Python and R). It decomposes a time series into trend, seasonality, and holiday components. For anomaly detection, the core idea is simple: **forecast what the value should be, then flag points where the actual value deviates too far from the prediction**.

Prophet handles trends, changepoints, and multiple seasonalities automatically, making it a practical baseline for anomaly detection on seasonal operational metrics.

## Data format

Prophet expects a dataframe with two columns:

- `ds` — datetime
- `y` — the metric value

```python
import pandas as pd
df = pd.read_csv('data.csv')
df.rename(columns={'timestamp': 'ds', 'value': 'y'}, inplace=True)
```

## Key parameters

### changepoint_prior_scale

Controls trend flexibility. Default is `0.05`. Increasing it (e.g. `0.95`) makes the trend more responsive to sudden shifts, but risks overfitting. Decreasing makes it smoother.

```python
m = Prophet(changepoint_range=0.8, changepoint_prior_scale=0.05)
```

Prophet places 25 potential changepoints uniformly in the first 80% of the series (controlled by `changepoint_range`) and uses L1 regularization (`changepoint_prior_scale`) to use as few of them as possible.

### Seasonality

Prophet fits weekly and yearly seasonality automatically for series > 2 cycles, and daily seasonality for sub-daily data. Custom seasonalities can be added:

```python
m.add_seasonality(name='hourly', period=0.04, fourier_order=20)
```

- `period` — fraction of a day (0.04 ≈ 1 hour, 0.5 = 12 hours)
- `fourier_order` — how many terms to use; higher = captures finer patterns but risks overfitting

## Approach 1: Uncertainty-factor method

Fit the model on the full training set, forecast, then flag points where the prediction error exceeds a multiple of the forecast's own uncertainty band.

```python
m.fit(df)
future = m.make_future_dataframe(periods=48, freq='H')
forecast = m.predict(future)

merged = pd.merge(
    forecast[['ds', 'yhat', 'yhat_upper', 'yhat_lower']],
    df, on='ds', how='inner'
)

merged['error'] = merged['y'] - merged['yhat']
merged['uncertainty'] = merged['yhat_upper'] - merged['yhat_lower']

factor = 1.5
merged['anomaly'] = merged.apply(
    lambda x: 'Yes' if abs(x['error']) > factor * x['uncertainty'] else 'No',
    axis=1
)
```

**Tuning knob**: the `factor` (1.5 in this example). Higher = fewer anomalies flagged, lower = more sensitive.

The uncertainty band (`yhat_upper - yhat_lower`) varies by time step, so the threshold adapts to regions where the model is already uncertain vs confident.

## Approach 2: Dynamic-threshold method

Train incrementally — at each time step *t*, fit Prophet on data `[0, t)`, predict `t`, compute the error, and flag anomalies where error > 2σ of all prior errors.

```python
from prophet import Prophet
import numpy as np

anomalies = []
predicted_values = []
prediction_errors = []

for t in range(2, len(data)):
    train_data = data[:t]
    model = Prophet()
    model.fit(train_data)

    future = model.make_future_dataframe(periods=1)
    forecast = model.predict(future)

    predicted_value = forecast['yhat'].iloc[-1]
    actual_value = data['y'].iloc[t]
    predicted_values.append(predicted_value)

    prediction_error = abs(predicted_value - actual_value)
    prediction_errors.append(prediction_error)

    threshold = 2 * np.std(prediction_errors)
    if prediction_error > threshold:
        anomalies.append(data['ds'].iloc[t])
```

**Tradeoffs vs Approach 1:**

| | Uncertainty-factor | Dynamic-threshold |
| --- | --- | --- |
| Speed | Fast — single fit | Slow — refits at every step |
| Threshold | Adapts per-step via Prophet's own uncertainty | Global 2σ over all prior errors |
| Warm-up | None | Needs enough history to have a meaningful σ |
| Use case | Batch analysis of historical data | Streaming / real-time detection |

The dynamic-threshold approach is more realistic for production use (the model never sees future data) but is computationally expensive — each iteration refits the full model. For large datasets, consider fitting at intervals rather than every step.

## Model evaluation

Standard regression metrics on the forecast vs actuals:

```python
from sklearn.metrics import (
    mean_absolute_error,
    median_absolute_error,
    mean_absolute_percentage_error,
    mean_squared_error
)
from math import sqrt

MAE = mean_absolute_error(merged['yhat'], merged['y'])
MedAE = median_absolute_error(merged['yhat'], merged['y'])
MSE = mean_squared_error(merged['yhat'], merged['y'])
RMSE = sqrt(MSE)
MAPE = mean_absolute_percentage_error(merged['yhat'], merged['y'])
```

A low MAPE (e.g. 0.04%) indicates the forecast closely tracks actuals, meaning deviations flagged as anomalies are likely genuine rather than noise from a poor model.

## Limitations

- **Warm-up period** — Prophet needs enough historical data to learn seasonality. Results are unreliable for the first few cycles.
- **Sampling frequency matters** — the model must match the data's cadence. Sub-hourly data with hourly seasonality needs explicit configuration.
- **Not a general-purpose detector** — Prophet is optimized for univariate time series with trend + seasonality. For multivariate or non-temporal anomaly detection, consider isolation forests, autoencoders, or platform-specific ML (see [anomaly-detection-overview](anomaly-detection-overview.md)).
- **Computational cost** — the dynamic-threshold approach refitting at every step is O(n²) in practice.

## See also

- [anomaly-detection-overview](anomaly-detection-overview.md) — types of anomalies, best practices, commercial platform capabilities
