---
title: "Prediction-based Time Series Anomaly Detection with Prophet Library"
source: "https://medium.com/@ersinesen/prediction-based-time-series-anomaly-detection-with-prophet-library-da07593008a0"
author:
  - "[[Ersinesen]]"
published: 2025-01-14
created: 2026-04-16
description: "Prediction-based Time Series Anomaly Detection with Prophet Library Time series anomaly detection plays a critical role in a broad spectrum that encompasses cyber security. There are various methods …"
tags:
  - "clippings"
---
Time series anomaly detection plays a critical role in a broad spectrum that encompasses cyber security. There are various methods used for this purpose. A process-centric taxonomy by Boniol et. al \[1\] is shown in Figure 1.

![Time series method taxonomy](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*64IETZqUA8FZf3zD_lX6hA.png)

Figure 1. Taxonomy by Boniol et al. \[1\]

[Here](https://colab.research.google.com/drive/14ZWlawVWtYvlZAxJ6VxxkypEC7V6YDQi?usp=sharing) we implement a prediction-based times series anomaly detection method using [Prophet library of Meta](https://facebook.github.io/prophet/). Prophet is a powerful tool developed by Facebook, designed to handle time series data with trends and seasonality.

## Method

1\. Preprocessing the Data

The datasets are structured into two columns: *ds* (date/time) and *y* (values). Data cleaning and formatting ensure compatibility with the Prophet library.

2\. Training the Model

For each time step *t*, the data up to *t* is used to train a Prophet model.

This step involves detecting seasonal patterns and trends in the historical data to predict the next time point.

3\. Forecasting and Error Calculation

The model forecasts the next value for time *t+1*:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/0*TpAy6nGZvKpM273X)

The prediction error is calculated as the absolute difference between the predicted and actual values:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/0*OiAWtVyMltrJLBe4)

4\. Defining Anomalies

## Get Ersinesen’s stories in your inbox

Join Medium for free to get updates from this writer.

A dynamic threshold is set based on two standard deviations of the prediction errors:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/0*UHsQOP9ywNvAXoNn)

where *σ(e)* is the standard deviation of the previous prediction errors.

If the prediction error exceeds the threshold *θ*, the time step *t+1* is flagged as an anomaly:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/0*JJEQR_qOcaOh9m52)

## Results

We test with different datasets. Anomaly points are displayed on the original data as well as the predicted values in the following figures.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*upbaLuBwDEhBPD_6Rmx4tA.png)

Figure 2. Sunspot dataset.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*l8yzZX1PcmP02n3L0MXyHg.png)

Figure 3. Seattle weather dataset.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*1oE4kMLH1TxtI6svH6X_Qw.png)

Figure 4. Nile volume.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*4j8FOqeWTNQiuXRdBk4O_Q.png)

Figure 5. Daily Bitcoin values in 2024.

## Conclusion

The Prophet library is a versatile tool for time-series anomaly detection, particularly suited for data with underlying trends and seasonal patterns. It automatically handles seasonality, making it applicable across a wide range of datasets. This basic setup serves as a solid baseline for simple anomaly detection cases.

However, the detection rule can be made more generic. Furthermore, the initial warm-up period must be considered, as Prophet requires a sufficient historical dataset to generate reliable predictions. Finally, the variation characteristics and sampling frequency of the input data should be carefully taken into account to ensure the model adapts effectively to different data behaviors.

## References

\[1\] Boniol, Paul, et al. “Dive into Time-Series Anomaly Detection: A Decade Review.” arXiv preprint arXiv:2412.20512 (2024).

## Code

\[1\] [Colab notebook](https://colab.research.google.com/drive/14ZWlawVWtYvlZAxJ6VxxkypEC7V6YDQi?usp=sharing)

```c
import matplotlib.pyplot as plt
import numpy as np
from prophet import Prophet
import pandas as pd
import logging

# Suppress all logs from the \`cmdstanpy\` and \`prophet\` loggers
logging.getLogger('cmdstanpy').setLevel(logging.ERROR)
logging.getLogger('prophet').setLevel(logging.ERROR)

# 'data' is the entire dataset with 'ds' (datetime) and 'y' (value) columns

# List to store anomalies and prediction errors
anomalies = []
predicted_values = []
prediction_errors = []

# Step 1: Loop over all time steps t in the dataset
for t in range(2, len(data)):  # Start at 2, so we have data up to t-1
    # Use data from time 0 to time t-1 for training
    train_data = data[:t]

    # Fit the model on the training data up to time t-1
    model = Prophet()
    model.fit(train_data)

    # Step 2: Forecast the next time point (t)
    future = model.make_future_dataframe(periods=1)
    forecast = model.predict(future)

    # Get the predicted value for time t
    predicted_value = forecast['yhat'].iloc[-1]
    actual_value = data['y'].iloc[t]  # Actual value at time t (next time point)

    predicted_values.append(predicted_value)  # Store the predicted value

    # Step 3: Calculate the prediction error
    prediction_error = abs(predicted_value - actual_value)
    prediction_errors.append(prediction_error)

    # Step 4: Define a threshold (e.g., 2 standard deviations of prediction errors)
    threshold = 2 * np.std(prediction_errors)  # Adjust the threshold as necessary

    # Step 5: Detect anomalies (if the prediction error exceeds the threshold)
    if prediction_error > threshold:
        anomalies.append(data['ds'].iloc[t])  # Store the date of the anomaly

# Step 6: Plot the results

# Plot the original time series
plt.figure(figsize=(10, 6))
plt.plot(data['ds'], data['y'], label="Original Data", color='blue')

# Plot the predicted values (yhat) with a dashed line
plt.plot(data['ds'][2:], predicted_values, label="Predicted Data", color='orange', linestyle='--')

# Highlight anomalies
plt.scatter(anomalies, data.loc[data['ds'].isin(anomalies), 'y'], color='red', label='Anomalies')

# Add labels and title
#plt.title("Time Series with Anomalies Detected Based on Prediction Error")
plt.title("Bitcoin")
plt.xlabel("Time")
plt.ylabel("Value")
plt.legend()

# Display the plot
plt.show()
```

[![Ersinesen](https://miro.medium.com/v2/resize:fill:96:96/0*xUKKfhnC98Tahxzq)](https://medium.com/@ersinesen?source=post_page---post_author_info--da07593008a0---------------------------------------)[4 following](https://medium.com/@ersinesen/following?source=post_page---post_author_info--da07593008a0---------------------------------------)