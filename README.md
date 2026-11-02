# WTI Oil Price Forecasting (SQL, Python, Tableau)

Forecasting daily WTI crude oil prices, and testing how well simple and advanced models really do, through a sudden change in the market.

**Live dashboard:** https://public.tableau.com/views/OilPricePredictor/Dashboard1

**Tools:** PostgreSQL, Python (pandas, statsmodels, matplotlib), Tableau Public

---

## Summary

- Loaded 5 years of daily WTI oil prices into PostgreSQL, then cleaned and explored them with SQL.
- Found that 2026 is far more volatile (jumpy) than any earlier year in the data.
- Tested four forecasting methods, one day ahead, on 85 days the models had never seen.
- **The simplest model (naive: "tomorrow equals today") was as good as ARIMA and better than the moving average.**
- Built a Tableau dashboard with the price history, forecasts vs actual, model errors, and volatility by year.

---

## Key findings

1. **A regime change in 2026.** A regime change is a shift in how the data behaves. The standard deviation (a number showing how spread out values are) of daily price changes was much higher in 2026 than in any other year. The largest one-day move was a drop of $18.41 on 8 April 2026. This matches the Middle East conflict that began in late February 2026.
2. **Daily oil price changes are close to random.** The correlation between a day's change and the changes in the previous 1 to 5 days stayed between -0.07 and +0.04, so a model has almost nothing to learn from. This is called a random walk (a series where each step is mostly unpredictable).
3. **Naive was hard to beat.** Smoothing hurt, and ARIMA tied with naive. The best choice is the simplest one.
4. **The forecast horizon (how far ahead you forecast) matters a lot.** A flat 4-month forecast was wrong by about $12 to $20 on average. A one-day-ahead forecast was wrong by about $2.40.

---

## Data

