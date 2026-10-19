# Methodology and Data Notes

This document sets out how the [UK Electricity Demand & Market Performance](README.md) dashboards were prepared, what the main measures mean, and where the results need care in interpretation.

**Coverage:** Great Britain, **1 January 2025–31 August 2026**. All year-on-year comparisons use **January–August in both years**.

## 1. Source data

### NESO demand and embedded generation

[NESO Historic Demand Data](https://www.neso.energy/data-portal/historic-demand-data) provides National Demand (`ND`) and estimates of embedded wind and solar generation at settlement-period level. Figures are in **MW**.

The project uses `demanddata_2025.csv` for 2025 and `demanddataupdate_2026.csv` for 2026. The latter is a **curated analysis copy**: Actual (`A`) records for **21–31 August 2026** from the [NESO Daily Demand Update](https://www.neso.energy/data-portal/daily-demand-update/demand_data_update) were added to the earlier 2026 historic data to reach the project cutoff. It must not be described as an untouched download.

The retained demand series has **17,520** settlement-period records in 2025 and **11,662** in January–August 2026: **29,182 records** across **608 days**. NESO National Demand is an operational definition rather than a direct count of all end-user consumption. Embedded wind and solar are estimates for distribution-connected generation, not all GB renewable output. NESO may retrospectively revise historic data.

### Elexon earliest day-ahead forecast

The forecast source is Elexon's [Earliest Day-Ahead Demand Forecast](https://bmrs.elexon.co.uk/api-documentation/endpoint/forecast/demand/day-ahead/earliest), using the **National Demand** forecast rather than Transmission System Demand. Each earliest published forecast is matched to NESO actual National Demand for the same date and settlement period.

This gives a consistent **earliest day-ahead** comparison. It does not imply that every forecast was published at exactly the same number of hours before delivery. The final accuracy measures do not use the *Latest* forecast series; comparing forecast revisions would need explicit checks of publication times.

### Elexon Market Index

[Elexon Market Index](https://bmrs.elexon.co.uk/api-documentation/endpoint/balancing/pricing/market-index) provides prices (**£/MWh**) and volumes by settlement period and provider. This project uses **`APXMIDP`** as its selected price reference; it does not claim to represent all GB wholesale or retail electricity prices.

An **active APX observation** is an `APXMIDP` record where **`Volume > 0`**. Through the cutoff, the retained APXMIDP series has **29,163 active observations** and **19 zero-volume records**. Zero-volume rows remain in the underlying source data but are excluded from active-price measures. Negative prices are retained when the observations are otherwise valid.

## 2. Preparing and relating the data

| Decision | Treatment |
| --- | --- |
| Time coverage | All of 2025; January–August 2026 |
| Year-on-year comparison | January–August against January–August |
| Date formats | Source dates standardised for Tableau relationships |
| Demand and forecast level of detail | Settlement Date × Settlement Period |
| Market level of detail | Settlement Date × Settlement Period × Data Provider |
| APX prices | `APXMIDP`, with `Volume > 0` for active-price metrics |
| Extreme and negative prices | Retained rather than automatically removed |
| Forecast benchmark | Earliest published day-ahead National Demand forecast |

The checked source data contains **29,182 matching actual-demand and earliest-forecast date/period observations**. Joining a settlement-period fact table to another fact table on date alone would produce duplicates, so the Tableau workbook uses **logical relationships** instead of one flattened join.

The model consists of five logical tables:

| Table | Content |
| --- | --- |
| `FactDemand` | NESO National Demand and estimated embedded wind/solar |
| `FactForecastEarliest` | Elexon earliest day-ahead National Demand forecast |
| `FactMarket` | Elexon Market Index prices, volumes and providers |
| `DimDate` | Date and derived calendar attributes |
| `DimSettlementPeriod` | Settlement-period identifiers |

The three fact tables each relate to the date and settlement-period dimensions: **six relationships** in total. The published workbook uses **18 worksheets across five dashboards**. Historical validation and exploratory sheets are kept separately in the local Source Audit workbook, not added as extra public dashboard tabs.

## 3. Measures and calculations

For the forecast measures, **Actual** is NESO National Demand and **Forecast** is Elexon's earliest day-ahead National Demand forecast. Each observation represents a matched date and settlement period.

| Measure | Calculation | What it tells us |
| --- | --- | --- |
| Forecast error | `Forecast − Actual` | Positive means the forecast overestimated actual demand |
| MAE (MW) | `AVG(ABS(Forecast − Actual))` | Average size of the miss, regardless of direction |
| MAPE (%) | `100 × AVG(ABS(Forecast − Actual) / Actual)` | Average absolute error as a percentage of actual demand |
| Bias (MW) | `AVG(Forecast − Actual)` | Whether forecasts tend to run high or low |
| RMSE (MW) | `SQRT(AVG((Forecast − Actual)^2))` | Emphasises larger forecast errors |
| APX volume-weighted price (£/MWh) | `SUM(Active APX Price × Volume) / SUM(Active APX Volume)` | Prices weighted by the volume of eligible APXMIDP observations |

MAPE requires actual demand to be non-zero; eligible records were checked in the retained source data. **The volume-weighted headline APX KPI is not the same measure as the mean active prices shown in monthly and intraday price charts.** They answer different questions and should not be compared as though they used identical weighting.

### Like-for-like results: January–August

| Measure | 2025 | 2026 | Difference |
| --- | ---: | ---: | ---: |
| Average National Demand | 25,454.46 MW | 25,459.07 MW | **+0.02%** |
| Earliest forecast MAPE | 2.9941% | 3.3572% | **+0.363 percentage points** |
| Volume-weighted active APX price | £84.21/MWh | £100.50/MWh | **+19.3%** |

Across the full **January 2025–August 2026** window, checked headline values include **25,878.73 MW** average National Demand, **47,382 MW** peak demand, **754.08 MW** forecast MAE, **3.06%** MAPE, **−30.39 MW** Bias, **1,030.79 MW** RMSE and **£88.97/MWh** volume-weighted active APX price.

These metrics were reconciled against retained source CSVs and displayed Tableau results. This was **not** a separate row-by-row query of every record inside the packaged `.hyper` extract.

## 4. Daylight saving time

GB electricity settlement normally uses **48 half-hour periods per day**. The dataset includes the expected clock-change exceptions:

| Date | Clock change | Settlement periods |
| --- | --- | ---: |
| 30 March 2025 | Forward | 46 |
| 26 October 2025 | Back | 50 |
| 29 March 2026 | Forward | 46 |

Settlement Periods **49 and 50** occur only on **26 October 2025** in this dataset. Each has one relevant active APX observation and one matching demand/forecast observation. The two APX prices are **−£4.00/MWh** and **−£4.68/MWh**. They are valid negative prices, but each period's average is based on a single observation and should not be presented as a normal intraday pattern.

## 5. Validation and quality checks

The project checks included:

- Confirming date coverage, expected settlement-period counts and one actual-demand and earliest-forecast record per date/period key.
- Matching forecast observations to actual demand at the same date and settlement period.
- Reconciling the main KPIs and January–August comparisons against calculations from the saved source CSVs.
- Separating **29,163 active APXMIDP** records from **19 zero-volume** records.
- Retaining valid negative prices and checking unusually high values rather than removing them automatically.
- Testing both interactive Tableau filter actions and confirming that selection and reset work as intended.

### A Tableau display caveat

During chart checks, a **“19 nulls”** indicator appeared on an active-price scatter view. Comparing the active-price calculation with the raw price measure showed that the calculated measure produces NULL for inactive observations. The number matches the **19 zero-volume APXMIDP records**, but the exact Tableau relationship/filter processing behind the visible warning was **not independently established**. The indicator was hidden for presentation; this did not remove the underlying data or change the active-price definition. We do not treat the warning as evidence of 19 missing source prices or claim that its root cause was fully diagnosed.

## 6. Interpreting the results

- **Price versus demand:** APX prices increased by about **19.3%** between the two January–August windows while average demand changed by roughly **0.02%**. This does not establish the cause of the price change. The Demand–APX fitted trend has **R² ≈ 0.157**, so this single-variable relationship leaves much of the price variation unexplained.
- **Midday forecast errors:** Settlement-period MAE peaks around **SP27 (approximately 1,181 MW)**. Embedded solar follows a daytime pattern too, but patterns that share a time of day are not evidence that one causes the other. A correlation calculated across aggregated half-hour averages can be misleading for that reason.
- **Wind and solar:** NESO embedded-generation figures are estimates. Any scatterplot relationship with APX prices is descriptive and does not isolate the effect of renewable generation from other market conditions.
- **Forecast accuracy over time:** January–August MAPE rises from **2.99%** to **3.36%**, but two historical windows are not enough to establish a permanent change in forecasting performance.

## 7. Viewing and reusing the work

The **[Tableau Public project](https://public.tableau.com/app/profile/yasin.yalcin/viz/UKElectricityDemandMarketPerformance2025FullYearJanAug2026/ExecutiveOverview)** can be explored in a browser. To examine the actual workbook, open `dashboard/UK_Electricity_Demand_and_Market_Performance.twbx` in a compatible **Tableau Desktop or Tableau Public Desktop Edition**. The `.twbx` package includes the analytical extract.

The project is a historical snapshot. Refreshing the workbook or replacing the locally curated 2026 NESO CSV with a newer official download may change the scope and the displayed results. Recheck the date cutoff, field mapping and headline KPIs before making such changes.

## 8. Data licences and attribution

- **NESO:** Supported by National Energy SO Open Data. [NESO Open Licence](https://www.neso.energy/data-portal/neso-open-licence), [Historic Demand Data](https://www.neso.energy/data-portal/historic-demand-data). The licence applies to datasets explicitly covered by it; check each dataset's terms.
- **Elexon:** Contains BMRS data © Elexon Limited copyright and database right 2025–2026. [Licence to use BMRS open data](https://www.elexon.co.uk/bsc/data/balancing-mechanism-reporting-agent/copyright-licence-bmrs-data/), [Elexon Insights API](https://bmrs.elexon.co.uk/api-documentation/).

These source acknowledgements do not imply that NESO or Elexon endorse the dashboards or their conclusions.
