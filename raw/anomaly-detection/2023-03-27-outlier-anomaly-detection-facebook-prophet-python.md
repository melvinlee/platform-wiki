---
title: "Outlier and Anomaly Detection using Facebook Prophet in Python"
source: "https://medium.com/@reza.rajabi/outlier-and-anomaly-detection-using-facebook-prophet-in-python-3a83d58b1bdf"
author:
  - "[[Reza Rajabi]]"
published: 2023-03-27
created: 2026-04-16
description: "Outlier and Anomaly Detection using Facebook Prophet in Python Anomaly detection is a data science application combining multiple tasks like classification, regression, and clustering. Anomaly …"
tags:
  - "clippings"
---
**Anomaly detection** is a data science application combining multiple tasks like classification, regression, and clustering. Anomaly detectionis a key tool for improving your ability to make better business decisions. It is a process in machine learning that identifies data points, events, and observations that deviate from a data set’s normal behaviour. **Time series anomaly detection** is a way to gain insights into specific business strategies. Time series data tracks the point-in-time value of a metric over time (such as customer acquisition costs, revenue per click rates, or website bounce rates).

The root cause of the anomaly depends on the kind of anomaly you are dealing with:

- **Point anomalies**: Global outliers are individual data points with extreme values, often indicators of experimental error. False positives are a common type of point anomaly.
- **Conditional or contextual anomalies**: These are data points with characteristics that deviate significantly from the other relative data points in a specific context. For example, seasonal time series data may show an anomaly for the summer months that might not appear to be an anomaly in the context of a decade’s worth of historical data.
- **Collective anomalies** are a collection of data points that do not appear anomalous in their values, either globally or contextually. As they occur in a cluster or sequence, they indicate a bigger anomaly is occurring. Collective anomalies are often identified through intrusion detection.

## Why is anomaly detection important?

- **Automated KPI analysis**: AI algorithms constantly scan your data across all your dashboards and analyze metrics 24/7.
- **Prevention of security breaches and threats**: Security breaches can be detected as soon as they happen because the AI is constantly scanning your data and will pick up on anything unusual immediately.
- **Discovery of hidden performance opportunities**: If anomaly detection is applied, this repetitive work can be eliminated, freeing time to plan and execute more performance-driving strategies.
- **Faster results**: Manual finding anomalies in data can be extremely time-consuming as it takes a long time to surface using traditional reporting techniques. AI-based anomaly detection provides results faster by identifying the anomalies immediately.

## Anomaly detection using Facebook Prophet

**Facebook Prophet** is an open-source library developed by Facebook’s in-house data science team to address time series-based forecasting problems. It is a rapid and easy way to create forecasts. It can make predictions about future events based on historical data. Facebook Prophet is available in both Python and R languages.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*FlLCTmIn_T7rJnbf)

An output of Facebook Prophet

## Anomaly detection with Python and Facebook Prophet

