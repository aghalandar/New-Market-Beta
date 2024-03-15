# Calculating New Market Beta proposed by Welch (2019)
[Welch (2019)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3371240) introduces a novel method for calculating Market Beta that outperforms traditional estimators in predicting future OLS market-beta as well as future realized estimates of other beta estimators.

Empirical research has shown that the Capital Asset Pricing Model (CAPM) does not hold, making it challenging to reliably predict future returns using market Beta. However, Market Beta remains a valuable indicator for assessing market-hedge by individual stocks. Welch's method offers improved prediction capabilities for future Beta values.

## What is this measure?

1) it uses daily stock returns instead of monthly returns as inputs
2) Employs a smooth exponential aging process for past returns, accounting for the time-varying and mean-reverting nature of true betas
3) it removes outliers. 

## Download the data: 
We need historical daily return to calculate the measure. I use the data from CRSP on WRDS. This data can be downloaded directly from WRDS or using Python. In the file [download_data](https://github.com/aghalandar/New-Market-Beta/blob/8cf9237f13b62c18f4a66222f34ef6bbbf835c20/download_data.ipynb), I use WRDS connection to download the CRSP data. Alternatively, other databases with similar data can be used to achieve comparable results.

## Code: 

