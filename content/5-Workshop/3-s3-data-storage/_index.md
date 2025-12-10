---
title: "Data Pipeline Optimization"
date: 2024-01-01T00:00:00Z
weight: 3
chapter: false
pre: "<b>3. </b>"
---

## 🎯 Task 3 Objectives

<<<<<<< HEAD
Create **S3 bucket** and organize data for MLOps pipeline following the standard **raw → silver → gold → artifacts**, convert CSV → Parquet using **AWS Glue Studio (Visual ETL)** and **measure read/write performance benchmarks**:

- Measure on **AWS CloudShell**.
- Measure on **local machine** (Windows, 16GB RAM).
=======
Create an **S3 bucket** and organize data for the MLOps pipeline following the standard **raw → silver → gold → artifacts**, convert CSV → Parquet using **AWS Glue Studio (Visual ETL)**, and **benchmark read/write performance**:

- Benchmark on **AWS CloudShell**.
- Benchmark on **local machine** (Windows, 16GB RAM).
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

Focus on:

- **Read/write performance**: CSV vs Parquet.
<<<<<<< HEAD
- **Storage size**: before/after compression.
- **How-to**: step-by-step specifics, reproducible.
=======
- **Storage footprint**: before/after compression.
- **How-to**: detailed, reproducible steps.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

{{% notice info %}}
**💡 Task 3 – S3 Storage Optimization**

<<<<<<< HEAD
- ✅ Optimize **format**: Parquet + Snappy instead of plain CSV.
- ✅ Optimize **read/write performance** for ETL & training.
- ✅ Optimize **storage size** (significantly reduce GB).
=======
- ✅ Optimize **format**: Parquet + Snappy instead of raw CSV.
- ✅ Optimize **read/write performance** for ETL & training.
- ✅ Optimize **storage size** (significant GB reduction).
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- ✅ Add **real benchmarks**: CloudShell + local.
  {{% /notice %}}

---
<<<<<<< HEAD
 
📥 **Input from Task 2:** `IAM Roles & Audit` — account ID, IAM roles/policies and CloudTrail/audit setup required to create buckets, Glue roles and permissions.

## 🔧 Real lab environment
=======

📥 **Input from Task 2:** `IAM Roles & Audit` — account ID, IAM roles/policies and CloudTrail/audit setup required to create buckets, Glue roles and permissions.

## 🔧 Real Lab Environment
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

- **Account ID:** `842676018087`
- **Lab region:** `us-east-1`
- **Bucket:** `mlops-retail-prediction-dev-842676018087`

Main dataset:

- `raw/transactions.csv`  
  ≈ **4,593.65 MB**, **33,850,823 rows**
- Example 1 Parquet file after ETL:  
  `silver/shop_week=200607/run-1761638745394-part-block-0-0-r-00000-snappy.parquet`  
  ≈ **458.45 MB**, **33,850,823 rows**

---

<<<<<<< HEAD
## 1. S3 bucket structure & organization

### 1.1. General storage structure

Apply to all accounts, using `{account-id}` as placeholder:

