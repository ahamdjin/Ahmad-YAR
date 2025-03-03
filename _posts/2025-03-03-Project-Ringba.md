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
![Result1](/Ahmad-YAR/assets/images/EventBridge1.png)
![Result1](/Ahmad-YAR/assets/images/LambdaCalllogs.png)

---

## 4. AWS Glue Job
  ### 4.1 Glue Job Purpose
  * Reads all CSV files from s3://ringba-calllogs, but uses job bookmarks so it only ingests new files once.
  * Transforms data columns if necessary (ApplyMapping).
  * Writes the results into PostgreSQL (in your RDS instance).
### 4.2 Job Bookmarks
  *  Ensures the job doesn’t reprocess old CSV files each day.
  *  In the Glue console, set “Job bookmark” to Enable in Advanced properties.
### 4.3 Scheduling
  *  Either via the Glue Scheduler or an additional EventBridge rule.
  *  Runs daily after the Lambda finishes—e.g., 00:10 UTC.
### 4.4 Glue Job Script

```python
Copy
Edit
import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.transforms import ApplyMapping

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'INPUT_PATH'])
sc = SparkContext()
glueContext = GlueContext(sc)
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

def main():
    # 1. Read from S3
    source = glueContext.create_dynamic_frame.from_options(
        connection_type="s3",
        format="csv",
        connection_options={"paths": [args["INPUT_PATH"]], "recurse": True},
        format_options={"withHeader": True}
    )

    # 2. ApplyMapping if column names differ
    mapped = ApplyMapping.apply(
        frame=source,
        mappings=[("Date", "string", "date", "string"), ("CampaignName", "string", "campaignname", "string")]
    )

    # 3. Write to PostgreSQL
    glueContext.write_dynamic_frame.from_jdbc_conf(
        frame=mapped,
        catalog_connection="PostgreSQLConnection",
        connection_options={"dbtable": "ringba_calllogs", "database": "postgres"}
    )
    job.commit()

if __name__ == "__main__":
    main()
```
![Result1](/Ahmad-YAR/assets/images/GlueCalllogsScript.png)
![Result1](/Ahmad-YAR/assets/images/GlueCalllogsRun.png)

---

## 5. Amazon RDS (PostgreSQL)
### 5.1 Table Schema
* sql
    CopyEdit
    CREATE TABLE ringba_calllogs (
    date                varchar(255),
    campaignname        varchar(255),
    callcount           int,
    -- ...
    );

![Result1](/Ahmad-YAR/assets/images/RDSConnectivity.png)

### 5.2 Verifying New Rows
* Use a query like:
    sql
    CopyEdit
    SELECT date, campaignname, callcount
    FROM ringba_calllogs
    ORDER BY date DESC
    LIMIT 10;

We Can see the latest day’s data appear each morning after Glue finishes.

---

## 6. Power BI Integration
 * Created an ODBC or native PostgreSQL connector in Power BI.
 * Provided host = <your-db-endpoint>.rds.amazonaws.com, port = 5432, database = postgres, plus credentials.
 * Installed RDS CA Certificate in Windows trust store if using SSL.

![Result1](/Ahmad-YAR/assets/images/PowerBI.png)

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


## Below are the original Scripts used in the Pipeline

### Lambda Function For Ringba insight

