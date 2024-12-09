+++
title  = "December 2024 Reading List"
date   = "2024-12-01"
author = "Jon"
tags   = ["reading-list"]
+++

# Books

- *Quantitative Trading* by Ernest P. Chan \[1],
- *Forecasting: Principles and Practice* by Rob J Hyndman and George Athanasopoulos \[2].

# Papers

- "Data-driven stock forecasting models based on neural networks: A review" \[3] ([link](https://www.sciencedirect.com/science/article/pii/S1566253524003944)).

# Notes

Towards the beginning of \[1], Chan categorizes trading strategies into two types:

- mean-reversion, and
- momentum-based.

He argues that the movement of a stock price at any given time is determined by one of these strategies: either a stock price is returning to a mean, or it is moving to a new mean. The question, then, is to determine which *regime* is prevailing for a given stock, and act accordingly.

There is a big reason why mean-reversion tactics are hard to apply to a singular stock. Stock price time series' are notoriously non-stationary, meaning that they rarely return to a single value. However, if you find two stock prices that are said to be *cointegrated*, the combination of these two series can be stationary. This is the inspiration for the "mean-reverting pairs-trading" strategy on assets that are usually in the same industry, such as GLD-GDX. They are usually in the same industry because the intrinsic value of these assets are most likely related and thus likely cointegrated. What you are profiting off of then, when you enact this strategy, are the slight statistical pricing inefficiencies of the market between these two assets -- meaning that if they moved perfectly and in proportion with each other, then there would be no opportunity for profit. This is also known as *statistical arbitrage*. Due to the widespread knowledge of this strategy, it has said to have succumbed to the notorious *alpha decay*: the loss of a profitability of a strategy over time.

I find this concept especially fascinating because, in a sense, you are cancelling out noise in one variable via noise in another variable. In \[3], there is mention of a technique called Empirical Mode Decomposition (EMD) which reminds me of this noise-cancel-noise technique.

> Yang et al. [48] utilized integrated EMD to decompose complex original stock price time series into smoother, more regular, and stable subsequences than the original time series, followed by using the LSTM method to train and predict each subsequence.

It should be said however that \[1] and \[3] fundamentally disagree about how complex your trading strategy should be. \[1] argues that trading strategies should be made as simple as possible, mentioning by-name that strategies using neural network-techniques should be avoided. Neural network-based trading strategies infamously perform poorly on new and real market data despite performing well on their test set. \[1] continues saying that, simple trading strategies perform well because they are more resilient to *regime changes* in the market.

# Keywords

- Time Series,
- Sharpe Ratio\*,
- Kelly Formula\*\*.

# Footnotes

\*: Quantifies excess return against risk. If you have a high-return, low-risk strategy, it will have a high Sharpe ratio. A *good* strategy might be one with a Sharpe ratio greater than 1.

\*\*: Quantifies the optimal bet to place to maximize long-term geometric returns. A quantity greater than 1 means that you should leverage your position. However, the Kelly formula is based on a Guassian distribution of returns, but returns on stock prices are historically non-Gaussian (they have "fat-tails"). So, the Kelly formula needs to be adjusted in accordance with tolerable risk. This is why the "half-Kelly formula" exists.

It also takes into account the historical profitability of your strategy. If your strategy suffers a massive historical loss, the profitability of your strategy will decrease. This will, in turn, decrease the Kelly ratio which dictates, unintuitively, that you should deleverage the strategy. This contests with our primal nature to bet more to "make up" for the loss.
