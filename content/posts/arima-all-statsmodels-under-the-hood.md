+++
author = "Andreas Lay"
title = "ARIMA Models in Python: All Just Statsmodels Under The Hood?"
date = "2025-05-11"
description = "Taking a closer look at popular time series libraries' implementation of ARIMA models"
tags = ["python", "sarimax", "time series analysis", "forecasting"]
categories = ["Python", "Time Series Analysis", "Forecasting"]
ShowToc = true
TocOpen = true
+++

## What's ARIMA and Why Should You Care?

If you're working with time series and you need to produce forecasts, [autoregressive moving-average models (_AR(I)MA_)](https://en.wikipedia.org/wiki/Autoregressive_moving-average_model) are still a good place to start. But which Python implementation should you use (if you don't want to use R)?

Recently I've been again looking into what the Python ecosystem has to offer in regards to time series analysis in general and `ARIMA` models in particular. There are quite a few options, however you should have a rough understanding what's happening under the hood: Are we dealing with a **framework** that wraps existing libraries or **native implementations**?

## Time Series Packages in the Python Ecosystem

### statsmodels: The Original

[statsmodels](https://www.statsmodels.org/stable/index.html) is the classic Python package for estimating `ARIMA` models. [statsmodels.tsa.arima.model.ARIMA](https://www.statsmodels.org/stable/generated/statsmodels.tsa.arima.model.ARIMA.html) provides support for standard `(p,d,q)` ARIMA models, seasonal orders `(P,D,Q)` as well as exogenous regressors. `statsmodels` implements the `ARIMA` model itself, sped up with [Cython](https://cython.readthedocs.io/en/latest/src/tutorial/cython_tutorial.html).

### sktime: A Unified Time Series API

[sktime](https://www.sktime.net/en/stable/) is _"an easy-to-use, easy-to-extend, comprehensive python framework for ML and AI with time series"_ which provides a _"unified API for ML/AI with time series, for model building, fitting, application, and validation"_. So this means that `sktime` wraps other libraries to provide one API.

Let's take a look at the source repo, specifically [sktime/forecasting/arima](https://github.com/sktime/sktime/tree/main/sktime/forecasting/arima): This folder contains the modules `_statsmodels.py` and `_pmdarima.py`.

- `_statsmodels.py`: This is a wrapper around [statsmodels.tsa.arima.model.ARIMA](https://www.statsmodels.org/stable/generated/statsmodels.tsa.arima.model.ARIMA.html)

- `_pmdarima.py`: This is a wrapper around [pmdarima.arima.auto_arima](https://alkaline-ml.com/pmdarima/modules/generated/pmdarima.arima.auto_arima.html)

We already know `statsmodels`! But how about `pmdarima`?

### pmdarima: Automatic ARIMA Model Selection

Surely `pdmarima` implements its own `ARIMA` estimation! Let's take a look at the [docs](https://pypi.org/project/pmdarima/):

> _Pmdarima wraps statsmodels under the hood, but is designed with an interface that's familiar to users coming from a scikit-learn background._

It's `statsmodels` again!

### Darts: User-friendly Forecasting

Now let's take a look at the `Darts` package. The module [darts/models/forecasting/arima.py](https://github.com/unit8co/darts/blob/master/darts/models/forecasting/arima.py) contains the [ARIMA class](https://unit8co.github.io/darts/generated_api/darts.models.forecasting.arima.html) implementation. And it's also a wrapper around `statsmodels.tsa.arima.model.ARIMA`!

There's also an [AutoARIMA](https://unit8co.github.io/darts/generated_api/darts.models.forecasting.sf_auto_arima.html) class in the [darts/models/forecasting/sf_auto_arima.py](https://github.com/unit8co/darts/blob/master/darts/models/forecasting/sf_auto_arima.py) module. This one actually wraps the `statsforecast` package, not to cofuse with `statsmodels`.

### statsforecast: Fast Forecasting

The [statsforecast](https://github.com/Nixtla/statsforecast) package provides us with an `ARIMA` class in [python/statsforecast/models.py](https://github.com/Nixtla/statsforecast/blob/main/python/statsforecast/models.py#L1755). This module in turn imports their own compiled `C++` implementation [src/arima.cpp](https://github.com/Nixtla/statsforecast/blob/main/src/arima.cpp). So finally, no wrapper around `statsmodels`!

But even `statsforecast` relies on `statsmodels`: To handle exogenous regressors, [statsmodels' ordinary least squares (sm.OLS)](https://github.com/Nixtla/statsforecast/blob/main/python/statsforecast/arima.py#L399) is used!

### autogluon: Automated Time Series Forecasting

Last but not least [autogluon](https://auto.gluon.ai/stable/index.html) aims to provide easy-to-use automated time series forecasting. Looking at the module [timeseries/src/autogluon/timeseries/models/local/statsforecast.py](https://github.com/autogluon/autogluon/blob/9f0475abc7981fa55ccd8f917a98fb2c60a5c930/timeseries/src/autogluon/timeseries/models/local/statsforecast.py) reveals its dependency on `statsforecast`, among others [statsforecast.models.AutoARIMA](https://github.com/autogluon/autogluon/blob/9f0475abc7981fa55ccd8f917a98fb2c60a5c930/timeseries/src/autogluon/timeseries/models/local/statsforecast.py#L165).

So this is actually the first library we found without any reliance on `statsmodels`. But that hasn't always been the case: [This commit from April 2024](https://github.com/autogluon/autogluon/commit/57e8ae19fe4ca39f33a69f884fe645e60abfc5b1) removed a former `statsmodels` dependency!

## statsmodels vs. statsforecast

So `ARIMA` models in the Python ecosystem apparently rely on either the `statsmodels` or `statsforecast` implementations (or both). Why would you choose on over the other? I have not benchmarked it myself yet, but [statsforecast claims to be around 4x faster than statsmodels](https://github.com/Nixtla/statsforecast?tab=readme-ov-file#highlights). So if performance matters to you, i.e. because you need to run a large amount of `ARIMA` models, you might want to check out `statsforecast` or `statsforecast` based frameworks.

## Lesson Learned: Always Spend Some Time Understanding Your Tools

I was actually surprised to find out that all of the above time series packages—with the exception of `autogluon`—depend on the `statsmodels` package!
While it makes sense that not each library reinvents the wheel it isn't always immediately clear what the respective library offers vs. what core functionality is inherited from another package. To be fair: Each package is generally transparent about this.

Nevertheless I recommend to at least take a quick look at the underlying source code and ask yourself: _Do you really need additional layers of abstractions a "wrapper" package offers?_
