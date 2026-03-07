# NYC Yellow Taxi Big Data Analytics (Databricks + Spark)

End-to-end big data analytics project on NYC Yellow Taxi trips (Jan 1-14, 2023), implemented in Databricks Free Tier with Apache Spark.

The project covers:
- Data ingestion and schema standardization (Bronze layer)
- Data cleaning and feature engineering (Silver layer)
- Exploratory analysis and visual insights (Gold/EDA)
- Supervised predictive modeling (tip classification)
- Model comparison, threshold tuning, and error analysis
- Optional supporting fare regression analysis

## Project Objective
This project analyzes spatial, temporal, and behavioral patterns in NYC taxi trips, with a main predictive task:
- Predict whether a trip results in a recorded tip (`tipped` = 1/0), using credit-card trips only (where tips are observable).

Core analytical themes:
- How fare and tipping patterns vary by pickup/dropoff zone
- How fare and tipping patterns vary across time buckets (`Night`, `Day`, `Evening`)
- Which features most influence tip prediction

## Dataset
- Source: NYC TLC Yellow Taxi Trip Data
- Period used: January 1-14, 2023
- Scale: ~1.2M rows (subset)
- Main fields: timestamps, trip distance, fares/surcharges, tip amount, payment type, pickup/dropoff location IDs

Additional lookup data:
- `data/taxi_zone_lookup.csv` for mapping `LocationID` to borough/zone names
- `data/NYC_Taxi_Zones_20260206.geojson` for spatial visualizations

## Tech Stack
- Databricks Free Tier
- Apache Spark (PySpark, Spark SQL, Spark MLlib)
- Delta tables in Unity Catalog (`workspace.bda_taxi.*`)
- Matplotlib / Plotly (EDA visualization)

## Pipeline Overview
The notebooks implement a medallion-style flow:

1. Bronze (Ingestion)
- Read source table
- Apply defensive type casting to a canonical schema
- Persist to Delta table: `workspace.bda_taxi.taxi_bronze`

2. Silver (Cleaning + Features)
- Remove invalid records and enforce basic quality checks
- Handle null numeric charges
- Add temporal features (`pickup_hour`, `pickup_dow`, `time_bucket`)
- Add tip-label features:
  - `tip_recorded` (credit card trip)
  - `tipped` (credit card and `tip_amount > 0`)
- Enrich with zone names
- Persist to Delta table: `workspace.bda_taxi.taxi_silver`

3. Gold / EDA
- Distribution checks and quantiles for key numeric variables
- Temporal summaries of fares and tip rates
- Spatial top-k analyses by pickup/dropoff zones
- Choropleth mapping with GeoJSON, good way to visualise the density of data by location

4. Predictive Modeling (Main)
- Binary classification task: predict `tipped`
- Credit-card-only modeling subset (`tip_recorded == 1`)
- Models:
  - Logistic Regression (baseline)
  - Random Forest Classifier (non-linear)
- Feature prep via `StringIndexer + OneHotEncoder + VectorAssembler`
- Evaluation metrics include AUC-ROC, accuracy, F1, and confusion matrix

5. Model Comparison + Threshold/Operational Analysis
- Compare model metrics and ROC behavior
- Save best model decision table
- Threshold analysis on stored predictions
- Persist threshold metrics for downstream analysis

6. Final Outputs and Error Analysis
- Choose operating threshold
- Calibration bins (predicted vs actual tip rates)
- Error slices by time and geography
- Persist report-ready output tables

7. Optional Supporting Analysis
- Fare regression:
  - Linear Regression (baseline)
  - Decision Tree Regressor
- Metrics: RMSE, R2, MAE

