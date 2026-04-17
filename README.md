# Telecom CDR Validation, Revenue Calculation & High Usage Alert System

## Project Overview

This project implements a complete Telecom Call Detail Record (CDR) ETL pipeline using Informatica Intelligent Cloud Services (IICS).

The solution ingests daily telecom CSV files, performs cleansing and validation, enriches the data with customer and tower information, calculates revenue, detects abnormal customer usage, generates audit logs, and sends email alerts for unusually high call duration.

---

# Objectives

* Read daily telecom CDR files
* Clean and standardize telecom data
* Remove duplicate call records
* Validate phone numbers and duration values
* Enrich data using customer and tower master tables
* Calculate revenue and international call flag
* Load data into staging and final tables
* Detect high usage customers
* Maintain audit logs and rejected records
* Trigger automated email alerts

---

# Source Files

## 1. CDR_RAW_Telecom.csv

| Column    | Description               |
| --------- | ------------------------- |
| CallID    | Unique call identifier    |
| Caller    | Calling phone number      |
| Receiver  | Receiving phone number    |
| Duration  | Call duration in seconds  |
| CallType  | LOCAL / STD / INT         |
| TowerID   | Tower used for the call   |
| Timestamp | Date and time of the call |

## 2. Customer_Master_Telecom.csv

| Column        | Description           |
| ------------- | --------------------- |
| Phone Number  | Customer phone number |
| Customer Name | Customer name         |
| Plan Type     | Customer telecom plan |

## 3. Tower_Master_Telecom.csv

| Column  | Description              |
| ------- | ------------------------ |
| TowerID | Telecom tower identifier |
| Region  | Region of the tower      |
| City    | City of the tower        |

---

# Technology Stack

* Informatica Intelligent Cloud Services (IICS)
* Oracle / MySQL / SQL Server
* CSV Flat Files
* SQL
* Taskflow + Notification Task
* Optional Dashboard in Excel or Power BI

---

# Project Folder Structure

```text
telecom-cdr-validation-high-usage-alerts/
│
├── source_files/
│   ├── CDR_RAW_Telecom.csv
│   ├── Customer_Master_Telecom.csv
│   └── Tower_Master_Telecom.csv
│
├── mappings/
│   ├── telecom_cdr_validation_mapping.xml
│   ├── telecom_customer_tower_join_mapping.xml
│   └── telecom_high_usage_alert_mapping.xml
│
├── taskflows/
│   ├── telecom_email_notification_taskflow.xml
│   └── telecom_audit_logging_taskflow.xml
│
├── sql/
│   ├── create_stage_tables.sql
│   ├── create_final_tables.sql
│   ├── create_rejected_cdr_table.sql
│   ├── create_high_usage_alert_table.sql
│   └── create_audit_log_table.sql
│
├── docs/
│   └── workflow_dfd.md
│
└── README.md
```

---

# Database Tables

## Stage Table

```sql
CREATE TABLE STG_CDR_TELECOM (
    CALL_ID           VARCHAR2(50),
    CALLER            VARCHAR2(20),
    RECEIVER          VARCHAR2(20),
    DURATION          NUMBER,
    CALL_TYPE         VARCHAR2(10),
    TOWER_ID          VARCHAR2(20),
    CALL_TIMESTAMP    TIMESTAMP
);
```

## Final Table

```sql
CREATE TABLE FACT_CDR_TELECOM (
    CALL_ID             VARCHAR2(50),
    CALLER              VARCHAR2(20),
    RECEIVER            VARCHAR2(20),
    CUSTOMER_NAME       VARCHAR2(100),
    PLAN_TYPE           VARCHAR2(50),
    REGION              VARCHAR2(50),
    CITY                VARCHAR2(50),
    DURATION            NUMBER,
    CALL_TYPE           VARCHAR2(10),
    REVENUE             NUMBER,
    IS_INTERNATIONAL    NUMBER,
    LOAD_DATE           TIMESTAMP
);
```

## Rejected Records Table

```sql
CREATE TABLE REJECTED_CDR_RECORDS (
    CALL_ID          VARCHAR2(50),
    CALLER           VARCHAR2(20),
    RECEIVER         VARCHAR2(20),
    DURATION         NUMBER,
    REJECT_REASON    VARCHAR2(200),
    REJECTED_TIME    TIMESTAMP
);
```

## High Usage Alert Table

```sql
CREATE TABLE HIGH_USAGE_ALERT (
    CUSTOMER_PHONE      VARCHAR2(20),
    TOTAL_DURATION      NUMBER,
    TOTAL_REVENUE       NUMBER,
    ALERT_STATUS        VARCHAR2(20),
    ALERT_TIME          TIMESTAMP
);
```

## Audit Log Table

```sql
CREATE TABLE LOAD_AUDIT_LOG (
    LOAD_ID            NUMBER,
    LOAD_DATE          TIMESTAMP,
    SOURCE_COUNT       NUMBER,
    SUCCESS_COUNT      NUMBER,
    FAILURE_COUNT      NUMBER,
    STATUS             VARCHAR2(20),
    ERROR_MESSAGE      VARCHAR2(500)
);
```

---

# Data Transformation Rules

| Rule                 | Logic                                |
| -------------------- | ------------------------------------ |
| Remove Duplicates    | Based on CallID                      |
| Missing Duration     | Replace NULL with 0                  |
| Invalid Phone Number | Reject if not 10 digits              |
| Negative Duration    | Reject record                        |
| Revenue              | Duration × 0.02                      |
| IsInternational      | 1 if CallType = 'INT' else 0         |
| Date Standardization | Convert timestamp to standard format |

