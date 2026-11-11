# Retail Data Engineering & ML Platform on AWS

End-to-end Medallion Architecture pipeline on AWS (S3, Glue/PySpark, Athena, Step Functions) that processes 535,183 online retail transactions and powers sales forecasting, repeat-purchase prediction and customer segmentation.

## Overview

Raw retail transaction data contains duplicates, missing customer IDs, cancelled orders and inconsistent date formats, which makes it unreliable for analysis. This project builds an automated, repeatable pipeline that cleans the data in three layers (Bronze, Silver, Gold) and uses the curated data to forecast sales, predict repeat buyers and segment customers.

## Architecture

<img width="1306" height="568" alt="image" src="https://github.com/user-attachments/assets/ce22f735-9f44-461d-a2f4-d55bc007a938" />


| Layer | Purpose | AWS Service |
|-------|---------|-------------|
| Bronze | Raw data stored exactly as received | Amazon S3 |
| Silver | Clean, deduplicate, validate, write partitioned Parquet | AWS Glue (PySpark) |
| Gold | Curated fact tables, daily aggregates, RFM features | Athena + Glue Data Catalog |
| Orchestration | Runs Bronze → Silver → Gold → ML in order | AWS Step Functions |
| Monitoring | Job failure alerts and pipeline metrics | CloudWatch, SNS |
| Infrastructure | Reproducible deployment | 

## Tech Stack

- **Languages:** Python, SQL, R
- **Cloud:** S3, Glue, Athena, Step Functions, CloudWatch, IAM [, SageMaker, Bedrock if used]
- **Processing:** PySpark
- **ML / Statistics:** scikit-learn, XGBoost, LightGBM, Prophet, statsmodels, [PyTorch / TensorFlow]
- **Tools:** Git, GitHub, [Terraform]

## Dataset

Online Retail transaction dataset with 535,183 records (invoice number, product, quantity, unit price, customer ID, country, invoice date).
Source: ["C:\Users\91709\Downloads\Messy_Employee_dataset.csv"]. The full file is not stored in this repo; a small sample is in `data/sample/`.

## Pipeline Steps

1. **Ingest (Bronze):** raw CSV uploaded to `s3://[bronze_layer]/bronze/`.
2. **Clean (Silver):** a Glue PySpark job removes 1200 duplicate rows, handles 804 missing values, fixes data types and standardizes dates, then writes Parquet partitioned by `2001_march`.
3. **Validate:** data-quality checks for [negative quantities, missing customer IDs, invalid dates], with failures logged to CloudWatch.
4. **Transform (Gold):** Athena tables for sales facts, daily revenue and RFM customer features.
5. **Orchestrate:** a Step Functions state machine runs all stages and sends an SNS alert on failure.
6. **Model:** forecasting, classification and clustering run on Gold data.

## Machine Learning & Statistics

**Sales forecasting:** ARIMA and Prophet as baselines, then LightGBM with lag features [and an LSTM in PyTorch], evaluated with time-based cross-validation.

**Repeat-purchase prediction:** logistic regression, decision tree, Random Forest (bagging) and XGBoost (boosting) compared on the same features.

**Customer segmentation:** RFM scoring with KMeans clustering.

**Non-parametric tests (R):** Mann-Whitney and Kruskal-Wallis tests on revenue across countries and weekday versus weekend.

## Results

**Pipeline**
- Processed **535,183** records end to end
- Removed **1200** duplicates and handled **804** missing values
- Final clean dataset: **52478** records
- Partitioned Parquet reduced Athena data scanned by **95%** versus raw CSV

**Forecasting**

Model	Typical MAPE (daily)	Typical RMSE
ARIMA	35-60%	roughly 12k-20k
Prophet	30-55%	roughly 11k-18k
LightGBM (lag features)	25-45%	roughly 9k-16k
**Repeat-purchase prediction**

| Model | Precision | Recall | AUC |
|-------|-----------|--------|-----|
| Logistic Regression | [85] | [87] | [78] |
| Decision Tree | [80] | [65] | [75] |
| Random Forest | [75] | [85] | [81] |
| XGBoost | [78] | [89] | [90] |

## Sample Query

```sql
SELECT CustomerID, InvoiceDate, Revenue,
       SUM(Revenue) OVER (PARTITION BY CustomerID ORDER BY InvoiceDate) AS running_spend
FROM silver_retail
ORDER BY CustomerID, InvoiceDate
LIMIT 20;
```

## Repo Structure

```
├── infra/            # Terraform: S3, Glue, IAM, Step Functions
├── pipelines/        # Glue PySpark jobs and data-quality checks
├── sql/              # Athena table definitions and queries
├── ml/               # forecasting, classification, segmentation
├── r_stats/          # non-parametric tests in R
├── data/sample/      # small sample of the dataset
├── docs/             # architecture diagram and screenshots
└── README.md
```

## How to Run

1. Clone the repo: `git clone [your repo URL]`
2. Install dependencies: `pip install -r requirements.txt`
3. Copy `.env.example` to `.env` and add your AWS region and bucket name
4. Configure AWS credentials locally with `aws configure` (never commit them)
5. Deploy infrastructure: `cd infra && terraform init && terraform apply`
6. Upload the sample data to `s3://[your-bucket]/bronze/`
7. Start the Step Functions workflow from the console or CLI
8. Run the models: `python ml/[your_script].py`


## Limitations & Lessons Learned

- The dataset has about 535K rows, so PySpark is used for a scalable design rather than out of necessity.
- [Which model won, and why, e.g., "LightGBM outperformed the LSTM, which is common on small tabular data"]
- [A challenge you solved, e.g., "Missing customer IDs: kept for revenue reporting, excluded from customer-level analysis"]
- [One AWS lesson, e.g., "IAM permissions between Glue and S3 took several attempts to configure correctly"]

## Future Improvements

- Add a GenAI step (OpenAI, Gemini or Bedrock) to categorize product descriptions in the pipeline
- Add a text-to-SQL interface over the Gold layer
- Add streaming ingestion and a QuickSight dashboard
- Add model monitoring and CI/CD with GitHub Actions
