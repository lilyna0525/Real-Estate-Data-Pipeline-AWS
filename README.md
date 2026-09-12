# Real Estate Transaction Data Pipeline on AWS

An AWS-based data pipeline that collects Korean apartment real estate transaction data from the public API provided by the Korean Ministry of Land, Infrastructure and Transport (MOLIT), processes the XML response, and stores the data in a MariaDB database hosted on Amazon RDS.

This project was developed as part of my hands-on AWS data engineering practice, with the original course implementation adapted to the current public API and AWS Lambda Python runtime.

---

## Project Overview

The goal of this project is to build a simple serverless data pipeline that automatically:

1. Requests apartment transaction data from the MOLIT public API
2. Parses the XML response
3. Extracts relevant transaction information
4. Connects to Amazon RDS MariaDB
5. Inserts the processed data into a relational database

### Data Flow

```text
MOLIT Public API
       │
       ▼
AWS Lambda (Python 3.14)
       │
       ├── requests
       ├── BeautifulSoup
       └── lxml
       │
       ▼
Amazon RDS
   MariaDB
       │
       ▼
   DataGrip
```

---

## Architecture

```text
                    ┌──────────────────────┐
                    │   MOLIT Public API   │
                    │  Apartment Trading   │
                    │        Data          │
                    └──────────┬───────────┘
                               │
                               │ HTTPS
                               ▼
                    ┌──────────────────────┐
                    │     AWS Lambda       │
                    │      Python 3.14     │
                    │                      │
                    │  • requests          │
                    │  • BeautifulSoup     │
                    │  • lxml              │
                    │  • mysql.connector   │
                    └──────────┬───────────┘
                               │
                               │ TCP 3306
                               ▼
                    ┌──────────────────────┐
                    │     Amazon RDS       │
                    │       MariaDB        │
                    │                      │
                    │   pipelinedb         │
                    │   apart_real_cost    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       DataGrip       │
                    │  Data Verification   │
                    └──────────────────────┘
```

---

## Tech Stack

| Category | Technology |
|---|---|
| Cloud | AWS |
| Compute | AWS Lambda |
| Runtime | Python 3.14 |
| Database | MariaDB |
| Database Hosting | Amazon RDS |
| Object Storage | Amazon S3 |
| API | MOLIT Public Data API |
| Data Parsing | BeautifulSoup, lxml |
| HTTP Requests | requests |
| Database Connector | mysql-connector-python |
| Development Environment | Amazon Linux 2023 EC2 |
| Database Client | DataGrip |
| Containerisation | Docker |
| Monitoring | Amazon CloudWatch |
| Testing | AWS Lambda Test |

---

## Data Collected

The pipeline extracts apartment transaction information including:

- Transaction amount
- Transaction year
- Transaction month
- Transaction day
- Apartment name
- Dong (neighborhood)
- Exclusive use area
- Regional code
- Floor
- Jibun (lot number)
- Building year

The extracted data is transformed into structured Python dictionaries before being inserted into MariaDB.

---

## API Migration

The original course implementation used an older MOLIT API endpoint.

Because the original endpoint was no longer available, the API integration was migrated to the current public API provided through `data.go.kr`.

### Original Implementation

```text
Legacy MOLIT OpenAPI endpoint
```

### Updated Implementation

```text
data.go.kr
RTMSDataSvcAptTradeDev
```

The response structure had also changed.

The current API returns fields such as:

```text
dealAmount
dealYear
umdNm
aptNm
dealMonth
dealDay
excluUseAr
sggCd
floor
jibun
buildYear
```

The XML parsing logic was updated to match the current API response schema.

---

## ☁️ AWS Lambda Implementation

The data ingestion logic was deployed as an AWS Lambda function.

### Lambda Function

```text
apart_real_cost
```

### Runtime

```text
Python 3.14
```

The Lambda function:

1. Calls the MOLIT public API
2. Parses the XML response
3. Creates structured transaction records
4. Connects to Amazon RDS MariaDB
5. Inserts the records into the database
6. Commits the database transaction

---

## Lambda Dependency Packaging

A key technical challenge was packaging external Python dependencies for AWS Lambda.

The Lambda function requires:

```text
requests
beautifulsoup4
lxml
mysql-connector-python
```

The development EC2 environment was running Python 3.9, while the Lambda function used Python 3.14.

To ensure compatibility between the Lambda runtime and packages containing native binaries, the dependencies were built using the AWS Lambda Python 3.14 Docker image.

### Requirements

```text
requests
beautifulsoup4
lxml
mysql-connector-python
```

### Docker Environment

```text
public.ecr.aws/lambda/python:3.14
```

The dependencies were installed into a deployment directory and packaged together with:

```text
lambda_function.py
```

The resulting ZIP package was uploaded to Amazon S3 and used to update the Lambda function.

---

## Database Integration

The Lambda function connects to an Amazon RDS MariaDB instance using:

```python
mysql.connector
```

The processed records are inserted into:

```text
pipelinedb
└── apart_real_cost
```

The database transaction is finalized using:

```python
connection.commit()
```