```python
import requests
import csv
import io
from datetime import datetime, timedelta
import boto3
import os

def process_day(day, api_token):
    """
    Fetch data for a specific day from Ringba API and store it in S3.
    
    Args:
        day (datetime.date): The date to process.
        api_token (str): Ringba API token from environment variables.
    """
    # Define the start and end of the day in UTC
    day_start_str = day.strftime("%Y-%m-%dT00:00:00Z")
    day_end_str = (day + timedelta(days=1)).strftime("%Y-%m-%dT00:00:00Z")

    # Ringba API endpoint
    api_url = "https://api.ringba.com/v2/RAa6f60f99a00045baba297656dbfe8893/insights"

    # Headers for API request
    headers = {
        "Authorization": f"Token {api_token}",
        "Content-Type": "application/json",
    }

    # Request body for Ringba API, using day-specific dates
    request_body = {
        "reportStart": day_start_str,
        "reportEnd": day_end_str,
        "groupByColumns": [{"column": "campaignName", "displayName": "Campaign"}],
        "valueColumns": [
            {"column": "callCount", "aggregateFunction": None},
            {"column": "liveCallCount", "aggregateFunction": None},
            {"column": "completedCalls", "aggregateFunction": None},
            {"column": "endedCalls", "aggregateFunction": None},
            {"column": "connectedCallCount", "aggregateFunction": None},
            {"column": "payoutCount", "aggregateFunction": None},
            {"column": "convertedCalls", "aggregateFunction": None},
            {"column": "nonConnectedCallCount", "aggregateFunction": None},
            {"column": "duplicateCalls", "aggregateFunction": None},
            {"column": "blockedCalls", "aggregateFunction": None},
            {"column": "incompleteCalls", "aggregateFunction": None},
            {"column": "earningsPerCallGross", "aggregateFunction": None},
            {"column": "conversionAmount", "aggregateFunction": None},
            {"column": "payoutAmount", "aggregateFunction": None},
            {"column": "profitGross", "aggregateFunction": None},
            {"column": "profitMarginGross", "aggregateFunction": None},
            {"column": "convertedPercent", "aggregateFunction": None},
            {"column": "callLengthInSeconds", "aggregateFunction": None},
            {"column": "avgHandleTime", "aggregateFunction": None},
            {"column": "totalCost", "aggregateFunction": None},
        ],
        "orderByColumns": [{"column": "callCount", "direction": "desc"}],
        "formatTimespans": True,
        "formatPercentages": True,
        "generateRollups": False,  # No summary rows
        "maxResultsPerGroup": 1000,
        "filters": [],
        "formatTimeZone": "America/Los_Angeles",
    }

    # Fetch data from Ringba API
    response = requests.post(api_url, headers=headers, json=request_body)
    if response.status_code != 200:
        raise Exception(f"Failed to fetch data for {day}: {response.status_code}, {response.text}")

    data = response.json()
    report_data = data.get("report", {}).get("records", [])

    # Create CSV in memory
    output = io.StringIO()
    writer = csv.writer(output)

    # Write CSV header
    writer.writerow([
        "Date", "Campaign Name", "Call Count", "Live Call Count", "Completed Calls",
        "Ended Calls", "Connected Call Count", "Payout Count", "Converted Calls",
        "Non-Connected Call Count", "Duplicate Calls", "Blocked Calls",
        "Incomplete Calls", "Earnings Per Call Gross", "Conversion Amount",
        "Payout Amount", "Profit Gross", "Profit Margin Gross", "Converted Percent",
        "Call Length (Seconds)", "Average Handle Time", "Total Cost"
    ])

    # Write data rows, excluding "Unknown" or null campaigns
    for record in report_data:
        campaign_name = record.get("campaignName")
        if campaign_name and campaign_name != "Unknown":
            writer.writerow([
                day.strftime("%Y-%m-%d"),
                campaign_name,
                record.get("callCount", 0),
                record.get("liveCallCount", 0),
                record.get("completedCalls", 0),
                record.get("endedCalls", 0),
                record.get("connectedCallCount", 0),
                record.get("payoutCount", 0),
                record.get("convertedCalls", 0),
                record.get("nonConnectedCallCount", 0),
                record.get("duplicateCalls", 0),
                record.get("blockedCalls", 0),
                record.get("incompleteCalls", 0),
                record.get("earningsPerCallGross", 0.0),
                record.get("conversionAmount", "0.00"),
                record.get("payoutAmount", "0.00"),
                record.get("profitGross", "0.00"),
                record.get("profitMarginGross", "0.00%"),
                record.get("convertedPercent", "0.00%"),
                record.get("callLengthInSeconds", "00:00:00"),
                record.get("avgHandleTime", "00:00:00"),
                record.get("totalCost", "0.00"),
            ])

    csv_data = output.getvalue()

    # Upload to S3
    s3 = boto3.client("s3")
    bucket_name = "ringba-insight"
    file_key = f"ringba-data-{day.strftime('%Y-%m-%d')}.csv"

    s3.put_object(
        Bucket=bucket_name,
        Key=file_key,
        Body=csv_data
    )

    print(f"Stored data for {day} to s3://{bucket_name}/{file_key}")


def lambda_handler(event, context):
    """
    Lambda handler to process data for 'yesterday' in UTC.
    
    Args:
        event (dict): Event data (not used here).
        context (object): Lambda context object.
    
    Returns:
        dict: Response with status code and message.
    """
    # Get Ringba API token
    api_token = os.environ.get("RINGBA_API_TOKEN")
    if not api_token:
        raise ValueError("Missing RINGBA_API_TOKEN in environment variables")

    # Calculate 'yesterday' in UTC
    yesterday = datetime.utcnow().date() - timedelta(days=1)

    # Fetch and store data for yesterday
    process_day(yesterday, api_token)

    return {
        "statusCode": 200,
        "body": f"Successfully processed data for {yesterday}"
    }
```
### Lambda Function For Ringba Call logs