## Repository Structure
- `notebooks/01_ingest_bronze.ipynb`
- `notebooks/02_clean_silver.ipynb`
- `notebooks/03_exploratory_data_analysis_gold.ipynb`
- `notebooks/04_model_training_and_evaluation_shared.ipynb`
- `notebooks/04_model_training_and_evaluation_logistic_regression.ipynb`
- `notebooks/04_model_training_and_evaluation_random_forest.ipynb`
- `notebooks/04_model_training_and_evaluation_comparision.ipynb`
- `notebooks/05_model_interpretation_error_analysis_and_final_outputs.ipynb`
- `notebooks/05_model_interpretation_error_analysis_and_final_outputs_interpretation_only.ipynb`
- `notebooks/05_model_interpretation_error_analysis_and_final_outputs_fare_regression.ipynb`
- `data/` (lookup + geojson assets)
- `requirements/` (assignment brief, proposal text, references)

## How To Reproduce (Databricks Free Tier)
Run notebooks in this order.

1. Data setup
- Ensure your raw taxi data is available as a Databricks table.
- Default source table expected in notebook widgets:
  - `workspace.default.2023_yellow_taxi_trip_data_jan_1st_to_jan_14th_20260108`
- Ensure taxi zone lookup table exists:
  - `workspace.bda_taxi.taxi_zone_lookup`
  - You can upload `data/taxi_zone_lookup.csv` and register it as that table.

2. Notebook execution order
1. `notebooks/01_ingest_bronze.ipynb`
2. `notebooks/02_clean_silver.ipynb`
3. `notebooks/03_exploratory_data_analysis_gold.ipynb`
4. `notebooks/04_model_training_and_evaluation_logistic_regression.ipynb`
5. `notebooks/04_model_training_and_evaluation_random_forest.ipynb`
6. `notebooks/04_model_training_and_evaluation_comparision.ipynb`
7. `notebooks/05_model_interpretation_error_analysis_and_final_outputs.ipynb`

Optional:
8. `notebooks/05_model_interpretation_error_analysis_and_final_outputs_interpretation_only.ipynb`
9. `notebooks/05_model_interpretation_error_analysis_and_final_outputs_fare_regression.ipynb`

Notes:
- Notebooks use Databricks widgets for table names and key parameters.
- Defaults are prefilled for this project and can be overridden at runtime.
- Some plotting cells use `.toPandas()`; for Free Tier limits, keep sampling enabled where applicable.

## Key Output Tables
Core pipeline:
- `workspace.bda_taxi.taxi_bronze`
- `workspace.bda_taxi.taxi_silver`

Classification outputs:
- `workspace.bda_taxi.model_train_set`
- `workspace.bda_taxi.model_test_set`
- `workspace.bda_taxi.model_preds_logreg`
- `workspace.bda_taxi.model_preds_rf`
- `workspace.bda_taxi.model_metrics_comparison`
- `workspace.bda_taxi.model_comparison_final`
- `workspace.bda_taxi.model_threshold_metrics`
- `workspace.bda_taxi.model_preds_best`

Final analysis outputs:
- `workspace.bda_taxi.best_model_scored`
- `workspace.bda_taxi.best_model_calibration_bins`
- `workspace.bda_taxi.best_model_threshold_choice`
- `workspace.bda_taxi.best_model_error_by_time_bucket`
- `workspace.bda_taxi.best_model_error_by_pickup_hour`
- `workspace.bda_taxi.best_model_error_by_pu_zone`
- `workspace.bda_taxi.best_model_error_by_do_zone`
- `workspace.bda_taxi.best_model_fpfn_by_pu_zone`
- `workspace.bda_taxi.best_model_fpfn_by_do_zone`

Optional regression outputs:
- `workspace.bda_taxi.fare_regression_metrics`
- `workspace.bda_taxi.fare_regression_predictions_lr`
- `workspace.bda_taxi.fare_regression_predictions_dt`

## Reproducibility Notes
- Train/test split uses fixed seed (`42`) and persisted split tables.
- Classification modeling excludes cash trips from the target task to avoid unobserved tip labels.
- Feature engineering and model logic are centralized in `notebooks/04_model_training_and_evaluation_shared.ipynb`.

## Limitations
- Dataset covers a 2-week window only (not full-year seasonality).
- Tip labels are only observable for card transactions.
- Databricks Free Tier resource limits may require sampling for heavy visualizations.
