---
title: "Data Pipeline Optimization"
date: 2024-01-01T00:00:00Z
weight: 3
chapter: false
pre: "<b>3. </b>"
---

## 🎯 Task 3 Objectives

Create **S3 bucket** and organize data for MLOps pipeline following the standard **raw → silver → gold → artifacts**, convert CSV → Parquet using **AWS Glue Studio (Visual ETL)** and **measure read/write performance benchmarks**:

- Measure on **AWS CloudShell**.
- Measure on **local machine** (Windows, 16GB RAM).

Focus on:

- **Read/write performance**: CSV vs Parquet.
- **Storage size**: before/after compression.
- **How-to**: step-by-step specifics, reproducible.

{{% notice info %}}
**💡 Task 3 – S3 Storage Optimization**

- ✅ Optimize **format**: Parquet + Snappy instead of plain CSV.
- ✅ Optimize **read/write performance** for ETL & training.
- ✅ Optimize **storage size** (significantly reduce GB).
- ✅ Add **real benchmarks**: CloudShell + local.
  {{% /notice %}}

---
 
📥 **Input from Task 2:** `IAM Roles & Audit` — account ID, IAM roles/policies and CloudTrail/audit setup required to create buckets, Glue roles and permissions.

## 🔧 Real lab environment

- **Account ID:** `842676018087`
- **Lab region:** `us-east-1`
- **Lab region:** `us-east-1`
- **Bucket:** `mlops-retail-prediction-dev-842676018087`

Main dataset:

- `raw/transactions.csv`  
  ≈ **4,593.65 MB**, **33,850,823 rows**
- Example 1 Parquet file after ETL:  
  ≈ **4,593.65 MB**, **33,850,823 rows**
- Example 1 Parquet file after ETL:  
  `silver/shop_week=200607/run-1761638745394-part-block-0-0-r-00000-snappy.parquet`  
  ≈ **458.45 MB**, **33,850,823 rows**
  ≈ **458.45 MB**, **33,850,823 rows**

---

## 1. S3 bucket structure & organization

### 1.1. General storage structure

Apply to all accounts, using `{account-id}` as placeholder:

```text
s3://mlops-retail-prediction-dev-{account-id}/
├── raw/        # original CSV data, immutable
├── silver/     # cleaned/standardized Parquet data
├── gold/       # features, aggregated datasets for training/serving
└── artifacts/  # model, metadata, logs, reports
```

Meaning:

