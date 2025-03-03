---
title: "Ringba AWS Data Pipeline"
date: 2025-01-13
categories: 
  - blog
---
---

## Project Overview

**Objective:** Automatically ingest call log data from Ringba, store it in Amazon S3, then load it daily into an Amazon RDS (PostgreSQL) database using AWS Glue. Finally, connect to the database via Power BI for reporting.

### Key AWS Services:
1. **AWS Lambda** – Fetches data from the Ringba API and writes CSV to S3.
2. **Amazon EventBridge** – Schedules the Lambda function to run daily.
3. **Amazon S3** – Stores the daily CSV files.
4. **AWS Glue** – Transforms and loads new data (only) into RDS using job bookmarks.
5. **Amazon RDS (PostgreSQL)** – Stores ingested data for easy querying and integration with BI tools.
6. **Power BI** – Connects via ODBC or native PostgreSQL connector to visualize the final data.

---

## 2. High-Level Architecture

1. **EventBridge (cron schedule) → Lambda**
   * Runs every morning (or end-of-day) to fetch Ringba data for "yesterday."
2. **Lambda → S3**
   * Writes `ringba-data-YYYY-MM-DD.csv` into a versioned S3 bucket.
3. **AWS Glue (also on a daily schedule)**
   * Uses Glue bookmarks to only process CSV files not previously ingested.
   * Loads the data into the `ringba_calllogs` table in RDS.
4. **Power BI**
   * Connects to the RDS instance to display dashboards.

**Screenshot Suggestion #1:** An architecture diagram or high-level overview.

---

## 3. AWS Lambda Setup

### 3.1 Lambda Purpose
* Fetch daily call logs from the Ringba API endpoint.
* Format them into CSV.
* Save to `s3://ringba-calllogs/`.

### 3.2 Scheduling with EventBridge
* Created a cron rule in EventBridge, e.g., `cron(0 0 * * ? *)`, to run Lambda at midnight UTC daily.
* This triggers the Lambda, which fetches data for "yesterday."

**Screenshot Suggestion #2:** EventBridge rule configuration page, showing the cron expression.

### 3.3 Lambda Code Sample

```python
import os
import json
import csv
import io
import requests
import boto3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    # 1. Determine 'yesterday'
    today = datetime.utcnow().date()
    yesterday = today - timedelta(days=1)

    # 2. Fetch data from Ringba API (example request)
    ringba_token = os.environ.get("RINGBA_API_TOKEN")
    if not ringba_token:
        raise ValueError("Missing RINGBA_API_TOKEN env var")

    api_url = "https://api.ringba.com/v2/..."
    request_body = {
        "reportStart": f"{yesterday}T00:00:00Z",
        "reportEnd": f"{today}T00:00:00Z",
    }

    headers = {
        "Authorization": f"Token {ringba_token}",
        "Content-Type": "application/json",
    }
    response = requests.post(api_url, json=request_body, headers=headers)
    data = response.json()

    # 3. Convert data to CSV in memory
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["Date", "CampaignName", "CallCount", "OtherColumns..."])
    for record in data["report"]["records"]:
        writer.writerow([
            record.get("date"),
            record.get("campaignName"),
            record.get("callCount"),
        ])
    csv_data = output.getvalue()

    # 4. Write CSV to S3
    s3 = boto3.client("s3")
    bucket_name = "ringba-calllogs"
    file_key = f"ringba-data-{yesterday}.csv"
    s3.put_object(Bucket=bucket_name, Key=file_key, Body=csv_data)

    print(f"Uploaded {file_key} to {bucket_name}")
    return {"statusCode": 200, "body": f"Data for {yesterday} fetched and stored."}
```

**Screenshot Suggestion #3:** Lambda function code in the AWS console or reference to your IDE.

---

## 7. Putting It All Together
1. **EventBridge runs Lambda daily**
   * Lambda writes `ringba-data-YYYY-MM-DD.csv` to S3.
2. **Glue (bookmarked job) runs daily**
   * Picks up only the new CSV, loads it into `ringba_calllogs`.
3. **Power BI refreshes data from RDS**
   * Shows updated call stats in dashboards.

**Screenshot Suggestion #4:** A sample daily run in CloudWatch logs or the AWS Glue logs, demonstrating the pipeline success message and row counts.

---

## 8. Challenges & Solutions

### Network & VPC Setup
* Ensured the Glue job runs in the same VPC as RDS, correct subnets/security groups, etc.

### SSL & Hostname
* For Power BI, installed the RDS certificate to avoid the "invalid certificate" error.

### Job Bookmarks
* Verified that old CSVs are skipped to prevent data duplication.

### Daily Scheduling
* Used separate schedules for Lambda and Glue to maintain data flow timing.

**Screenshot Suggestion #5:** A sample daily run in AWS Glue logs, showing successful data processing.

---

## 9. Conclusion
This pipeline:
* Automates data ingest from Ringba to S3.
* Loads daily into a managed Postgres database.
* Provides up-to-date analytics in Power BI.

**Key Takeaways:**
* Using Glue bookmarks prevents reprocessing old CSV data.
* Proper scheduling in EventBridge orchestrates the entire workflow without manual intervention.
* A correct setup of security groups, IAM roles, and SSL ensures secure, stable operation.