```text
s3://mlops-retail-prediction-dev-{account-id}/
├── raw/        # original CSV data, immutable
├── silver/     # cleaned/standardized Parquet data
├── gold/       # features, aggregated datasets for training/serving
=======
## 1. S3 Bucket Structure & Organization

### 1.1. High-level storage layout

Apply this for any account, using `{account-id}` as a placeholder:

```text
s3://mlops-retail-prediction-dev-{account-id}/
├── raw/        # immutable original CSV data
├── silver/     # cleaned/standardized Parquet data
├── gold/       # features & aggregated datasets for training/serving
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
└── artifacts/  # model, metadata, logs, reports
```

Meaning:

<<<<<<< HEAD
- **raw/**: append only, no edit/delete → serve audit & reprocessing.
- **silver/**: optimized Parquet storage (standard schema, clean).
- **gold/**: final dataset for training/inference.
- **artifacts/**: model.tar.gz, notebook export, log, benchmark CSV, etc.
=======
- **raw/**: append-only, no edits/deletes → supports audit & reprocessing.
- **silver/**: optimized Parquet (clean schema, standardized).
- **gold/**: final datasets for training/inference.
- **artifacts/**: model.tar.gz, notebook exports, logs, benchmark CSV, …
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

{{% notice tip %}}
<<<<<<< HEAD
**Tip:** Use clear and consistent prefixes (e.g. `raw/`, `silver/`, `gold/`) to easily set up lifecycle rules, IAM policies and S3 analytics. Add `ingest-date=YYYY-MM-DD/` if need to track by upload date.
{{% /notice %}}

### 1.2. Actual structure in lab

With your account ID:
=======
**Tip:** Clear and consistent prefixes (e.g., `raw/`, `silver/`, `gold/`) make it easier to set lifecycle rules, IAM policies, and S3 analytics. Add `ingest-date=YYYY-MM-DD/` if you need upload-date tracking.
{{% /notice %}}

### 1.2. Actual lab structure

For your account ID:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
S3 Bucket: mlops-retail-prediction-dev-842676018087
├── raw/
<<<<<<< HEAD
│   └── transactions.csv                # original file ~4.59GB
├── silver/
│   ├── transactions/                   # output from Glue ETL (if not partitioned by week)
=======
│   └── transactions.csv                # source file ~4.59GB
├── silver/
│   ├── transactions/                   # Glue ETL output (if not partitioned by week)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
│   └── shop_week=200607/
│       └── run-1761638745394-part-block-0-0-r-00000-snappy.parquet  # ~458MB
├── gold/
│   └── (reserved for feature store / aggregated tables)
└── artifacts/
<<<<<<< HEAD
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
=======
    └── (store benchmark results, model, logs, …)
```

Open the S3 Console and verify paths, especially:

- `raw/transactions.csv`
- A representative Parquet file under `silver/shop_week=.../`.

---

## 2. Create bucket & folders in AWS Console

### 2.1. Create an S3 Bucket

1. Go to **AWS Console → S3 → Create bucket**.
2. Configure:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
Bucket name: mlops-retail-prediction-dev-842676018087
Region: us-east-1
Block all public access: ✅ Enabled
Versioning: (recommended) Enabled
Default encryption: ✅ SSE-S3
```

<!-- IMAGE PLACEHOLDER: Create-bucket - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Create bucket](/imagess3-data-storage/placeholder-create-bucket.png)

---

### 2.2. Create 4 main folders
=======
![Placeholder - Create bucket](/images/s3-data-storage/placeholder-create-bucket.png)

---

### 2.2. Create the 4 main folders
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

In S3 Console:

1. Open bucket `mlops-retail-prediction-dev-842676018087`.
<<<<<<< HEAD
2. **Create folder** sequentially:
=======
2. **Create folder** in order:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
raw/
silver/
gold/
artifacts/
```

<!-- IMAGE PLACEHOLDER: Create-folders - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Create folders](/imagess3-data-storage/placeholder-folders.png)
=======
![Placeholder - Create folders](/images/s3-data-storage/placeholder-folders.png)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

{{% notice warning %}}
<<<<<<< HEAD
**Warning:** Avoid renaming bucket after deployment — bucket name is global and changes will affect all pipelines, IAM policies, and configs. Always enable `Block all public access` unless there's a specific reason and it's reviewed.
=======
**Warning:** Avoid renaming the bucket after deployment — bucket names are global and changes will break pipelines, IAM policies, and configs. Keep `Block all public access` enabled unless you have a specific, reviewed reason.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
{{% /notice %}}

## 3. Enable Intelligent-Tiering (cost optimization)

<<<<<<< HEAD
Purpose: infrequently accessed data (e.g. old `raw/`, old `artifacts/` logs) automatically moved to cheaper storage tiers, URL unchanged.

Steps:

1. Go to bucket → **Properties** tab.
2. Find **Intelligent-Tiering archive configurations** section → **Edit**.
3. Add configuration:
=======
Goal: cold data (e.g., old `raw/`, old `artifacts/` logs) moves automatically to cheaper storage classes without changing URLs.

Steps:

