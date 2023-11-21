+++
author = "Andreas  Lay"
title = "A Primer on SARIMAX"
date = "2023-11-21"
description = "Generating & modelling synthetic time series data"
tags = ["python", "sarimax", "time series analysis", "forecasting"]
categories = ["Python", "Time Series Analysis", "Forecasting"]
ShowToc = true
TocOpen = true
+++

A while ago I created a notebook with an introduction to time series analysis. [Here is this notebook as a Gist](https://gist.github.com/layandreas/6e47f069418e9f21b9ed7d8e31c6310b):

- Generate a synthetic time series with cycles, trend (random walk) and noise components
- Look at some descriptive statistics (e.g. autocorrelations)
- Model the synthetic data with a SARIMA model

Working with synthetic data first forces you to be explicit about your assumptions and is great for debugging: Unlike real data, as you know the true process the synthetic data follows you can validate your estimates easily against the "true" values. In my experience it's also generally easier and more intuitive when it comes to teaching basic time series concepts.
