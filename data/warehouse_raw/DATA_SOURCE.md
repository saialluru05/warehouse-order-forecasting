## Data Source

This dataset is synthetically generated — no external download required.

Run this notebook to generate all raw files:
notebooks/track_2_warehouse/00_Warehouse_Ord_ForeCast_Data.ipynb

Files it will create in this folder:
- dim_warehouses.csv       (5 warehouses across US regions)
- dim_products.csv         (600 SKUs with ABC classification)
- dim_calendar.csv         (daily calendar 2022–2025)
- fact_orders_header.csv   (~1M order headers)
- fact_orders_detail.csv   (~3M order line items)

Note: fact_orders_header.csv and fact_orders_detail.csv are large files
(500MB+) and are excluded from this repository via .gitignore.