1. Open the bucket → **Properties** tab.
2. Find **Intelligent-Tiering archive configurations** → **Edit**.
3. Add a configuration:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
Configuration name: storage-optimization
Status: Enabled
<<<<<<< HEAD
Scope: Entire bucket (or specific prefix: raw/, silver/, gold/, artifacts/)
=======
Scope: Entire bucket (or specific prefixes: raw/, silver/, gold/, artifacts/)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
```

<!-- IMAGE PLACEHOLDER: Intelligent-tiering - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Intelligent Tiering](/imagess3-data-storage/placeholder-intelligent-tiering.png)
=======
![Placeholder - Intelligent Tiering](/images/s3-data-storage/placeholder-intelligent-tiering.png)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

## 4. Convert CSV → Parquet using AWS Glue Studio (Visual ETL)

### 4.1. Upload `transactions.csv` to `raw/`

<<<<<<< HEAD
On S3 Console:

1. Open bucket → folder `raw/`.
2. **Upload → Add files →** select file `transactions.csv` from your machine.
=======
In S3 Console:

1. Open the bucket → folder `raw/`.
2. **Upload → Add files →** select `transactions.csv` from your machine.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
3. **Upload**.

<!-- IMAGE PLACEHOLDER: Upload-csv - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Upload CSV](/imagess3-data-storage/placeholder-upload.png)

---

### 4.2. Create Glue Job (Visual ETL)

1. Go to **AWS Glue Studio → Jobs → Create job → Visual with a blank canvas**.
2. Set name:
=======
![Placeholder - Upload CSV](/images/s3-data-storage/placeholder-upload.png)

---

### 4.2. Create a Glue Job (Visual ETL)

1. Go to **AWS Glue Studio → Jobs → Create job → Visual with a blank canvas**.
2. Set the job name:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
Job name: csv-to-parquet-converter
```

<<<<<<< HEAD
3. Select/create IAM Role with permissions:
=======
3. Select/create an IAM Role with permissions:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

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

<<<<<<< HEAD
In Glue Studio canvas:

1. Add **S3 Source**.
2. Configuration:
=======
In the Glue Studio canvas:

1. Add **S3 Source**.
2. Configure:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
Data source: S3
Format: CSV
S3 URL: s3://mlops-retail-prediction-dev-842676018087/raw/transactions.csv
First row as header: Enabled
Delimiter: ,
```

<!-- IMAGE PLACEHOLDER: Glue Source config - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Glue Source](/imagess3-data-storage/placeholder-glue-source.png)
=======
![Placeholder - Glue Source](/images/s3-data-storage/placeholder-glue-source.png)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

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
<<<<<<< HEAD
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
=======
3. Example data type mapping:

| Column      | Source type | Target type   | Notes                |
| ----------- | ----------: | ------------- | -------------------- |
| SHOP_WEEK   |        long | int           | `int32` is enough    |
| SHOP_HOUR   |        long | tinyint       | 0–23                 |
| QUANTITY    |        long | smallint      | quantity             |
| STORE_CODE  |      string | string        | keep as-is           |
| SPEND       |     decimal | decimal(10,2) | currency, 2 decimals |
| BASKET_TYPE |      string | string        | categorical          |

<!-- IMAGE PLACEHOLDER: Transform schema - paste screenshot here -->

![Placeholder - Transform schema](/images/s3-data-storage/placeholder-transform.png)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

**Benefits:**

- Reduce Parquet file size.
<<<<<<< HEAD
- Optimize scan & aggregation.
- Reduce RAM when reading data.
=======
- Optimize scans & aggregations.
- Reduce RAM usage when reading.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

### 4.5. Target – write Parquet (Snappy) to `silver/`

1. Add **S3 Target** node.
2. Connect **ApplyMapping → Target**.
<<<<<<< HEAD
3. Configuration:
=======
3. Configure:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
Data target: S3
Format: Parquet
Compression: Snappy
S3 path: s3://mlops-retail-prediction-dev-842676018087/silver/transactions/
Partition keys: SHOP_WEEK (recommended)
```

_Illustration:_

<<<<<<< HEAD
-- Target config: `/imagess3-data-storage/target-config.png`
-- Full pipeline: `/imagess3-data-storage/04-glue-etl.png`

<!-- IMAGE PLACEHOLDER: Glue Target / Pipeline - paste screenshot here -->

![Placeholder - Glue Target](/imagess3-data-storage/placeholder-glue-target.png)

4. **Save & Run job** → monitor **Job run details** → check output in `silver/`.

![Placeholder - Glue Target](/imagess3-data-storage/04-glue-etl.png)

