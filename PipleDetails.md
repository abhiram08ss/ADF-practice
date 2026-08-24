# ADF Weekend Assessment Pipeline

## Overview

This Azure Data Factory (ADF) pipeline loads user data from a REST API into Azure SQL Database and processes it through a data transformation workflow.

\---

## What Does This Pipeline Do?

```
REST API
   ↓
Copy JSON Data
   ↓
Truncate users Table
   ↓
Transform \& Clean Data (Data Flow)
   ↓
Merge into curated\_users
   ↓
Complete ✓
```

\---

## Pipeline Activities (Step by Step)

### 1\. **Lookup1** (Get Last Watermark)

* **What:** Reads the last processed ID from the watermark table
* **Why:** Tracks which records were already processed (prevents reprocessing)
* **Query:**

```sql
  SELECT LastProcessedId FROM adfassesment.ETL\_Watermark
  WHERE TargetTable = 'Users'
  ```

* **Output:** Last processed ID stored in variable `watermark`

\---

### 2\. **Set variable1** (Store Watermark)

* **What:** Saves the watermark value to a pipeline variable
* **Why:** Used by Data Flow to filter new records only
* **Variable:** `@activity('Lookup1').output.firstRow.LastProcessedId`
* **Next Step:** Copy data1 starts

\---

### 3\. **Copy data1** (Fetch from REST API)

* **What:** Downloads user data from REST API as JSON
* **Source:** HTTP REST API (GET request)
* **Format:** JSON
* **Destination:** Azure Blob Storage (ResponseDataset)
* **Datasets Used:**

  * Input: `RestApiRequest` (API endpoint)
  * Output: `ResponseDataset` (Blob storage)
* **Next Step:** Script1 runs

\---

### 4\. **Script1** (Clear users Table)

* **What:** Deletes all existing records from users table
* **SQL Command:**

```sql
  TRUNCATE TABLE adfassesment.users;
  ```

* **Why:** Fresh load - remove old data before inserting new
* **Speed:** Faster than DELETE (no log entries)
* **Next Step:** Data flow1 starts

\---

### 5\. **Data flow1** (Transform \& Process)

* **What:** Applies transformations to the REST API data
* **Input:** JSON from ResponseDataset
* **Processing:**

  * Flatten JSON structure
  * Convert data types (string → date, int, etc)
  * Add derived columns (createdDate, loadDate)
  * Validate data quality
  * Filter invalid records
* **Output:** Cleaned data inserted into users table
* **Watermark Parameter:** Uses `@variables('watermark')` to filter new records
* **Compute:** 8 cores (General compute type)
* **Next Step:** Stored procedure1 runs

\---

### 6\. **Stored procedure1** (Merge to Curated)

* **What:** Calls stored procedure to merge users into curated\_users
* **Procedure:** `sp\_merge\_users\_to\_curated`
* **Logic:** UPSERT

  * If user ID exists → UPDATE all columns
  * If user ID is new → INSERT new row
* **Result:** curated\_users table has cleaned, deduplicated data
* **End:** Pipeline completes

\---

## Data Flow

### Activity Dependencies (Order of Execution)

```
Lookup1
   ↓
Set variable1
   ├─→ Copy data1
   │      ↓
   │   Script1
   │      ↓
   │   Data flow1
   │      ↓
   └─→ Stored procedure1
         ↓
      Complete ✓
```

**Explanation:**

* `Lookup1` starts first (no dependencies)
* `Set variable1` waits for `Lookup1` to succeed
* `Copy data1` waits for `Set variable1` to succeed
* `Script1` waits for `Copy data1` to succeed
* `Data flow1` waits for `Script1` to succeed
* `Stored procedure1` waits for `Data flow1` to succeed

\---

## Tables Involved

### Source Table: `users`

|Column|Type|Purpose|
|-|-|-|
|id|INT|Primary Key|
|firstName|NVARCHAR|First name|
|lastName|NVARCHAR|Last name|
|email|NVARCHAR|Email|
|birthDate|DATE|Birth date|
|createdDate|DATETIME2|When loaded|

### Watermark Table: `ETL\_Watermark`

|Column|Type|Purpose|
|-|-|-|
|TargetTable|NVARCHAR|Table name (e.g., 'Users')|
|LastProcessedId|INT|Last processed record ID|

### Target Table: `curated\_users`

|Column|Type|Purpose|
|-|-|-|
|id|INT|Primary Key|
|firstName|NVARCHAR|First name|
|lastName|NVARCHAR|Last name|
|email|NVARCHAR|Email|
|birthDate|DATE|Birth date|
|createdDate|DATETIME2|When created/updated|

\---

## Key Concepts

### Watermark Pattern

* **Purpose:** Tracks incremental loads (only new/modified data)
* **How it works:**

  1. Lookup1 reads last processed ID
  2. Data flow filters: `id > @watermark`
  3. Only new records are processed
  4. Reduces data volume \& improves performance

### TRUNCATE vs DELETE

* **TRUNCATE:** Removes all rows, faster, no transaction log
* **DELETE:** Removes rows, slower, creates log entries
* **Used here:** TRUNCATE (Script1) for speed

### UPSERT (Update or Insert)

* **If exists:** Update the record
* **If not exists:** Insert new record
* **Used in:** Stored procedure1 (MERGE statement)

### Data Flow

* **Transformation engine** in ADF
* **Capabilities:** type conversion, filtering, aggregation, joining
* **Used here:** Clean and validate REST API data before storing

\---

## Variables

### `watermark`

* **Type:** Integer
* **Value:** Last processed user ID
* **Source:** Lookup1 query
* **Usage:** Data flow filtering (only new records)

## Datasets Used

### 1\. **RestApiRequest**

* Type: REST API
* Format: JSON
* Purpose: Fetch user data from API

### 2\. **ResponseDataset**

* Type: Azure Blob Storage
* Format: JSON
* Purpose: Store API response temporarily

### 3\. **DS\_WATERMARK**

* Type: Azure SQL Database
* Purpose: Store/read watermark values

\---

## How to Run

### Manual Execution

1. Open ADF Pipeline: `adfWeekendAssesment`
2. Click **"Debug"** (top toolbar)
3. Pipeline runs immediately
4. Monitor activities in output panel

\---

\---

## Summary

**This pipeline:**

* ✅ Fetches user data from REST API
* ✅ Cleans and transforms the data
* ✅ Stores in Azure SQL Database
* ✅ Uses watermark for incremental loads
* ✅ Merges into curated table for analytics

**Total time:** \~5-10 minutes (depending on data volume)