- **Source:** FRED series [DCOILWTICO](https://fred.stlouisfed.org/series/DCOILWTICO) (daily WTI spot price, USD per barrel).
- **Range:** 27 Oct 2021 to 27 Oct 2026 (1,305 rows).
- **Gaps:** 57 days had no price (US market holidays). Weekends are not in the file.

---

## Method

### 1. SQL (PostgreSQL)

| Step | What I did |
| --- | --- |
| Load | Loaded the CSV into `oil_prices_raw`. This table is never edited. |
| Check | Checked the date range, duplicate dates, price range, missing values and zero or negative prices. |
| Clean | Built `oil_prices_clean`. Missing prices were filled with the last known price (forward fill), because the market was closed on those days. |
| Daily table | Built `oil_prices_daily` with a `prev_price` column, using the `LAG` window function (a function that looks at the row above). |
| Explore | Ranked the biggest one-day moves. Calculated the standard deviation of daily changes for each year. |
| Results | Loaded the forecasts into `forecast_results`. Recalculated the MAE in SQL to check it matched Python. |
| Dashboard tables | Built `tableau_prices`, `tableau_errors`, `tableau_lines` and `tableau_yearly`. |

**Forward fill (last known price for each date):**

```sql
CREATE TABLE oil_prices_clean AS
SELECT
    observation_date,
    price,
    (
        SELECT r2.price
        FROM oil_prices_raw r2
        WHERE r2.observation_date <= r1.observation_date
          AND r2.price IS NOT NULL
        ORDER BY r2.observation_date DESC
        LIMIT 1
    ) AS price_filled
FROM oil_prices_raw r1
ORDER BY observation_date;
```

**Previous day's price:**

```sql
CREATE TABLE oil_prices_daily AS
SELECT
    observation_date,
    price_filled AS price,
    LAG(price_filled) OVER (ORDER BY observation_date) AS prev_price
FROM oil_prices_clean
ORDER BY observation_date;
```

**Volatility by year:**

```sql
SELECT
    EXTRACT(YEAR FROM observation_date) AS year,
    COUNT(*) AS days,
    AVG(price) AS avg_price,
    STDDEV(price - prev_price) AS daily_change_stddev
FROM oil_prices_daily
WHERE prev_price IS NOT NULL
GROUP BY EXTRACT(YEAR FROM observation_date)
ORDER BY year;
```

### 2. Forecasting (Python)

- **Train/test split by date, not at random.** A random split would let the model see the future.
  - Train: 27 Oct 2021 to 30 Jun 2026 (1,220 days)
  - Test: 1 Jul 2026 to 27 Oct 2026 (85 days)
- **One-day-ahead testing.** For each test day, the forecast only uses real prices from before that day.
- **Settings chosen on training data only.** This avoids data leakage (using test data to make choices, which makes scores look better than they should).

**Models tested**

| Model | Idea |
| --- | --- |
| Naive | Tomorrow's price equals today's price. |
| 5-day moving average | The average of the last 5 prices. |
| Exponential smoothing | Like a moving average, but recent days count more. Tested several alpha values (the setting for how fast it reacts). |
| ARIMA(2, 1, 0) | Predicts the daily change from the last 2 daily changes. |

**Error measures**

- **MAE** (Mean Absolute Error): the average size of the errors.
- **RMSE** (Root Mean Squared Error): like MAE, but big misses count more.

### 3. Dashboard (Tableau Public)

1. Price history, 2021 to 2026
2. Forecasts vs actual price on the test days
3. Average error for each model
4. Volatility by year

---

## Results

One-day-ahead forecasts on the 85 test days:

| Model | MAE (USD) | RMSE (USD) |
| --- | --- | --- |
| Naive | 2.40 | 3.40 |
| ARIMA(2, 1, 0) | 2.43 | 3.42 |
| Exponential smoothing | 2.40 | 3.40 (same as naive) |
| 5-day moving average | 3.61 | 4.45 |

- **Exponential smoothing:** the best alpha on the training data was 1.0, which is exactly the naive model. Every bit of smoothing made the training error worse.
- **Moving average:** it lags (reacts late) when prices move in a run, so it did worst.
- **ARIMA vs naive:** the gap is about 2 cents, which is too small to mean anything with only 85 test days. When two models tie, the simpler one wins.

**Flat 4-month forecasts (a bad horizon, kept as a lesson):**

| Model | MAE (USD) |
| --- | --- |
| Flat naive (last training price, $70.56) | 19.68 |
| Flat average of training prices ($79.09) | 12.64 |

The flat naive forecast froze on a temporary dip at the end of June 2026 and stayed wrong for four months.

---

## Limits

- **The test set is small** (85 days), so close scores can't be separated.
- **2021 and 2026 are partial years**, because the data starts in late October 2021 and ends in late October 2026.
- **The test period is unusual.** It sits inside the volatile 2026 regime, so scores may not carry over to calm markets.
- **Models can't predict news.** Conflicts, deals and supply shocks move oil, and the models only see past prices.
- **No future forecast was published.** This project is about fair testing, not about predicting where oil goes next.
- **The 2020 negative price event is not in the data**, because the series starts in 2021.

## Possible next steps

- Add outside data, such as inventories or the US dollar, and test whether it helps.
- Test longer horizons fairly (1 week, 1 month) with a rolling split.
- Compare calm periods against the 2026 period to see how much each model's error grows.
- Add prediction intervals (a range of likely values, not one number).

---

## How to run it

1. Download `DCOILWTICO.csv` from FRED and save it as `oil_prices.csv`.
2. In PostgreSQL, create a database called `oil_project`, then run the SQL in the `sql/` folder in order.
3. Export `oil_prices_daily` to `oil_daily.csv`.
4. Install the libraries:

   ```
   pip install pandas matplotlib statsmodels
   ```

5. Run the Python scripts in the order below.

## Files

```
README.md
sql/                        SQL scripts (load, clean, explore, dashboard tables)
look_at_data.py             First look at the raw CSV
plot_prices.py              Price chart
baselines.py                Flat 4-month naive and average forecasts
baselines2.py               One-day-ahead naive forecast
moving_average.py           5-day moving average vs naive
exp_smoothing.py            Exponential smoothing with several alpha values
check_pattern.py            Autocorrelation check on daily changes
arima_model.py              ARIMA(2, 1, 0) vs naive
save_results.py             Saves all test-day forecasts and RMSE
```

---

## Author

Moram Khalid