![Placeholder - Glue Target](/imagess3-data-storage/result-in-silver.png)
=======
-- Target config: `/images/s3-data-storage/target-config.png`
-- Full pipeline: `/images/s3-data-storage/04-glue-etl.png`

<!-- IMAGE PLACEHOLDER: Glue Target / Pipeline - paste screenshot here -->

![Placeholder - Glue Target](/images/s3-data-storage/placeholder-glue-target.png)

4. **Save & Run job** → monitor **Job run details** → verify output in `silver/`.

![Placeholder - Glue Target](/images/s3-data-storage/04-glue-etl.png)

![Placeholder - Glue Target](/images/s3-data-storage/result-in-silver.png)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

{{% notice info %}}
<<<<<<< HEAD
**Info:** When partitioning, balance between number of partitions and file size — too many small files will slow down queries; consider running compaction step (Glue/Athena/EMR) to merge into larger files (e.g., 128–512 MB each) if needed.
{{% /notice %}}

{{% notice tip %}}
**Tip:** Choose Snappy for balance between compression ratio and CPU usage. If need to save more I/O, try `ZSTD` (if your stack supports) for better ratio but check CPU cost.
{{% /notice %}}

## 5. Real benchmark on AWS CloudShell (read directly from S3)
=======
**Info:** With partitioning, balance partition count vs file size — too many small files will slow queries. Consider a compaction step (Glue/Athena/EMR) to merge into larger files (e.g., 128–512 MB per file) if needed.
{{% /notice %}}

{{% notice tip %}}
**Tip:** Snappy is a good balance between compression ratio and CPU usage. If you need more I/O savings, try `ZSTD` (if your stack supports it) for better compression, but benchmark CPU cost.
{{% /notice %}}

## 5. Real Benchmark on AWS CloudShell (read directly from S3)
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

### 5.1. Dataset info & how to run

- Run on **AWS CloudShell**.
- Read directly:

  - `raw/transactions.csv` (~4,593.65 MB, 33,850,823 rows).
  - 1 Parquet file (~458.45 MB, 33,850,823 rows).

You used scripts like:

<<<<<<< HEAD
- `read_csv_s3(...)` to measure CSV reading.
- `read_parquet_s3(...)` to measure Parquet reading.

Detailed logs appeared in CloudShell.
=======
- `read_csv_s3(...)` to benchmark CSV reads.
- `read_parquet_s3(...)` to benchmark Parquet reads.

Detailed logs were shown in CloudShell.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

<!-- IMAGE PLACEHOLDER: CloudShell benchmark - paste screenshot here -->

{{% notice info %}}
<<<<<<< HEAD
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
=======
**Info:** CloudShell has resource limits (vCPU/RAM/I/O) — timings may differ from EC2/Glue worker types. When comparing, record instance/worker sizing to make benchmarks reproducible.
{{% /notice %}}

CSV read benchmark:
![Placeholder - CloudShell benchmark](/images/s3-data-storage/placeholder-cloudshell.png)

Parquet read benchmark:
![Placeholder - CloudShell benchmark](/images/s3-data-storage/placeholder-cloudshell-parquet.png)

---

### 5.2. Benchmark Results (CloudShell)

**CSV – read full `raw/transactions.csv` from S3**

