# Hotel Bookings: Delta Lake Pipeline on Databricks

A bronze/silver/gold data pipeline on Databricks that loads 119,390 hotel bookings into
Delta Lake, checks every row against data quality rules, quarantines bad rows instead of
dropping them silently, and publishes cancellation metrics to Unity Catalog.

Built on the same dataset as my
[hotel booking cancellation model](https://github.com/nannurinirupamreddy/hotel_booking_cancellation).
That project cleaned the data in pandas for one model; this one turns the cleaning into a
repeatable pipeline with logged quality checks.

## Pipeline

```
data.csv (Unity Catalog volume)
   |
   v
bookings_bronze      raw rows, all strings, plus ingest time, run id, source file
   |
   |  "NA"/"NULL" -> real nulls, cast types, fill missing values,
   |  drop leaky columns, build arrival_date, run 7 quality rules
   v
bookings_silver      rows that passed every rule (CHECK constraints enforced)
bookings_quarantine  rows that failed, with the list of rules each one broke
dq_check_log         one row per rule per run (appended every run)
   |
   v
cancellation_metrics_gold   cancel rate, avg daily rate, lead time
                            by hotel, month, market segment
```

## Data quality rules

| Rule | What it checks | Rows failing |
|---|---|---|
| `valid_hotel` | hotel is Resort Hotel or City Hotel | 0 |
| `valid_cancel_flag` | is_canceled is 0 or 1 | 0 |
| `has_guests` | at least one adult, child, or baby | 180 |
| `non_negative_price` | average daily rate is not negative | 1 |
| `non_negative_lead_time` | lead time is not negative | 0 |
| `has_stay_nights_or_canceled` | a booking that wasn't canceled has at least one night | 622 |
| `valid_arrival_date` | year, month, and day form a real date | 0 |

803 of 119,390 rows (0.67%) are quarantined. The run fails on purpose if more than 1% of
rows fail, or if any rows go missing between bronze and silver.

Missing values found and filled: `company` (112,593), `agent` (16,340), `country` (488),
`children` (4). `reservation_status` and `reservation_status_date` are dropped because
they are only known after the stay, so they would leak the answer into a cancellation model.

## Results

| | |
|---|---|
| Bronze rows | 119,390 |
| Silver rows | 118,587 |
| Quarantined rows | 803 |
| Gold rows | 328 |
| City Hotel cancel rate | 41.92% |
| Resort Hotel cancel rate | 28.03% |

## Run it (Databricks Free Edition)

1. Sign up at [databricks.com/learn/free-edition](https://www.databricks.com/learn/free-edition).
2. **Workspace > Import**, then upload `hotel_delta_pipeline.py`. It opens as a notebook.
3. Run the first two cells. They create the `workspace.hotel` schema and a `raw` volume.
4. **Catalog > workspace > hotel > raw > Upload to this volume**, and upload `data.csv`
   from the hotel booking repo.
5. **Run all.** The last cell prints the summary numbers above.
6. Open any table in **Catalog** to see its description, columns, history, and lineage.

## Tech

Databricks, Delta Lake (CHECK constraints, table history), Unity Catalog (schema, volume,
table and column comments), PySpark, SQL.
