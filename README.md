# Warehouse Order Forecasting Suite

An end-to-end time-series forecasting project covering two tracks:

- **Track 1 — Retail Store Sales:** LightGBM and XGBoost forecasting pipeline built on the Kaggle Store Sales dataset (Ecuador grocery chain, 54 stores, 33 product families).
- **Track 2 — Warehouse Order Forecasting:** A fully synthetic automotive parts warehouse dataset (built from scratch using Faker) with LightGBM daily-grain SKU-level forecasting across 5 US regional warehouses.

---

## Repository Structure

```
warehouse-order-forecasting/
│
├── notebooks/
│   ├── track_1_store_sales/
│   │   ├── 00_Order_Forecast_Data.ipynb       # Date-shift utility (makes data feel current)
│   │   ├── 01_Order_Forecast_step1.ipynb      # Feature engineering + train/val split
│   │   ├── 02_Order_Forecast_step2.ipynb      # LightGBM training
│   │   ├── 03_Order_Forecast_step3.ipynb      # XGBoost training
│   │   └── 04_Order_Forecast_step4.ipynb      # Model evaluation + comparison
│   │
│   └── track_2_warehouse/
│       ├── 00_Warehouse_Ord_ForeCast_Data.ipynb   # Synthetic data generator
│       ├── 01_WH_ORD_Forecast_Trial1.ipynb        # LightGBM forecast v1 (WH001)
│       └── 02_WH_ORD_Forecast_Trial2.ipynb        # LightGBM forecast v2 - refined (WH001)
│
├── data/
│   ├── store_sales_raw/
│   │   └── DATA_SOURCE.md             # Instructions to download from Kaggle
│   └── warehouse_raw/
│       └── DATA_SOURCE.md             # Instructions to generate synthetic data
│
├── outputs/
│   ├── store_sales/                   # Generated when Track 1 notebooks are run
│   └── warehouse/                     # Generated when Track 2 notebooks are run
│
├── requirements.txt
└── README.md
```

---

## Track 1 — Store Sales Forecasting