5 runs:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
151.91s, 146.34s, 141.52s, 126.03s, 115.95s
```

<<<<<<< HEAD
Average calculation (approximate):
=======
Approximate averages:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

- Avg time ≈ **136.35 s**
- Size = **4,593.65 MB**
- Avg throughput ≈ **33.7 MB/s**
- Rows/s ≈ **~248k rows/s**

---

**Parquet – read 1 file ~458.45 MB from S3**

<<<<<<< HEAD
5 measurements:
=======
5 runs:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```text
61.37s, 53.65s, 52.51s, 49.66s, 49.55s
```

<<<<<<< HEAD
Average calculation (approximate):
=======
Approximate averages:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

- Avg time ≈ **53.35 s**
- Size = **458.45 MB**
- Avg throughput ≈ **8.6 MB/s**
- Rows/s ≈ **~635k rows/s**

---

<<<<<<< HEAD
### 5.3. Comparison table (CloudShell)

|    Type | Size on S3 | Avg time (s) | Avg throughput (MB/s) |       Rows | Rows/s (approx) | Relative rows/s |
| ------: | ----------: | -----------: | --------------------: | ---------: | --------------: | --------------: |
|     CSV | 4,593.65 MB |       136.35 |                  33.7 | 33,850,823 |           ~248k |              1× |
| Parquet |   458.45 MB |        53.35 |                   8.6 | 33,850,823 |           ~635k |           ~2.6× |

**Explanation:**

- By **MB/s**, CSV seems "faster" because each run processes more MB (4.59 GB).
- But in terms of **rows per second (rows/s)**, Parquet is **~2.6× faster**, suitable for ETL / training.
=======
### 5.3. Comparison Table (CloudShell)

|    Type |  Size on S3 | Avg time (s) | Avg throughput (MB/s) |       Rows | Rows/s (approx.) | Relative rows/s |
| ------: | ----------: | -----------: | --------------------: | ---------: | ---------------: | --------------: |
|     CSV | 4,593.65 MB |       136.35 |                  33.7 | 33,850,823 |            ~248k |              1× |
| Parquet |   458.45 MB |        53.35 |                   8.6 | 33,850,823 |            ~635k |           ~2.6× |

**Explanation:**

- By **MB/s**, CSV may look “faster” because each run processes many more MB (4.59 GB).
- But by **rows/s**, Parquet is **~2.6× faster**, which is what matters for ETL/training.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

{{% notice info %}}
**CloudShell Conclusion**

<<<<<<< HEAD
- Parquet (Snappy) **dramatically reduces storage size**: 4.59 GB → ~0.46 GB.
- With same 33.85M rows, Parquet processes **~2.6× faster** in rows/s.
=======
- Parquet (Snappy) **reduces storage drastically**: 4.59 GB → ~0.46 GB.
- For the same 33.85M rows, Parquet processes **~2.6× faster** in rows/s.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
  {{% /notice %}}

---

## 6. Benchmark on local machine

<<<<<<< HEAD
### 6.1. Prepare directory & download data
=======
### 6.1. Prepare folder & download data
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

On Windows:

```bash
mkdir s3-local-benchmark
cd s3-local-benchmark
```

<<<<<<< HEAD
Download 2 files:
=======
Download two files:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```bash
aws s3 cp s3://mlops-retail-prediction-dev-842676018087/raw/transactions.csv ./transactions.csv
aws s3 cp s3://mlops-retail-prediction-dev-842676018087/silver/shop_week=200607/run-1761638745394-part-block-0-0-r-00000-snappy.parquet ./transactions_200607.parquet
```

<!-- IMAGE PLACEHOLDER: Local download - paste screenshot here -->

<<<<<<< HEAD
![Placeholder - Local download](/imagess3-data-storage/placeholder-local-download.png)

### 6.2. Benchmark script

Create file `local_benchmark.py`:
=======
## ![Placeholder - Local download](/images/s3-data-storage/placeholder-local-download.png)

### 6.2. Benchmark script

Create `local_benchmark.py`:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

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
<<<<<<< HEAD
        # Read in chunks to avoid RAM overflow
=======
        # Read in chunks to avoid running out of RAM
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
        for chunk in pd.read_csv(path, chunksize=500_000):
            rows += len(chunk)
        t1 = time.time()
        elapsed = t1 - t0
        throughput = size_mb / elapsed
        print(f"[local_csv_stream] run={i} time={elapsed:.2f}s, "
              f"size={size_mb:.2f} MB, throughput={throughput:.2f} MB/s, rows={rows}")

def bench_parquet_stream(path: str, runs: int = 3):
<<<<<<< HEAD
    print(f"=== Benchmark Parquet (streaming): {path} ===")
=======
    print(f"=== Benchmark Parquet: {path} ===")
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

<<<<<<< HEAD
### 6.3. Actual logs

![Placeholder - Local download](/imagess3-data-storage/placeholder-result-readfile.png)

**Observations:**

- CSV full 4.59 GB: still processable thanks to chunk reading, throughput ~90–95 MB/s.
- Parquet (sample 1 week, 6.48 MB): read time ~0.05–0.09s → extremely low latency.
- With many small Parquet files (partitioned by `shop_week`), querying by week/month will be very fast.
=======
### 6.3. Real logs

## ![Placeholder - Local download](/images/s3-data-storage/placeholder-result-readfile.png)

**Notes:**

- Full CSV 4.59 GB: still manageable by chunk reading, throughput ~90–95 MB/s.
- Parquet (one week sample, 6.48 MB): read time ~0.05–0.09s → very low latency.
- With many small Parquet files (partitioned by `shop_week`), week/month queries will be very fast.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

---

{{% notice warning %}}
<<<<<<< HEAD
**Warning (Local):** On local machine, reading entire Parquet dataset into memory (`pd.read_parquet`) can cause RAM shortage. Use reading by pieces (pyarrow dataset, iter_batches) or increase `chunksize` when using CSV to avoid OOM.
{{% /notice %}}

{{% notice tip %}}
**Tip (Local):** When benchmarking on Windows, close other heavy programs and IO results may differ between HDD and SSD — note drive type (SSD/HDD) in benchmark report.
=======
**Warning (Local):** On a local machine, reading the full Parquet dataset into memory (`pd.read_parquet`) may cause OOM. Use piece-wise reads (pyarrow dataset, iter_batches), or tune CSV `chunksize` to avoid OOM.
{{% /notice %}}

{{% notice tip %}}
**Tip (Local):** On Windows, close heavy apps during benchmarking. Disk I/O differs between HDD and SSD — include your disk type (SSD/HDD) in the benchmark report.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
{{% /notice %}}

## 7. IAM – Minimum permissions for Glue Job (summary)

Minimum required:

- **S3:**
  - `s3:GetObject` for `raw/*`
  - `s3:PutObject` for `silver/*`
- **Glue:**
<<<<<<< HEAD
  - Permission to create/run job, read metadata (depends on environment).
=======
  - Permissions to create/run jobs, read metadata (depends on environment).
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
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

<<<<<<< HEAD
**About architecture:**

- Design bucket following MLOps standard:
=======
**Architecture:**

- Standard MLOps bucket design:
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

  ```text
  raw/ → silver/ → gold/ → artifacts/
  ```

<<<<<<< HEAD
- Maintain **raw/ immutable**.
- Standardize data to **Parquet (Snappy)** in **silver/**.

**About performance (from your actual measurements):**

- 4.59 GB CSV → ~0.46 GB Parquet for same **33.85M rows**.
=======
- Keep **raw/ immutable**.
- Standardize data into **Parquet (Snappy)** under **silver/**.

**Performance (from your real benchmarks):**

- 4.59 GB CSV → ~0.46 GB Parquet for the same **33.85M rows**.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- On CloudShell:

  - CSV: ~136s, ~248k rows/s.
  - Parquet: ~53s, ~635k rows/s → **~2.6× rows/s**.

- On local machine (16GB RAM):

<<<<<<< HEAD
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
=======
  - CSV 4.59 GB is still manageable with **500k-row chunks**.
  - One-week Parquet sample (~6.48 MB) reads in **~0.05–0.09s**.

**Cost & operations:**

- Parquet + Snappy **significantly reduces storage** → lower S3 cost.
- Intelligent-Tiering automatically reduces storage class for old data.
- Glue Visual ETL reduces coding and is easier to present in reports.

## 9. Clean Up Resources (AWS CLI)

### 9.1. Delete all objects in the S3 bucket

```bash
# Delete all files in the bucket
aws s3 rm s3://mlops-retail-prediction-dev-842676018087 --recursive

# Verify the bucket is empty
aws s3 ls s3://mlops-retail-prediction-dev-842676018087 --recursive
```

### 9.2. Delete the S3 bucket

```bash
# Delete the bucket (only when empty)
aws s3 rb s3://mlops-retail-prediction-dev-842676018087

# Verify deletion
aws s3 ls | grep mlops-retail-prediction-dev
```

### 9.3. Delete the Glue Job
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

```bash
# List Glue jobs
aws glue get-jobs --query 'Jobs[?contains(Name, `csv-to-parquet`)].Name'

<<<<<<< HEAD
# Delete Glue job
aws glue delete-job --job-name csv-to-parquet-converter

# Check job is deleted
aws glue get-job --job-name csv-to-parquet-converter
```

### 9.4. Delete IAM Role (if created specifically for Glue)

```bash
# Detach policies from role
=======
# Delete the Glue job
aws glue delete-job --job-name csv-to-parquet-converter

# Verify the job is deleted
aws glue get-job --job-name csv-to-parquet-converter
```

### 9.4. Delete the IAM Role (if created specifically for Glue)

```bash
# Detach policies from the role
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
aws iam detach-role-policy --role-name GlueETLRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# Delete inline policies (if any)
aws iam delete-role-policy --role-name GlueETLRole --policy-name S3AccessPolicy

<<<<<<< HEAD
# Delete role
=======
# Delete the role
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
aws iam delete-role --role-name GlueETLRole
```

{{% notice success %}}
<<<<<<< HEAD
**Success tip:** If you delete resources to avoid costs, check CloudWatch log groups and Athena query history — some logs or query history may still store metadata and cause small costs if not cleaned up.
=======
**Success tip:** If you delete resources to avoid costs, also check CloudWatch log groups and Athena query history — some logs or metadata can still generate small costs if not cleaned up.
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
{{% /notice %}}

---

## 10. S3 Storage Pricing Table (ap-southeast-1)

### 10.1. Storage cost by class

<<<<<<< HEAD
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
=======
| Storage Class           | Price (USD/GB/month) | Minimum Duration | Notes                         |
| ----------------------- | -------------------- | ---------------- | ----------------------------- |
| **S3 Standard**         | $0.025               | None             | Frequent access               |
| **S3 Standard-IA**      | $0.0138              | 30 days          | Infrequent access             |
| **S3 One Zone-IA**      | $0.011               | 30 days          | Single AZ                     |
| **S3 Glacier Instant**  | $0.005               | 90 days          | Archive, instant retrieval    |
| **S3 Glacier Flexible** | $0.0045              | 90 days          | Archive, 1-12 hours retrieval |
| **S3 Deep Archive**     | $0.002               | 180 days         | Long-term archive, 12+ hours  |

### 10.2. Request costs

| Request Type          | Price (USD/1000 requests) | Notes                |
| --------------------- | ------------------------- | -------------------- |
| **PUT/POST/LIST**     | $0.0055                   | Write operations     |
| **GET/SELECT**        | $0.00044                  | Read operations      |
| **Data Transfer OUT** | $0.12/GB                  | First 1GB free/month |

### 10.3. Cost estimate for the project

**Current data:**

- Raw CSV: 4.59 GB
- Silver Parquet: 0.46 GB
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
- **Total:** ~5 GB

**Monthly cost (S3 Standard):**

<<<<<<< HEAD
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
=======
| Component             | Size        | Price/GB | Monthly Cost      |
| --------------------- | ----------- | -------- | ----------------- |
| Raw data (CSV)        | 4.59 GB     | $0.025   | $0.11             |
| Silver data (Parquet) | 0.46 GB     | $0.025   | $0.01             |
| Gold + artifacts      | ~0.5 GB     | $0.025   | $0.01             |
| **Total Storage**     | **~5.5 GB** |          | **$0.14**         |
| Requests (estimated)  | ~1000 req   | $0.0055  | $0.006            |
| **Grand Total**       |             |          | **≈ $0.15/month** |

**With Intelligent-Tiering:**

- After 30 days: Raw data moves to Standard-IA → saves ~45%
- After 90 days: Old artifacts move to Glacier → saves ~80%
- **Estimated savings:** ~$0.05–0.08/month

{{% notice info %}}
**💰 Optimized storage cost**

- **Current:** ~$0.15/month for 5.5GB
- **With Intelligent Tiering:** ~$0.07–0.10/month
- **Parquet format:** ~90% storage reduction vs CSV
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48
  {{% /notice %}}

---

<<<<<<< HEAD
## 📹 Task 3 Implementation Video
=======
{{% notice success %}}
**🎯 Task 3 completed**

- Clear, standard MLOps S3 layout.
- CSV → Parquet via Glue Studio (visual, with screenshots placeholders).
- Real benchmarks on **CloudShell** and **local**, with concrete numbers.
- Detailed **clean up commands** and **pricing breakdown**.
- Easy to present in report & demo.
  {{% /notice %}}

---

## 📹 Task 3 Walkthrough Video
>>>>>>> e2332b6d9a96695941b1fb2baeb1eb38bfa46e48

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
