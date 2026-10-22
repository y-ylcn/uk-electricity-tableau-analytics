# Methodology and Data Notes

This document sets out how the [UK Electricity Demand & Market Performance](README.md) dashboards were prepared, what the main measures mean, and where the results need care in interpretation.

**Coverage:** Great Britain, **1 January 2025–31 August 2026**. All year-on-year comparisons use **January–August in both years**.

## 1. Source data

### NESO demand and embedded generation

[NESO Historic Demand Data](https://www.neso.energy/data-portal/historic-demand-data) provides National Demand (`ND`) and estimates of embedded wind and solar generation at settlement-period level. Figures are in **MW**.

The project uses `demanddata_2025.csv` for 2025 and `demanddataupdate_2026.csv` for 2026. The 2026 file is a **curated analysis copy**: Actual (`A`) records for **21–31 August 2026** from the [NESO Daily Demand Update](https://www.neso.energy/data-portal/daily-demand-update/demand_data_update) were appended to the earlier 2026 historic extract. The combined file is a fixed project snapshot rather than an unchanged official download.

The retained demand series has **29,182 settlement-period records** (all of 2025 and January–August 2026). NESO National Demand is an operational definition rather than a direct count of all end-user consumption. Embedded wind and solar are estimates for distribution-connected generation, not all GB renewable output. NESO may retrospectively revise historic data.

### Elexon earliest day-ahead forecast

The forecast source is Elexon's [Earliest Day-Ahead Demand Forecast](https://bmrs.elexon.co.uk/api-documentation/endpoint/forecast/demand/day-ahead/earliest), using the **National Demand** forecast rather than Transmission System Demand. Each earliest published forecast is matched to NESO actual National Demand for the same date and settlement period.

This gives a consistent **earliest day-ahead** comparison. It does not imply that every forecast was published at exactly the same number of hours before delivery. The final accuracy measures do not use the *Latest* forecast series; comparing forecast revisions would need explicit checks of publication times.

### Elexon Market Index

[Elexon Market Index](https://bmrs.elexon.co.uk/api-documentation/endpoint/balancing/pricing/market-index) provides prices (**£/MWh**) and volumes by settlement period and provider. This project uses **`APXMIDP`** as its selected price reference; it does not claim to represent all GB wholesale or retail electricity prices.

APX was selected because it provides substantially better active-period coverage. Through the cutoff, **29,163 of 29,182 APXMIDP records** have positive volume, compared with **79 of 28,991 N2EXMIDP records**. A direct all-period comparison of average provider prices would therefore be misleading.

An **active APX observation** is an `APXMIDP` record with **`Volume > 0`**. The remaining **19 zero-volume APX records** are retained for quality checks but excluded from active-price measures. Zero-volume values may reflect market-index liquidity or defaulting rules and should not automatically be treated as traded prices. Valid negative prices are retained.

## 2. Preparing and relating the data

| Decision | Treatment |
| --- | --- |
| Time coverage | All of 2025; January–August 2026 |
| Year-on-year comparison | January–August against January–August |
| Date formats | Year-specific source formats parsed to a common Date type |
| Demand and forecast level of detail | Settlement Date × Settlement Period |
| Market level of detail | Settlement Date × Settlement Period × Data Provider |
| APX prices | `APXMIDP`, with `Volume > 0` for active-price metrics |
| Extreme and negative prices | Retained rather than automatically removed |
| Forecast benchmark | Earliest published day-ahead National Demand forecast |

The Elexon Market Index source files use `DD/MM/YYYY` dates in 2025 and `YYYY-MM-DD` in 2026. The workbook's `Market Settlement Date` calculated field handles both formats:

```tableau
IF [Market Source File] = "market_index_2025.csv" THEN
    DATE(DATEPARSE('dd/MM/yyyy', [Market Settlement Date Raw]))
ELSEIF [Market Source File] = "market_index_2026.csv" THEN
    DATE(DATEPARSE('yyyy-MM-dd', [Market Settlement Date Raw]))
END
```

Settlement periods are identified by date and period number, with daylight saving time handled separately. The checked source data contains **29,182 matching actual-demand and earliest-forecast date/period observations**. Joining a settlement-period fact table to another fact table on date alone would produce duplicates, so the Tableau workbook uses **logical relationships** instead of one flattened join.

The model consists of five logical tables:

| Table | Content |
| --- | --- |
| `FactDemand` | NESO National Demand and estimated embedded wind/solar |
| `FactForecastEarliest` | Elexon earliest day-ahead National Demand forecast |
| `FactMarket` | Elexon Market Index prices, volumes and providers |
| `DimDate` | Date and derived calendar attributes |
| `DimSettlementPeriod` | Settlement-period identifiers |

The three fact tables each relate to the date and settlement-period dimensions: **six relationships** in total. This preserves the provider-level market records without inflating National Demand aggregates. The published workbook uses **24 worksheets across six dashboards**.

## 3. Measures and calculations

The formulas below use the field names shown in Tableau and match the calculated fields saved in the workbook.

### Forecast accuracy

The forecast fields compare `Earliest Forecast National Demand` with `Actual National Demand` for the same settlement date and period.

| Tableau calculated field | Formula |
| --- | --- |
| `Forecast Error` | `SUM([Earliest Forecast National Demand]) - SUM([Actual National Demand])` |
| `Absolute Forecast Error` | `ABS([Forecast Error])` |
| `Absolute Percentage Error` | `IF SUM([Actual National Demand]) <> 0 THEN ABS([Forecast Error] / SUM([Actual National Demand])) END` |
| `MAE - Mean Absolute Error` | `WINDOW_AVG([Absolute Forecast Error])` |
| `MAPE - Mean Absolute Percentage Error` | `WINDOW_AVG([Absolute Percentage Error])` |
| `Bias - Mean Forecast Error` | `WINDOW_AVG([Forecast Error])` |
| `RMSE - Root Mean Squared Error` | `SQRT(WINDOW_AVG([Forecast Error] * [Forecast Error]))` |

A positive forecast error or bias means over-forecasting; a negative value means under-forecasting. MAE describes the average absolute miss in MW, while RMSE gives larger errors more weight. **MAPE is stored as a fraction and formatted as a percentage in Tableau**; the saved calculation does not multiply by 100. Its denominator is guarded against zero actual demand.

### Demand and embedded generation

| Tableau calculated field | Formula |
| --- | --- |
| `Average Actual National Demand` | `AVG([Actual National Demand])` |
| `Peak Actual National Demand` | `MAX([Actual National Demand])` |
| `Average Embedded Wind Generation` | `AVG([Embedded Wind Generation])` |
| `Average Embedded Solar Generation` | `AVG([Embedded Solar Generation])` |

`Daily Peak National Demand` is an LOD calculation and is covered in its own section below.

### Active market prices

| Tableau calculated field | Formula |
| --- | --- |
| `Active Market Record` | `[Volume] > 0` |
| `Active Market Price` | `IF [Active Market Record] THEN [Price] END` |
| `Average Active Market Price` | `AVG([Active Market Price])` |
| `Median Active Market Price` | `MEDIAN([Active Market Price])` |
| `Maximum Active Market Price` | `MAX([Active Market Price])` |
| `Volume-Weighted Market Index Price` | `IF SUM([Volume]) > 0 THEN SUM([Price] * [Volume]) / SUM([Volume]) END` |

The **`APXMIDP` provider filter** and the positive-volume rule are applied to the relevant market views. The volume-weighted price calculation has a non-zero denominator guard. The headline volume-weighted APX KPI should not be confused with the **unweighted mean** of active APX prices shown in monthly and settlement-period charts. The conditional `Active Market Price` field is intentionally null for inactive records; this is not the same as a missing raw market `Price` value.

**Year-on-year calculations:** The January–August demand and APX price comparisons use Tableau's native **Percent Difference from Previous** table calculation, comparing 2026 with 2025. This is a view-level table calculation, not a separate named calculated field. The difference between the two forecast MAPE rates is instead reported in **percentage points**.

### Average daily peaks (Supporting Analysis)

The `Jan–Aug Average Daily Peak vs Highest Peak (MW)` worksheet uses the existing `Daily Peak National Demand` level of detail (LOD) expression to find the highest National Demand on each date:

```tableau
{ INCLUDE [Settlement Date] :
    MAX([Actual National Demand])
}
```

The view compares the **average of those daily peaks** with the **single highest settlement-period demand** over the same January–August dates. This is not the average of all half-hourly demand readings, and the full-year 2025 daily-peak average is not used as a comparator for January–August 2026.

| January–August period | Average daily peak | Highest observed peak |
| --- | ---: | ---: |
| 2025 | 31,723.70 MW | 45,924 MW |
| 2026 | 31,983.22 MW | 47,382 MW |

### Like-for-like results: January–August

| Measure | 2025 | 2026 | Difference |
| --- | ---: | ---: | ---: |
| Average National Demand | 25,454.46 MW | 25,459.07 MW | **+0.02%** |
| Earliest forecast MAPE | 2.99% | 3.36% | **+0.36 percentage points** |
| Volume-weighted active APX price | £84.21/MWh | £100.50/MWh | **+19.3%** |

The MAPE difference is calculated from unrounded values.

Across the full **January 2025–August 2026** window, checked headline values include **25,878.73 MW** average National Demand, **47,382 MW** peak demand, **754.08 MW** forecast MAE, **3.06%** MAPE, **−30.39 MW** Bias, **1,030.79 MW** RMSE and **£88.97/MWh** volume-weighted active APX price.

## 4. Daylight saving time

GB electricity settlement normally uses **48 half-hour periods per day**. The dataset includes the expected clock-change exceptions:

| Date | Clock change | Settlement periods |
| --- | --- | ---: |
| 30 March 2025 | Forward | 46 |
| 26 October 2025 | Back | 50 |
| 29 March 2026 | Forward | 46 |

Settlement Periods **49 and 50** occur only on **26 October 2025** in this dataset. Each has one relevant active APX observation and one matching demand/forecast observation. The two APX prices are **−£4.00/MWh** and **−£4.68/MWh**. They are valid negative prices, but each period's average is based on a single observation and should not be presented as a normal intraday pattern.

## 5. Validation and quality checks

The three null-count fields used in the Data Quality dashboard have these saved calculations, shown with Tableau's displayed field names:

| Tableau calculated field | Formula |
| --- | --- |
| `Null Actual Demand Count` | `SUM(IF ISNULL([Actual National Demand]) THEN 1 ELSE 0 END)` |
| `Null Earliest Forecast Count` | `SUM(IF ISNULL([Earliest Forecast National Demand]) THEN 1 ELSE 0 END)` |
| `Null Market Price Count` | `SUM(IF ISNULL([Price]) THEN 1 ELSE 0 END)` |

The `Show KPI Mark` calculation (`LAST() = 0`) displays a single KPI result while preserving the underlying table-calculation window.

The project checks included:

- Confirming date coverage, expected settlement-period counts and one actual-demand and earliest-forecast record per date/period key.
- Matching forecast observations to actual demand at the same date and settlement period.
- Reconciling the main KPIs and January–August comparisons against calculations from the saved source CSVs.
- Confirming that retained actual demand, earliest-forecast demand and source Market Index price fields have **zero null values** in the core quality checks.
- Separating **29,163 active APXMIDP** records from **19 zero-volume** records; inactivity is not the same as a missing source price.
- Retaining valid negative prices and checking unusually high values rather than removing them automatically.
- Testing both scoped dashboard filter actions, including selection and reset, in Tableau Desktop and after Tableau Public publication.
- Checking the average of distinct daily demand peaks and the highest observed peak against the retained NESO records for both January–August windows.

## 6. Interpreting the results

- **Price versus demand:** APX prices increased by about **19.3%** between the two January–August windows while average demand changed by roughly **0.02%**. This does not establish the cause of the price change. The Demand–APX fitted trend has **R² ≈ 0.157**, so this single-variable relationship leaves much of the price variation unexplained.
- **Midday forecast errors:** Settlement-period MAE peaks around **SP27 (approximately 1,181 MW)**. Embedded solar follows a daytime pattern too, but patterns that share a time of day are not evidence that one causes the other. A correlation calculated across aggregated half-hour averages can be misleading for that reason.
- **Wind and solar:** Estimated embedded wind and solar generation both have negative linear associations with active APX prices. The wind relationship is stronger (**R² = 0.186**) than the solar relationship (**R² = 0.044**). These are simple unadjusted relationships; they do not isolate the effect of renewable generation from demand, weather or other market conditions.
- **Forecast accuracy by day type:** Across the retained period, weekday MAPE is **2.92%**, compared with **3.40%** at weekends.
- **Forecast accuracy over time:** January–August MAPE rises from **2.99%** to **3.36%**, but two historical windows are not enough to establish a permanent change in forecasting performance.

## 7. Viewing and reusing the work

The six-dashboard **[Tableau Public project](https://public.tableau.com/app/profile/yasin.yalcin/viz/UKElectricityDemandMarketPerformance2025FullYearJanAug2026/ExecutiveOverview)** can be explored in a browser. To examine the actual workbook, open `dashboard/UK_Electricity_Demand_and_Market_Performance.twbx` in a compatible **Tableau Desktop or Tableau Desktop Public Edition**. The `.twbx` package includes the analytical extract.

The project is a historical snapshot. Refreshing the workbook or replacing the locally curated 2026 NESO CSV with a newer official download may change the scope and the displayed results. Recheck the date cutoff, field mapping and headline KPIs before making such changes.

## 8. Data licences and attribution

- **NESO:** Supported by National Energy SO Open Data. [NESO Open Licence](https://www.neso.energy/data-portal/neso-open-licence), [Historic Demand Data](https://www.neso.energy/data-portal/historic-demand-data). The licence applies to datasets explicitly covered by it; check each dataset's terms.
- **Elexon:** Contains BMRS data © Elexon Limited copyright and database right 2025–2026. [Licence to use BMRS open data](https://www.elexon.co.uk/bsc/data/balancing-mechanism-reporting-agent/copyright-licence-bmrs-data/), [Elexon Insights API](https://bmrs.elexon.co.uk/api-documentation/).

These source acknowledgements do not imply that NESO or Elexon endorse the dashboards or their conclusions.