```python
import requests
import csv
import io
from datetime import datetime, timedelta
import boto3
import os

def lambda_handler(event, context):
    """
    Fetch Ringba /calllogs data for *yesterday* only, and upload a single CSV
    into the 'ringba-calllogs' S3 bucket. Uses the smaller 'value_columns' set
    you included, but you can easily add more columns.

    1. Determines yesterday's date in UTC
    2. Builds the ringba POST request for that 24-hour window
    3. Collects records, builds CSV in memory, uploads to S3
    """

    # -- 1) Figure out "yesterday" in UTC
    today_utc = datetime.utcnow().date()
    yesterday_utc = today_utc - timedelta(days=1)

    # Start of yesterday (00:00 UTC)
    start_dt = datetime(yesterday_utc.year, yesterday_utc.month, yesterday_utc.day)
    # End of yesterday (which is 00:00 UTC "today")
    end_dt   = start_dt + timedelta(days=1)

    day_str = yesterday_utc.strftime("%Y-%m-%d")
    day_start = start_dt.strftime("%Y-%m-%dT00:00:00Z")
    day_end   = end_dt.strftime("%Y-%m-%dT00:00:00Z")

    # -- 2) Read Ringba token from environment variable
    api_token = os.environ.get("RINGBA_API_TOKEN")
    if not api_token:
        raise ValueError("Missing RINGBA_API_TOKEN in environment variables.")

    # Example account ID, adjust if needed
    account_id = "RAa6f60f99a00045baba297656dbfe8893"
    api_url = f"https://api.ringba.com/v2/{account_id}/calllogs"

    # -- 3) Prepare S3 client & bucket name
    s3 = boto3.client('s3')
    bucket_name = "ringba-calllogs"

    # -- 4) The columns you want (based on your snippet)
    value_columns = [
        {"column": "callDt"},  
        {"column": "inboundCallId"},  
        {"column": "hasConnected"},   
        {"column": "isIncomplete"},  
        {"column": "campaignName"},  
        {"column": "publisherName"}, 
        {"column": "inboundPhoneNumber"},
        {"column": "number"},  
        {"column": "timeToCallInSeconds"},
        {"column": "isDuplicate"},
        {"column": "endCallSource"},
        {"column": "targetName"},
        {"column": "callLengthInSeconds"},
        {"column": "conversionAmount"},
        {"column": "payoutAmount"},
    ]

    # -- 5) Build the request body for "yesterday"
    request_body = {
        "reportStart": day_start,
        "reportEnd": day_end,
        "orderByColumns": [{"column": "callDt", "direction": "desc"}],
        "filters": [],
        "valueColumns": value_columns,
        "formatTimespans": True,
        "formatPercentages": True,
        "formatDateTime": True,
        "formatTimeZone": "America/Sao_Paulo",
        "size": 150,
        "offset": 0
    }

    print(f"Fetching yesterday's data ({day_str}) from {day_start} to {day_end}...")

    resp = requests.post(
        api_url,
        headers={
            "Authorization": f"Token {api_token}",
            "Content-Type": "application/json"
        },
        json=request_body
    )

    if resp.status_code != 200:
        print(f"ERROR {resp.status_code}: {resp.text}")
        return {
            "statusCode": resp.status_code,
            "body": f"Failed to fetch data: {resp.text}"
        }

    data = resp.json()
    records = data.get("report", {}).get("records", [])
    print(f"  Received {len(records)} records for {day_str}.")

    # -- 6) Build CSV in memory
    output = io.StringIO()
    writer = csv.writer(output)

    if len(records) == 0:
        # If no data, just write 2 rows with "Date" and "No Data"
        writer.writerow(["Date", "No Data"])
        writer.writerow([day_str, ""])
    else:
        # Dynamically gather all record keys for columns
        all_fields = set()
        for r in records:
            all_fields.update(r.keys())
        # We'll ensure "Date" is included
        all_fields.add("Date")
        # Sort columns so "Date" is first
        fieldnames_sorted = ["Date"] + sorted(f for f in all_fields if f != "Date")
        # Write header
        writer.writerow(fieldnames_sorted)

        # Fill out each record
        for r in records:
            r["Date"] = day_str
            row_data = [r.get(fn, "") for fn in fieldnames_sorted]
            writer.writerow(row_data)

    csv_content = output.getvalue()

    # -- 7) Upload to S3 with a single file name for "yesterday"
    file_key = f"ringba-calllogs-{day_str}.csv"

    s3.put_object(
        Bucket=bucket_name,
        Key=file_key,
        Body=csv_content
    )

    print(f"Uploaded s3://{bucket_name}/{file_key}")

    return {
        "statusCode": 200,
        "body": f"Successfully fetched yesterday's calllogs for {day_str} into {bucket_name}."
    }
```
### Glue Script For Ringba Insight

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'INPUT_PATH'])
sc = SparkContext()
glueContext = GlueContext(sc)
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