### Dataset
The [Kaggle Store Sales - Time Series Forecasting](https://www.kaggle.com/competitions/store-sales-time-series-forecasting) dataset from Corporación Favorita (Ecuador). Download and place all files in `data/store_sales_raw/`.

| File | Description |
|---|---|
| `train.csv` | 3,000,888 rows of daily sales by store and product family |
| `test.csv` | 28,512 rows to forecast |
| `stores.csv` | 54 stores with city, state, type, cluster |
| `holidays_events.csv` | 350 Ecuadorian holidays and events |
| `oil.csv` | Daily oil prices (Ecuador is oil-dependent economy) |
| `sample_submission.csv` | Submission format reference |

### Notebooks in order

**00 — Date Shift (optional)**
Shifts the entire dataset's dates forward by a multiple of 7 days so the data feels current while preserving weekday alignment. Useful for portfolio presentations.

**01 — Feature Engineering**
- Merges train + stores + holidays + oil into one dataframe
- Engineers time features: day of week, week of year, month, year, day, is_weekend
- Engineers lag features: sales at lag 1, 7, 14, 28 days (grouped by store + family)
- Engineers rolling features: 7/14/28-day rolling mean, 28-day rolling std
- Target encoded as log1p(sales) to handle right skew
- Creates a 60-day validation split
- Saves processed parquet files to `Processed/`

**02 — LightGBM**
- 2,844,072 training rows, 24 features
- LGBMRegressor: 2,000 estimators, learning rate 0.03, 63 leaves
- Early stopping at 100 rounds
- Predictions inverse-transformed with expm1
- Output: `pred_lightgbm.csv`

**03 — XGBoost**
- Same feature set as LightGBM
- XGBRegressor: 4,000 estimators, learning rate 0.03, max depth 8, hist tree method
- Output: `pred_xgboost.csv`

**04 — Evaluation**

| Model | RMSE | MAE | RMSLE |
|---|---|---|---|
| LightGBM | 226.67 | 59.60 | 3.131 |
| XGBoost | 227.53 | 58.07 | 3.135 |

Winner: **LightGBM** (best RMSLE). Final output saved as `validation_predictions_all_models.csv` with both predictions and error columns side by side.

### Track 1 output files (generated when notebooks are run — not committed to repo)
```
outputs/store_sales/
├── X_train.parquet
├── X_val.parquet
├── y_train.parquet
├── y_val.parquet
├── val_metadata.parquet
├── pred_lightgbm.csv
├── pred_xgboost.csv
└── validation_predictions_all_models.csv
```

---

## Track 2 — Warehouse Order Forecasting

### Why synthetic data?
Real warehouse order data is confidential. Rather than use a generic public dataset, this project builds a realistic automotive parts warehouse dataset from scratch — encoding real business rules like Pareto ABC demand distribution, regional seasonality, and multi-line orders. This mirrors how real supply chain analytics teams work.

### Synthetic dataset design

**00 — Data Generator** (`Warehouse_Ord_ForeCast_Data.ipynb`)

Generates a full star-schema warehouse dataset using Faker, NumPy, and custom business logic:

| Table | Rows | Description |
|---|---|---|
| `dim_warehouses.csv` | 5 | WH001–WH005 across LA, NY, Orlando, Kansas City, Phoenix |
| `dim_products.csv` | 600 | SKUs across 7 automotive categories with ABC classification |
| `dim_calendar.csv` | 1,461 | Daily calendar 2022–2025 with fiscal year and season flags |
| `fact_orders_header.csv` | ~1,000,000 | One row per order with amounts and status |
| `fact_orders_detail.csv` | ~3,000,000 | One row per order line with SKU, qty, price, bin |

**Business logic built into the data:**
- Pareto demand: Class A SKUs (top 20%) drive 80% of order volume
- Regional seasonality: Tires spike +20% in winter at WH002 (NY) and WH004 (Kansas City); AC Parts spike +20% in summer at WH003 (Orlando) and WH005 (Phoenix)
- Order status mix: Completed 70%, Pending 15%, Backordered 10%, Cancelled 5%
- 1–5 line items per order, 1–50 units per line, 0–25% discount range
- 3,000 bin locations per warehouse

### Forecasting notebooks

**01 — Trial 1** (`WH_ORD_Forecast_Trial1.ipynb`)
- Loads WH001 only: 200,000 orders → 601,213 detail rows → 388,601 daily-grain rows
- Feature matrix: 864,000 rows × 14 columns
- Features: day of week, month, day of year, year, week, is_weekend + lags at 7/14/21 days + rolling means at 7/14 days
- Train/test split: last 90 days held out as test (810,000 train / 54,000 test rows)
- LightGBM: 500 estimators, learning rate 0.05, 63 leaves, early stopping at 50 rounds
- **Test RMSE: 24.48 | Test MAE: 18.13**
- Generates a recursive 7-day forward forecast per SKU
- Output: `lgbm_forecast_7day.csv`

**02 — Trial 2** (`WH_ORD_Forecast_Trial2.ipynb`)
- Refined version with improved feature engineering and model tuning
- Same WH001 data source, same daily grain
- Run independently after the data generator

### Track 2 output files (generated when notebooks are run — not committed to repo)
```
outputs/warehouse/
├── dim_warehouses.csv          (5 rows — run generator to create)
├── dim_products.csv            (600 rows — run generator to create)
├── dim_calendar.csv            (1,461 rows — run generator to create)
├── fact_orders_header.csv      (~200MB — excluded from repo)
├── fact_orders_detail.csv      (~500MB — excluded from repo)
├── train_dataset.csv           (large — excluded from repo)
├── test_dataset.csv            (large — excluded from repo)
└── lgbm_forecast_7day.csv      (final 7-day SKU forecast)
```

---

## Setup & Installation

### Prerequisites
- Python 3.8+
- Google Colab (recommended) or local Jupyter environment

### Install dependencies

```bash
pip install -r requirements.txt
```

### Update the data path before running

Each notebook has a `DATA_DIR` or `BASE_PATH` variable near the top. Update it to match your environment:

```python
# Google Colab — Track 1
DATA_DIR = "/content/drive/My Drive/Colab Notebooks/Order_Forecasting/Input/shifted_recent"

# Google Colab — Track 2
DATA_DIR = "/content/drive/My Drive/Colab Notebooks/Warehouse_Order_Forecast"

# Local
DATA_DIR = "./data/store_sales_raw"   # Track 1
DATA_DIR = "./data/warehouse_raw"     # Track 2
```

### Running order — Track 1

```
00_Order_Forecast_Data.ipynb        ← optional, only if you want date shifting
         ↓
01_Order_Forecast_step1.ipynb       ← required first, creates all parquet files
         ↓
02_Order_Forecast_step2.ipynb  }
03_Order_Forecast_step3.ipynb  }    ← run both after step 1
         ↓
04_Order_Forecast_step4.ipynb       ← run last, needs outputs from steps 2 and 3
```

### Running order — Track 2

```
00_Warehouse_Ord_ForeCast_Data.ipynb    ← required first, generates all raw data
         ↓
01_WH_ORD_Forecast_Trial1.ipynb    }
02_WH_ORD_Forecast_Trial2.ipynb    }   ← run either or both after generator
```

---

## Requirements

```
pandas
numpy
lightgbm
xgboost
scikit-learn
matplotlib
faker
pyarrow
```

---

## Data Sources

**Track 1:** Kaggle Store Sales — Time Series Forecasting
Download from: https://www.kaggle.com/competitions/store-sales-time-series-forecasting
Place all 6 CSV files in `data/store_sales_raw/`

**Track 2:** Fully synthetic — no external download needed.
Run `notebooks/track_2_warehouse/00_Warehouse_Ord_ForeCast_Data.ipynb` to generate everything.