- **raw/**: append only, no edit/delete → serve audit & reprocessing.
- **silver/**: optimized Parquet storage (standard schema, clean).
- **gold/**: final dataset for training/inference.
- **artifacts/**: model.tar.gz, notebook export, log, benchmark CSV, etc.

---

{{% notice tip %}}
**Tip:** Use clear and consistent prefixes (e.g. `raw/`, `silver/`, `gold/`) to easily set up lifecycle rules, IAM policies and S3 analytics. Add `ingest-date=YYYY-MM-DD/` if need to track by upload date.
{{% /notice %}}

### 1.2. Actual structure in lab

With your account ID:

```text
S3 Bucket: mlops-retail-prediction-dev-842676018087
├── raw/
│   └── transactions.csv                # original file ~4.59GB
├── silver/
│   ├── transactions/                   # output from Glue ETL (if not partitioned by week)
│   └── shop_week=200607/
│       └── run-1761638745394-part-block-0-0-r-00000-snappy.parquet  # ~458MB
├── gold/
│   └── (reserved for feature store / aggregated tables)
│   └── (reserved for feature store / aggregated tables)
└── artifacts/
    └── (store benchmark results, model, logs, etc.)
```

You can open S3 Console to confirm correct paths, especially:

- `raw/transactions.csv`
- A typical Parquet file in `silver/shop_week=.../`.

---

## 2. Create bucket & folders on AWS Console

### 2.1. Create S3 Bucket

1. Go to **AWS Console → S3 → Create bucket**.
2. Configuration:

```text
Bucket name: mlops-retail-prediction-dev-842676018087
Region: us-east-1
Block all public access: ✅ Enabled
Versioning: (recommended) Enabled
Versioning: (recommended) Enabled
Default encryption: ✅ SSE-S3
```

<!-- IMAGE PLACEHOLDER: Create-bucket - paste screenshot here -->

![Placeholder - Create bucket](/imagess3-data-storage/placeholder-create-bucket.png)

---

### 2.2. Create 4 main folders

In S3 Console:

1. Open bucket `mlops-retail-prediction-dev-842676018087`.
2. **Create folder** sequentially:

```text
raw/
silver/
gold/
artifacts/
```

<!-- IMAGE PLACEHOLDER: Create-folders - paste screenshot here -->

![Placeholder - Create folders](/imagess3-data-storage/placeholder-folders.png)

---

{{% notice warning %}}
**Warning:** Avoid renaming bucket after deployment — bucket name is global and changes will affect all pipelines, IAM policies, and configs. Always enable `Block all public access` unless there's a specific reason and it's reviewed.
{{% /notice %}}

## 3. Enable Intelligent-Tiering (cost optimization)

Purpose: infrequently accessed data (e.g. old `raw/`, old `artifacts/` logs) automatically moved to cheaper storage tiers, URL unchanged.

Steps:

1. Go to bucket → **Properties** tab.
2. Find **Intelligent-Tiering archive configurations** section → **Edit**.
3. Add configuration:

```text
Configuration name: storage-optimization
Status: Enabled
Scope: Entire bucket (or specific prefix: raw/, silver/, gold/, artifacts/)
```

<!-- IMAGE PLACEHOLDER: Intelligent-tiering - paste screenshot here -->

![Placeholder - Intelligent Tiering](/imagess3-data-storage/placeholder-intelligent-tiering.png)

---

## 4. Convert CSV → Parquet using AWS Glue Studio (Visual ETL)

### 4.1. Upload `transactions.csv` to `raw/`

On S3 Console:

1. Open bucket → folder `raw/`.
2. **Upload → Add files →** select file `transactions.csv` from your machine.
3. **Upload**.

<!-- IMAGE PLACEHOLDER: Upload-csv - paste screenshot here -->

![Placeholder - Upload CSV](/imagess3-data-storage/placeholder-upload.png)

---

### 4.2. Create Glue Job (Visual ETL)

1. Go to **AWS Glue Studio → Jobs → Create job → Visual with a blank canvas**.
2. Set name:

```text
Job name: csv-to-parquet-converter
```

3. Select/create IAM Role with permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": [
        "arn:aws:s3:::mlops-retail-prediction-dev-842676018087/raw/*",
        "arn:aws:s3:::mlops-retail-prediction-dev-842676018087/silver/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:*",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### 4.3. Source node – read CSV from S3

In Glue Studio canvas:

1. Add **S3 Source**.
2. Configuration:

```text
Data source: S3
Format: CSV
S3 URL: s3://mlops-retail-prediction-dev-842676018087/raw/transactions.csv
First row as header: Enabled
Delimiter: ,
```

<!-- IMAGE PLACEHOLDER: Glue Source config - paste screenshot here -->

![Placeholder - Glue Source](/imagess3-data-storage/placeholder-glue-source.png)

Summary:

|     Field | Value                                      |
| --------: | ------------------------------------------ |
| S3 bucket | `mlops-retail-prediction-dev-842676018087` |
|      Path | `raw/transactions.csv`                     |
|    Format | CSV                                        |
|    Header | Yes                                        |
| Delimiter | `,`                                        |

---

### 4.4. Transform – ApplyMapping (schema optimization)

1. Add **ApplyMapping** node.
2. Connect **Source → ApplyMapping**.
3. Data type mapping (example):

| Column      | Source type | Target type   | Notes            |
| ----------- | ----------: | ------------- | ---------------- |
| SHOP_WEEK   |        long | int           | `int32` is enough    |
| SHOP_HOUR   |        long | tinyint       | 0–23             |
| QUANTITY    |        long | smallint      | Quantity         |
| STORE_CODE  |      string | string        | Keep as is       |
| SPEND       |     decimal | decimal(10,2) | Currency, 2 decimals |
| BASKET_TYPE |      string | string        | Categorical      |

<!-- IMAGE PLACEHOLDER: Transform schema - paste screenshot here -->

![Placeholder - Transform schema](/imagess3-data-storage/placeholder-transform.png)

**Benefits:**

- Reduce Parquet file size.
- Optimize scan & aggregation.
- Reduce RAM when reading data.

---

### 4.5. Target – write Parquet (Snappy) to `silver/`
### 4.5. Target – write Parquet (Snappy) to `silver/`

1. Add **S3 Target** node.
2. Connect **ApplyMapping → Target**.
3. Configuration:

```text
Data target: S3
Format: Parquet
Compression: Snappy
S3 path: s3://mlops-retail-prediction-dev-842676018087/silver/transactions/
Partition keys: SHOP_WEEK (recommended)
Partition keys: SHOP_WEEK (recommended)
```

_Illustration:_

-- Target config: `/imagess3-data-storage/target-config.png`
-- Full pipeline: `/imagess3-data-storage/04-glue-etl.png`

<!-- IMAGE PLACEHOLDER: Glue Target / Pipeline - paste screenshot here -->

![Placeholder - Glue Target](/imagess3-data-storage/placeholder-glue-target.png)

4. **Save & Run job** → monitor **Job run details** → check output in `silver/`.

![Placeholder - Glue Target](/imagess3-data-storage/04-glue-etl.png)

![Placeholder - Glue Target](/imagess3-data-storage/result-in-silver.png)

---

{{% notice info %}}
**Info:** When partitioning, balance between number of partitions and file size — too many small files will slow down queries; consider running compaction step (Glue/Athena/EMR) to merge into larger files (e.g., 128–512 MB each) if needed.
{{% /notice %}}

{{% notice tip %}}
**Tip:** Choose Snappy for balance between compression ratio and CPU usage. If need to save more I/O, try `ZSTD` (if your stack supports) for better ratio but check CPU cost.
{{% /notice %}}

## 5. Real benchmark on AWS CloudShell (read directly from S3)

### 5.1. Dataset info & how to run

- Run on **AWS CloudShell**.
- Read directly:
- Run on **AWS CloudShell**.
- Read directly:

  - `raw/transactions.csv` (~4,593.65 MB, 33,850,823 rows).
  - 1 Parquet file (~458.45 MB, 33,850,823 rows).
  - 1 Parquet file (~458.45 MB, 33,850,823 rows).

You used scripts like:

- `read_csv_s3(...)` to measure CSV reading.
- `read_parquet_s3(...)` to measure Parquet reading.

Detailed logs appeared in CloudShell.

<!-- IMAGE PLACEHOLDER: CloudShell benchmark - paste screenshot here -->

{{% notice info %}}
**Info:** CloudShell has resource limits (vCPU/RAM/IO) — measured timings may differ from EC2/Glue worker sizes. When comparing, specify instance/worker size to reproduce accurate results.
{{% /notice %}}

CSV read results:
![Placeholder - CloudShell benchmark](/imagess3-data-storage/placeholder-cloudshell.png)

Parquet read results:
![Placeholder - CloudShell benchmark](/imagess3-data-storage/placeholder-cloudshell-parquet.png)

---

### 5.2. Measurement results (CloudShell)

**CSV – read entire `raw/transactions.csv` from S3**

5 measurements:

```text
151.91s, 146.34s, 141.52s, 126.03s, 115.95s
```

Average calculation (approximate):

- Avg time ≈ **136.35 s**
- Size = **4,593.65 MB**
- Avg throughput ≈ **33.7 MB/s**
- Rows/s ≈ **~248k rows/s**

---

**Parquet – read 1 file ~458.45 MB from S3**

5 measurements:

```text
61.37s, 53.65s, 52.51s, 49.66s, 49.55s
```

Average calculation (approximate):

- Avg time ≈ **53.35 s**
- Size = **458.45 MB**
- Avg throughput ≈ **8.6 MB/s**
- Rows/s ≈ **~635k rows/s**

---

### 5.3. Comparison table (CloudShell)

|    Type | Size on S3 | Avg time (s) | Avg throughput (MB/s) |       Rows | Rows/s (approx) | Relative rows/s |
| ------: | ----------: | -----------: | --------------------: | ---------: | --------------: | --------------: |
|     CSV | 4,593.65 MB |       136.35 |                  33.7 | 33,850,823 |           ~248k |              1× |
| Parquet |   458.45 MB |        53.35 |                   8.6 | 33,850,823 |           ~635k |           ~2.6× |

**Explanation:**
**Explanation:**

- By **MB/s**, CSV seems "faster" because each run processes more MB (4.59 GB).
- But in terms of **rows per second (rows/s)**, Parquet is **~2.6× faster**, suitable for ETL / training.

{{% notice info %}}
**CloudShell Conclusion**

- Parquet (Snappy) **dramatically reduces storage size**: 4.59 GB → ~0.46 GB.
- With same 33.85M rows, Parquet processes **~2.6× faster** in rows/s.
  {{% /notice %}}

---

## 6. Benchmark on local machine

### 6.1. Prepare directory & download data

On Windows:

```bash
mkdir s3-local-benchmark
cd s3-local-benchmark
```

Download 2 files:

```bash
aws s3 cp s3://mlops-retail-prediction-dev-842676018087/raw/transactions.csv ./transactions.csv
aws s3 cp s3://mlops-retail-prediction-dev-842676018087/silver/shop_week=200607/run-1761638745394-part-block-0-0-r-00000-snappy.parquet ./transactions_200607.parquet
```

<!-- IMAGE PLACEHOLDER: Local download - paste screenshot here -->

![Placeholder - Local download](/imagess3-data-storage/placeholder-local-download.png)

### 6.2. Benchmark script

Create file `local_benchmark.py`:

```python
import time
import os
import pandas as pd

def bench_csv_stream(path: str, runs: int = 3):
    print(f"=== Benchmark CSV (streaming): {path} ===")
    size_mb = os.path.getsize(path) / (1024 * 1024)
    for i in range(1, runs + 1):
        t0 = time.time()
        rows = 0
        # Read in chunks to avoid RAM overflow
        for chunk in pd.read_csv(path, chunksize=500_000):
            rows += len(chunk)
        t1 = time.time()
        elapsed = t1 - t0
        throughput = size_mb / elapsed
        print(f"[local_csv_stream] run={i} time={elapsed:.2f}s, "
              f"size={size_mb:.2f} MB, throughput={throughput:.2f} MB/s, rows={rows}")

def bench_parquet_stream(path: str, runs: int = 3):
    print(f"=== Benchmark Parquet (streaming): {path} ===")
    size_mb = os.path.getsize(path) / (1024 * 1024)
    for i in range(1, runs + 1):
        t0 = time.time()
        df = pd.read_parquet(path)
        rows = len(df)
        t1 = time.time()
        elapsed = t1 - t0
        throughput = size_mb / elapsed
        print(f"[local_parquet_stream] run={i} time={elapsed:.2f}s, "
              f"size={size_mb:.2f} MB, throughput={throughput:.2f} MB/s, rows={rows}")

if __name__ == "__main__":
    bench_csv_stream("transactions.csv", runs=3)
    bench_parquet_stream("transactions_200607.parquet", runs=3)
```

Run:

```bash
python local_benchmark.py
```

---

### 6.3. Actual logs

![Placeholder - Local download](/imagess3-data-storage/placeholder-result-readfile.png)

**Observations:**

- CSV full 4.59 GB: still processable thanks to chunk reading, throughput ~90–95 MB/s.
- Parquet (sample 1 week, 6.48 MB): read time ~0.05–0.09s → extremely low latency.
- With many small Parquet files (partitioned by `shop_week`), querying by week/month will be very fast.

---

{{% notice warning %}}
**Warning (Local):** On local machine, reading entire Parquet dataset into memory (`pd.read_parquet`) can cause RAM shortage. Use reading by pieces (pyarrow dataset, iter_batches) or increase `chunksize` when using CSV to avoid OOM.
{{% /notice %}}

{{% notice tip %}}
**Tip (Local):** When benchmarking on Windows, close other heavy programs and IO results may differ between HDD and SSD — note drive type (SSD/HDD) in benchmark report.
{{% /notice %}}

## 7. IAM – Minimum permissions for Glue Job (summary)

Minimum required:

- **S3:**
  - `s3:GetObject` for `raw/*`
  - `s3:PutObject` for `silver/*`
  - `s3:GetObject` for `raw/*`
  - `s3:PutObject` for `silver/*`
- **Glue:**
  - Permission to create/run job, read metadata (depends on environment).
- **CloudWatch Logs:** write job logs.

Example policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": [
        "arn:aws:s3:::mlops-retail-prediction-dev-842676018087/raw/*",
        "arn:aws:s3:::mlops-retail-prediction-dev-842676018087/silver/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "glue:*",
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```
---

## 8. Task 3 Summary – S3 Data Storage

**About architecture:**

- Design bucket following MLOps standard:

  ```text
  raw/ → silver/ → gold/ → artifacts/
  ```

- Maintain **raw/ immutable**.
- Standardize data to **Parquet (Snappy)** in **silver/**.

**About performance (from your actual measurements):**

- 4.59 GB CSV → ~0.46 GB Parquet for same **33.85M rows**.
- On CloudShell:

  - CSV: ~136s, ~248k rows/s.
  - Parquet: ~53s, ~635k rows/s → **~2.6× rows/s**.

- On local machine (16GB RAM):
- On local machine (16GB RAM):

  - CSV 4.59 GB still processable with **500k rows chunk**.
  - Parquet sample 1 week (~6.48 MB) read in **~0.05–0.09s**.

**About cost & operations:**

- Parquet + Snappy **significantly reduces size** → reduce S3 costs.
- Intelligent-Tiering helps automatically downgrade storage tier for old data.
- Glue Visual ETL helps minimize coding, easy to show in reports.

## 9. Clean Up Resources (AWS CLI)

### 9.1. Delete all objects in S3 bucket

```bash
# Delete all files in bucket
aws s3 rm s3://mlops-retail-prediction-dev-842676018087 --recursive

# Check bucket is empty
aws s3 ls s3://mlops-retail-prediction-dev-842676018087 --recursive
```

### 9.2. Delete S3 bucket

```bash
# Delete bucket (only when empty)
aws s3 rb s3://mlops-retail-prediction-dev-842676018087

# Check bucket is deleted
aws s3 ls | grep mlops-retail-prediction-dev
```

### 9.3. Delete Glue Job

```bash
# List Glue jobs
# List Glue jobs
aws glue get-jobs --query 'Jobs[?contains(Name, `csv-to-parquet`)].Name'

# Delete Glue job
aws glue delete-job --job-name csv-to-parquet-converter

# Check job is deleted
aws glue get-job --job-name csv-to-parquet-converter
```

### 9.4. Delete IAM Role (if created specifically for Glue)

```bash
# Detach policies from role
aws iam detach-role-policy --role-name GlueETLRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Delete inline policies (if any)
# Delete inline policies (if any)
aws iam delete-role-policy --role-name GlueETLRole --policy-name S3AccessPolicy

# Delete role
aws iam delete-role --role-name GlueETLRole
```

{{% notice success %}}
**Success tip:** If you delete resources to avoid costs, check CloudWatch log groups and Athena query history — some logs or query history may still store metadata and cause small costs if not cleaned up.
{{% /notice %}}

---

## 10. S3 Storage Pricing Table (ap-southeast-1)

### 10.1. Storage cost by class

| Storage Class | Price (USD/GB/month) | Minimum Duration | Notes |
|---------------|---------------------|------------------|---------|
| **S3 Standard** | $0.025 | None | Frequent access |
| **S3 Standard-IA** | $0.0138 | 30 days | Infrequent access |
| **S3 One Zone-IA** | $0.011 | 30 days | Single AZ |
| **S3 Glacier Instant** | $0.005 | 90 days | Archive, instant retrieval |
| **S3 Glacier Flexible** | $0.0045 | 90 days | Archive, 1-12 hours retrieval |
| **S3 Deep Archive** | $0.002 | 180 days | Long-term archive, 12+ hours |

### 10.2. Request costs

| Request Type | Price (USD/1000 requests) | Notes |
|--------------|--------------------------|---------|
| **PUT/POST/LIST** | $0.0055 | Write operations |
| **GET/SELECT** | $0.00044 | Read operations |
| **Data Transfer OUT** | $0.12/GB | First 1GB free/month |

### 10.3. Project cost estimate

**Current data:**
- Raw CSV: 4.59 GB
- Silver Parquet: 0.46 GB  
- **Total:** ~5 GB

**Monthly cost (S3 Standard):**
**Monthly cost (S3 Standard):**

| Component | Size | Price/GB | Monthly Cost |
|-----------|------|----------|--------------|
| Raw data (CSV) | 4.59 GB | $0.025 | $0.11 |
| Silver data (Parquet) | 0.46 GB | $0.025 | $0.01 |
| Gold + artifacts | ~0.5 GB | $0.025 | $0.01 |
| **Total Storage** | **~5.5 GB** | | **$0.14** |
| Requests (estimated) | ~1000 req | $0.0055 | $0.006 |
| **Grand Total** | | | **≈ $0.15/month** |

**With Intelligent Tiering:**
- After 30 days: Raw data moves to Standard-IA → save ~45%
- After 90 days: Old artifacts move to Glacier → save ~80%
- **Estimated savings:** ~$0.05-0.08/month

{{% notice info %}}
**💰 Optimized Storage Cost**
- **Current:** ~$0.15/month for 5.5GB
- **With Intelligent Tiering:** ~$0.07-0.10/month  
- **Parquet format:** Reduces 90% storage size vs CSV
{{% /notice %}}

---

{{% notice success %}}
**🎯 Task 3 Completed**

- Clear S3 architecture, MLOps standard.
- CSV → Parquet using Glue Studio (Visual, with illustrations).
- Real benchmarks on **CloudShell** and **local**, with specific numbers.
- Detailed **clean up commands** and **pricing breakdown**.
- Easy to present in reports & demo for professors.
  {{% /notice %}}

---

## 📹 Task 3 Implementation Video

<div style="position: relative; width: 100%; max-width: 2000px; margin: 0 auto; padding-bottom: 56.25%; height: 0; overflow: hidden;">
  <iframe 
    style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;" 
    src="https://www.youtube.com/embed/watch?v=CzxyXken33Y&list=PL53MEKrSAUpu0i5F-ttcVdKkSv0jb48Mc&index=2" 
    title="YouTube video player" 
    frameborder="0" 
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" 
    referrerpolicy="strict-origin-when-cross-origin" 
    allowfullscreen>
  </iframe>
</div>

---

**Next Step**: [Task 4: SageMaker Training](../4-sagemaker-training/)