---

# Informatica Mapping Workflow

```text
Source CSV
   ↓
Source Qualifier
   ↓
Expression Transformation
   ├─ Replace NULL duration with 0
   ├─ Standardize timestamp
   ├─ Calculate Revenue
   ├─ Calculate IsInternational
   └─ Add Validation Flags
   ↓
Sorter
   ↓
Aggregator / Dedup Logic
   ↓
Router
   ├─ Valid Records → Join with Customer Master → Join with Tower Master → Final Table
   └─ Invalid Records → Rejected Table
```

---

# Detailed DFD Workflow

```text
+-----------------------+
|  CDR_RAW_Telecom.csv  |
+-----------------------+
            |
            v
+-----------------------+
|   Read Source File    |
+-----------------------+
            |
            v
+-----------------------+
| Remove Duplicate Rows |
|     using CallID      |
+-----------------------+
            |
            v
+-----------------------+
| Standardize Timestamp |
| Replace NULL Duration |
+-----------------------+
            |
            v
+-----------------------+
| Validation Checks     |
| - Invalid Caller      |
| - Invalid Receiver    |
| - Negative Duration   |
+-----------------------+
       |                          
       | Valid                    | Invalid
       v                          v
+-----------------------+    +-----------------------+
| Join Customer Master  |    | Rejected_CDR_Records |
+-----------------------+    +-----------------------+
            |
            v
+-----------------------+
| Join Tower Master     |
+-----------------------+
            |
            v
+-----------------------+
| Calculate Revenue     |
| Revenue=Duration*.02  |
| IsInternational Flag  |
+-----------------------+
            |
            v
+-----------------------+
| Load to Stage Table   |
+-----------------------+
            |
            v
+-----------------------+
| Load to Final Table   |
+-----------------------+
            |
            v
+-----------------------+
| Aggregate Duration by |
| Customer              |
+-----------------------+
            |
            v
+-----------------------+
| High Usage Check      |
| Duration > Threshold  |
+-----------------------+
       |                          
       | Yes                      | No
       v                          v
+-----------------------+    +----------------+
| HIGH_USAGE_ALERT      |    | Normal End     |
+-----------------------+    +----------------+
            |
            v
+-----------------------+
| Send Email Alert      |
+-----------------------+
```

---

# End-to-End Taskflow

```text
Start
  ↓
Task 1: Load Raw CDR Data
  ↓
Task 2: Validate and Reject Invalid Records
  ↓
Task 3: Enrich with Customer + Tower Data
  ↓
Task 4: Load Stage and Final Table
  ↓
Task 5: Insert Audit Log
  ↓
Decision Task
  ├─ If High Usage Exists
  │      ↓
  │   Send Notification Email
  │      ↓
  │   Update HIGH_USAGE_ALERT Table
  │
  └─ Else End
```

---

# Validation Logic Used in Expression Transformation

```text
IS_CALLER_VALID =
IIF(
   LENGTH(CALLER) = 10
   AND REG_MATCH(CALLER,'^[0-9]{10}$'),
   1,
   0
)

IS_RECEIVER_VALID =
IIF(
   LENGTH(RECEIVER) = 10
   AND REG_MATCH(RECEIVER,'^[0-9]{10}$'),
   1,
   0
)

IS_DURATION_VALID =
IIF(DURATION >= 0,1,0)
```

Router Conditions:

```text
VALID_RECORDS:
IS_CALLER_VALID = 1
AND IS_RECEIVER_VALID = 1
AND IS_DURATION_VALID = 1

INVALID_RECORDS:
IS_CALLER_VALID = 0
OR IS_RECEIVER_VALID = 0
OR IS_DURATION_VALID = 0
```

---

# High Usage Alert Logic

Threshold Example:

```text
IF TOTAL_DURATION > 1000 SECONDS
THEN FLAG CUSTOMER AS HIGH USAGE
```

Expression:

```text
IS_HIGH_USAGE = IIF(TOTAL_DURATION > 1000,1,0)
```

---

# Email Notification Configuration

Subject:

```text
Telecom High Usage Alert Notification
```

Body:

```text
One or more telecom customers exceeded the configured call duration threshold.

Please review the HIGH_USAGE_ALERT table.
```

---

# Audit Logging

The project captures:

* Total records read from source
* Total successful records loaded
* Total rejected records
* Execution status
* Error details if mapping fails

Example:

| Source Count | Success Count | Failure Count | Status  |
| ------------ | ------------- | ------------- | ------- |
| 1000         | 950           | 50            | SUCCESS |

---

# Analytical Outputs

## Daily Call Volume Summary

* Total Calls
* Total Call Duration
* Average Duration
* Calls per Customer

## Call Type Analysis

* Calls by Local / STD / INT
* Revenue by Call Type
* Average Duration by Type

## International Call Monitoring

* International Call Count
* International Revenue
* Percentage of INT Calls

## Revenue Analysis

* Revenue per Customer
* Top 2 Customers by Revenue
* Revenue Contribution Percentage

---

# Future Enhancements

* Real-time ingestion using APIs
* Dashboard using Power BI or Tableau
* Predictive high-usage detection using Machine Learning
* SMS notification in addition to email
* Fraud detection for unusual telecom activity