This ensures that successfully processed transaction records are persisted in the database.

---

## Network Configuration

The RDS instance runs inside an Amazon VPC and listens on:

```text
Port: 3306
```

The RDS instance was configured as publicly accessible for this hands-on learning environment.

The RDS Security Group was configured to allow MySQL connections from the Lambda execution environment.

> Note: This configuration was used for learning purposes. A production implementation should use a more restrictive network architecture and avoid exposing the database publicly.

---

## Monitoring and Troubleshooting

Amazon CloudWatch was used to monitor Lambda executions and investigate runtime issues.

### Issue 1 — Lambda Timeout

The initial Lambda timeout was set to 3 seconds.

The function timed out while performing the API and database operations.

```text
Status: timeout
Duration: 3000 ms
```

The Lambda timeout was increased to allow sufficient time for:

```text
API request
      ↓
XML parsing
      ↓
RDS connection
      ↓
Database insertion
```

---

### Issue 2 — Missing Python Dependencies

The initial Lambda deployment contained only:

```text
lambda_function.py
```

This resulted in:

```text
Runtime.ImportModuleError:
No module named 'requests'
```

The issue was resolved by packaging the required third-party dependencies into the Lambda deployment ZIP.

---

### Issue 3 — Python Runtime Compatibility

The EC2 development environment used:

```text
Python 3.9
```

while the Lambda function used:

```text
Python 3.14
```

To avoid compatibility issues with packages containing native binaries, such as `lxml` and `mysql-connector-python`, the dependencies were rebuilt using the AWS Lambda Python 3.14 Docker image.

The resulting package contained Python 3.14-compatible binaries, including:

```text
_mysql_connector.cpython-314-x86_64-linux-gnu.so
```

---

### Issue 4 — RDS Connection Timeout

After resolving the dependency issue, the Lambda function executed successfully but the data was not inserted into the database.

CloudWatch revealed:

```text
Error: 2003 (HY000):
Can't connect to MySQL server ...:3306
```

The issue was identified as an RDS network and security configuration problem rather than a Python dependency or SQL transaction issue.

After updating the RDS access configuration, the Lambda function successfully connected to MariaDB and inserted the transaction data.

---

## Final Result

The final pipeline successfully completed the end-to-end data flow:

```text
MOLIT Public API
        ↓
    AWS Lambda
        ↓
   XML Parsing
        ↓
Data Transformation
        ↓
   Amazon RDS
      MariaDB
        ↓
     DataGrip
        ↓
Data Successfully Verified
```

The apartment transaction records were successfully inserted into the `apart_real_cost` table and verified through DataGrip.

---

## Repository Structure

```text
real-estate-data-pipeline-aws/
│
├── lambda_function.py
├── requirements.txt
├── README.md
│
├── screenshots/
│   ├── 01-lambda-function.png
│   ├── 02-lambda-test-success.png
│   ├── 03-cloudwatch-logs.png
│   ├── 04-rds-connectivity.png
│   ├── 05-datagrip-result.png
│   └── 06-architecture.png
│
└── .gitignore
```

---

## Key Learning Outcomes

Through this project, I gained hands-on experience with:

- Building a serverless data ingestion pipeline using AWS Lambda
- Working with public government APIs
- Migrating an outdated API integration to a current API
- Parsing XML data using BeautifulSoup and lxml
- Connecting AWS Lambda to Amazon RDS MariaDB
- Packaging Python dependencies for AWS Lambda
- Managing Python runtime compatibility
- Using Docker to build Lambda-compatible dependencies
- Troubleshooting Lambda timeout issues
- Troubleshooting AWS networking and security group configuration
- Monitoring Lambda executions using CloudWatch
- Validating pipeline results using DataGrip

---

## Future Improvements

Potential improvements for a production-oriented implementation include:

- Store API keys and database credentials in AWS Secrets Manager
- Move credentials out of source code and environment-specific files
- Avoid hard-coded API parameters
- Add configurable date and region parameters
- Add error handling and retry logic
- Introduce structured logging
- Schedule automatic data collection using Amazon EventBridge
- Store raw API responses in Amazon S3
- Build an ETL layer for data quality validation
- Visualise real estate transaction trends using Power BI
- Add infrastructure as code using AWS CloudFormation or Terraform

---

## What I Practised

This project helped me move beyond simply following a tutorial by adapting an existing pipeline to a current AWS and API environment.

The original implementation required several changes due to differences between the course environment and the current AWS and API environment.

I independently troubleshot:

- API endpoint compatibility
- API response schema changes
- Lambda runtime compatibility
- Python dependency packaging
- Lambda execution timeouts
- RDS connectivity
- Security group configuration

The final result was a working end-to-end serverless data pipeline that collects real estate transaction data from a public API and loads it into a relational database hosted on Amazon RDS.

---

## Project Highlights

**Cloud:** AWS  
**Compute:** AWS Lambda  
**Database:** Amazon RDS / MariaDB  
**Language:** Python  
**Runtime:** Python 3.14  
**Containerisation:** Docker  
**Data Source:** MOLIT Public Data API  
**Monitoring:** Amazon CloudWatch