Facebook Prophet provides a [Python library](https://facebook.github.io/prophet/docs/installation.html#python) for forecasting. Let’s assume we have a product sale dataset which includes the number of sold products (`Quantity`) based on hour, and we want to identify possible anomalies in data. For this experiment, we consider seven days of data, similar to the following structure:

## Get Reza Rajabi’s stories in your inbox

Join Medium for free to get updates from this writer.

First, we import `Facebook Prophet` library:

```c
# Requirement for Facebook Prophet library
%pip install pystan==2.19.1.1

# Importing facebook prophet library
%pip install fbprophet
```

We load the data into a dataframe (The code and data are available at [my GitHub](https://github.com/erajabi/anomaly_detection)):

```c
import pandas as pd
sale = pd.read_csv('./data.csv')
```

And the sale dataset is shown below:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*MlKF7CZ_MdVGAhGVywM0Yw.png)

Hourly-based sale product

We can visualize this time-series dataset in a line graph:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*7gtAnRTZuLjRra_eG6l-rg.png)

We should prepare the dataset to fit the model. To this end, we rename the column accordingly:

```c
sale.rename(columns={'hour_col':'ds', 'quantity':'y'}, inplace=True)
```

As the [Facebook Prophet website](https://facebook.github.io/prophet/) mentioned, we can set a few parameters to initialize the model.

```c
m = Prophet(changepoint_range=0.8, changepoint_prior_scale=0.05)
```

Prophet detects changepoints by first specifying many potential changepoints at which the rate is allowed to change. It then puts a sparse prior on the magnitudes of the rate changes (equivalent to L1 regularization). This essentially means that Prophet has many possible places where the rate can change but will use as few of them as possible (see [here](https://facebook.github.io/prophet/docs/diagnostics.html) for more info). By default, Prophet specifies 25 potential changepoints uniformly placed in the first 80% of the time series. If the trend changes are being overfitted (too much flexibility) or underfit (not enough flexibility), you can adjust the strength of the sparse prior using the input argument changepoint\_prior\_scale. By default, this parameter is set to 0.05. Increasing it will make the trend more flexible. If the trend changes are being overfitted (too much flexibility) or underfit (not enough flexibility), you can adjust the strength of the sparse prior using the input argument *changepoint\_prior\_scale*. By default, this parameter is set to 0.05. Increasing it will make the trend more flexible. For example, in the figure below, I increased the *changepoint\_prior\_scale* to 0.95, and you can see that the prediction scale changes accordingly.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*BbUtV97ndUe8trzBwX4Atw.png)

changepoint\_prior\_scale was set to 0.95

Prophet will, by default, fit weekly and yearly seasonalities if the time series is more than two cycles long. It will also fit daily seasonality for a sub-daily time series. We can add other seasonalities (monthly, quarterly, hourly) using the `add_seasonality` method (Python). You can add a *period* to the seasonality (0.5 = 12 hours, 0.04 = 1 hour). You can also add fourier\_order, to what extent it can capture points at one time (bigger = wider).

```c
m.add_seasonality(name='hourly', period=0.04, fourier_order=20)
```

After fitting the model, we need to create a dataframe to make future predictions with hourly frequency because, by default, it produces daily.

```c
m.fit(sale)
future = m.make_future_dataframe(periods=48, freq='H')
forecast = m.predict(future)
```

Then, we can visualize the result:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*dj5oTdNIjHA0re5dNE4MYQ.png)

Facebook Prophet forecasting

If we look at the forecasting dataframe, we can see that it includes the prediction along with a confidence level:

```c
forecast_df = forecast[['ds','yhat','yhat_upper','yhat_lower']]
```
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*4Lpktn4MeQdpLvdFgMN1UQ.png)

Result of forecasting

We can define an error and a confidence level based on lower and upper-level predictions. We can merge this dataframe with the sale dataset to get the values. We can use a `factor` to compare the error with the confidence level to identify the outliers. If the error is greater than the uncertainty factor (e.g., `1.5` greater than uncertainty), we can consider that specific record as a potential anomaly.

```c
#Merging two dataset to have the actual and prediction values
forecasting_final = pd.merge(forecast_df, sale, how='inner',
                                     left_on = 'ds', right_on = 'ds')

# We calculate the prediction error here and uncertainty 
forecasting_final['error'] = forecasting_final['y'] - forecasting_final['yhat']
forecasting_final['uncertainty'] = forecasting_final['yhat_upper'] - forecasting_final['yhat_lower']

# We this factor we can identify the outlier or anomaly. 
# This factor can be customized based on the data
factor = 1.5
forecasting_final['anomaly'] = forecasting_final.apply(lambda x: 'Yes' 
      if(np.abs(x['error']) >  factor*x['uncertainty']) else 'No', axis = 1)
```

In this way, we can spot possible anomalies in the data.

```c
color_discrete_map = {'Yes': 'rgb(255,12,0)', 'No': 'blue'}
fig = px.scatter(forecasting_final, x='ds', y='y', color='anomaly', title='Anomaly',
                 color_discrete_map=color_discrete_map)
fig.show()
```

And visualize it using the [Plotly library](https://plotly.com/python/) in Python:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*2bEXKe1xBdhsW5BPP6Cyjg.png)

## Model evaluation

To evaluate the `facebook prophet` model, we can use *Mean Absolute Error (MAE)*, *Median Absolute Error (MedAE)*, *Mean Squared Error (MSE)*, *Root Mean Squared Error (RMSE)*, *and Mean Absolute Percentage Error (MAPE)* metrics.

```c
from sklearn.metrics import mean_absolute_error, median_absolute_error, mean_absolute_percentage_error, mean_squared_error
from math import sqrt
from fbprophet.plot import add_changepoints_to_plot

# Mean Absolute Error (MAE)
MAE = mean_absolute_error(forecasting_final['yhat'],forecasting_final['y'])
print('Mean Absolute Error (MAE): ' + str(np.round(MAE, 2)))

# Median Absolute Error (MedAE)
MEDAE = median_absolute_error(forecasting_final['yhat'],forecasting_final['y'])
print('Median Absolute Error (MedAE): ' + str(np.round(MEDAE, 2)))

# Mean Squared Error (MSE)
MSE = mean_squared_error(forecasting_final['yhat'],forecasting_final['y'])
print('Mean Squared Error (MSE): ' + str(np.round(MSE, 2)))

# Root Mean Squarred Error (RMSE) 
RMSE = sqrt(int(mean_squared_error(forecasting_final['yhat'],forecasting_final['y'])))
print('Root Mean Squared Error (RMSE): ' + str(np.round(RMSE, 2)))

# Mean Absolute Percentage Error (MAPE)
MAPE = mean_absolute_percentage_error(forecasting_final['yhat'],forecasting_final['y'])
print('Mean Absolute Percentage Error (MAPE): ' + str(np.round(MAPE, 2)) + ' %')
```

Which gives us a MAP of `0.04` which is considered a low prediction error:

```c
Mean Absolute Error (MAE): 1073.51
Median Absolute Error (MedAE): 760.0
Mean Squared Error (MSE): 2435341.22
Root Mean Squared Error (RMSE): 1560.56
Mean Absolute Percentage Error (MAPE): 0.04 %
```

## Responses (2)

Write a response[What are your thoughts?](https://medium.com/m/signin?operation=register&redirect=https%3A%2F%2Fmedium.com%2F%40reza.rajabi%2Foutlier-and-anomaly-detection-using-facebook-prophet-in-python-3a83d58b1bdf&source=---post_responses--3a83d58b1bdf---------------------respond_sidebar------------------)

```c
Very nice article !
```

```c
hmmmmm
```