# Calculating New Market Beta proposed by Welch (2019)
[Welch (2019)](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3371240) introduces a novel method for calculating Market Beta that outperforms traditional estimators in predicting future OLS market-beta.

Empirical research has shown that the Capital Asset Pricing Model (CAPM) does not hold, making it challenging to predict future returns using market Beta reliably. However, Market Beta is still valuable for investors who want to hedge against market risk factors. The Welch Method outperforms other measures in calculating the Market Beta. 

## What is this measure?

1) it uses daily stock returns instead of monthly returns as inputs. (The analysis is based on 5 years of observation; we need at least 250 daily stock returns) 
2) Employs a smooth exponential aging process for past returns, accounting for the time-varying and mean-reverting nature of true betas
3) it removes outliers in daily stock returns. (Welch proposes to winsorize daily stock return by Delat of 3 around the market return. For instance, if the daily market return is 5%, the range for daily stock return is (1 ± 3): (-2, 4) of market return or -10% and 20%. )

## Download the data: 
We need historical daily returns to calculate the measure. I use the data from CRSP on WRDS. This data can be downloaded directly from WRDS or using Python. In the code [download_data](https://github.com/aghalandar/New-Market-Beta/blob/8cf9237f13b62c18f4a66222f34ef6bbbf835c20/download_data.ipynb), I use WRDS connection to download the CRSP data. Alternatively, other databases with similar data can be used to achieve comparable results.

## Code: 
<pre><code class="language-python">
import pandas as pd
from datetime import datetime
from dateutil.relativedelta import relativedelta
import numpy as np
from statsmodels.regression.linear_model import WLS, OLS

def welch_beta(year_start=2023, year_finish=2023, rw_mon=60, rho=2/252, delta=3):
    """
    Calculate Welch betas for a given range of years and months.

    Parameters:
    - year_start (int): The starting year for the calculation (default: 2023).
    - year_finish (int): The ending year for the calculation (default: 2023).
    - rw_mon (int): The number of months to consider for each calculation (default: 60).
    - rho (float): The decay factor for age decay calculation (default: 2/252).
    - delta (int): The delta value for winsorizing the excess return (default: 3).

    Returns:
    - result (DataFrame): A DataFrame containing the calculated Welch betas for each permno and mdate.

    """
    # Load your datasets here
    daily_data = pd.read_csv(r'D:\OneDrive - Concordia University - Canada\github\history_data.csv')
    factors_daily = pd.read_csv(r'D:\OneDrive - Concordia University - Canada\github\ff.csv')
    factors_daily['date'] = pd.to_datetime(factors_daily['date'], format='%Y%m%d')
    daily_data.loc[:,'ret'] = daily_data.loc[:,'ret'] * 100
    daily_data['date'] = pd.to_datetime(daily_data['date'])

    # attach the rf and market return
    _dset0 = pd.merge(daily_data, factors_daily[['date','MKT','Mkt-RF','RF']], on='date')

    result = pd.DataFrame()  # Initialize an empty DataFrame to store the results

    for yy in range(year_start, year_finish + 1):
        for mm in range(1, 13):
            # the month of calculation: the end date
            mdate = pd.to_datetime(f'{yy}-{mm}-01')
            # start date
            sdate = mdate - relativedelta(months=rw_mon)
            # Extract the most recent rw_mon months of daily data
            _dset1 = _dset0[(_dset0['date'] >= sdate) & (_dset0['date'] < mdate)]
            # calculate the excess return
            _dset1.loc[:,'exret'] = _dset1.loc[:,'ret'] - _dset1.loc[:,'RF']
            _dset1 = _dset1.groupby('permno').filter(lambda x: x['exret'].count() >= 250)
            _dset1 = _dset1.sort_values(['permno', 'date'])
            _dset1['n'] = _dset1.groupby('permno').cumcount()

            # age decay for each stock
            _dset1['agew'] = np.exp(rho * _dset1['n'])

            # winsorize the excess return based on market return and delta
            _dset1['exrlo'] = _dset1['MKT'].apply(lambda x: min((1 - delta) * x, (1 + delta) * x))
            _dset1['exrhi'] = _dset1['MKT'].apply(lambda x: max((1 - delta) * x, (1 + delta) * x))
            _dset1['exretwins'] = _dset1.apply(lambda row: min(max(row['exrlo'], row['exret']), row['exrhi']), axis=1)
            _dset1 = _dset1.drop(['exrlo', 'exrhi', 'n', 'RF'], axis=1)

            # Calculate welch betas
            model = _dset1.groupby('permno').apply(lambda x: WLS(x['exretwins'], x['Mkt-RF'], weights=x['agew']).fit().params['Mkt-RF'], include_groups=False)
            model_df1 = pd.DataFrame({'permno': model.index, 'mdate': mdate, 'bMkt_welch': model.values})

            # Calculate old market betas for comparison: no winsorize, not weight
            model2 = _dset1.groupby('permno').apply(lambda x: OLS(x['exret'], x['Mkt-RF']).fit().params['Mkt-RF'], include_groups=False)
            model_df2 = pd.DataFrame({'permno': model2.index, 'mdate': mdate, 'bMkt': model2.values})

            merged_df = pd.merge(model_df1, model_df2, on=['permno', 'mdate'])

            result = pd.concat([result, merged_df], ignore_index=True)

    return result
</code></pre>