def main():
    try:
        # 1. Read CSV files directly from the specified path (including subfolders)
        source = glueContext.create_dynamic_frame.from_options(
            connection_type="s3",
            format="csv",
            connection_options={
                "paths": [args['INPUT_PATH']],  # e.g. s3://my-bucket/some-prefix/
                "recurse": True  # Reads through subdirectories if needed
            },
            format_options={
                "withHeader": True,
                "separator": ",",
                "quoteChar": '"',
                "escapeChar": "\\",
                "encoding": "UTF-8-BOM"
            }
        )

        # 2. Write the data directly to the PostgreSQL table
        glueContext.write_dynamic_frame.from_jdbc_conf(
            frame=source,
            catalog_connection="Postgresql connection",  # Replace with your actual Glue Connection name
            connection_options={
                "dbtable": "ringba_insight",
                "database": "postgres",
                "batchsize": 1000
            }
        )

        job.commit()
        print("SUCCESS: Data loaded to ringba_insight")

    except Exception as e:
        print(f"ERROR: {str(e)}")
        job.commit()

if __name__ == "__main__":
    main()
```
### Glue Script For Ringba Calllogs

```python
import sys
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.transforms import ApplyMapping

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'INPUT_PATH'])
sc = SparkContext()
glueContext = GlueContext(sc)
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

def main():
    try:
        # 1. Read the CSV from S3
        source = glueContext.create_dynamic_frame.from_options(
            connection_type="s3",
            format="csv",
            connection_options={
                "paths": [args['INPUT_PATH']],  # e.g. "s3://ringba-calllogs/"
                "recurse": True
            },
            format_options={
                "withHeader": True,
                "separator": ",",
                "quoteChar": '"',
                "escapeChar": "\\",
                "encoding": "UTF-8-BOM"
            }
        )

        # DEBUG: Print the schema and some sample rows to confirm the columns
        print("=== CSV Schema (inferred by Glue) ===")
        source.printSchema()

        df_source = source.toDF()
        print("=== Sample Rows ===")
        df_source.show(5, truncate=False)

        # 2. Map CSV columns to your ringba_calllogs table columns
        #    Left side = EXACT CSV headers from your screenshot
        #    Right side = EXACT Postgres column names
        mapped = ApplyMapping.apply(
            frame=source,
            mappings=[
                ("Date",                 "string", "date",                 "string"),
                ("callDt",              "string", "calldt",               "string"),
                ("callLengthInSeconds", "string", "calllengthinseconds",  "string"),
                ("campaignName",        "string", "campaignname",         "string"),
                ("conversionAmount",    "string", "conversionamount",     "string"),
                ("endCallSource",       "string", "endcallsource",        "string"),
                ("hasConnected",        "string", "hasconnected",         "string"),
                ("inboundCallId",       "string", "inboundcallid",        "string"),
                ("inboundPhoneNumber",  "string", "inboundphonenumber",   "string"),
                ("isDuplicate",         "string", "isduplicate",          "string"),
                ("number",              "string", "number",               "string"),
                ("publisherName",       "string", "publishername",        "string"),
                ("targetName",          "string", "targetname",           "string"),
                ("timeToCallInSeconds", "string", "timetocallinseconds",  "string"),
            ]
        )

        # 3. Write the mapped data into ringba_calllogs in PostgreSQL
        glueContext.write_dynamic_frame.from_jdbc_conf(
            frame=mapped,
            catalog_connection="Postgresql connection",  # your actual Glue connection name
            connection_options={
                "dbtable": "ringba_calllogs",
                "database": "postgres",
                "batchsize": 1000
            }
        )

        job.commit()
        print("SUCCESS: Data loaded into ringba_calllogs")

    except Exception as e:
        print(f"ERROR: {str(e)}")
        job.commit()

if __name__ == "__main__":
    main()
```
