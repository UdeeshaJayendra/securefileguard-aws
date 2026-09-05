# 🛡️ SecureFileGuard — Serverless File Security & Threat Detection

SecureFileGuard is a serverless AWS security platform that automatically analyzes uploaded files, detects suspicious characteristics, quarantines potentially dangerous files, stores scan results, and sends security alerts.

The project demonstrates practical **cloud security, serverless architecture, threat detection, IAM least privilege, event-driven processing, reliability, monitoring, and Infrastructure as Code** using AWS and Terraform.

---

##  Project Overview

The system provides an automated pipeline for analyzing uploaded files:

```text
User / API Client
       │
       ▼
 API Gateway
       │
       ▼
 Upload Lambda
       │
       ▼
 S3 ── uploads/
       │
       ▼
 S3 Event Notification
       │
       ▼
 SQS Security Queue
       │
       ▼
 Scanner Lambda
       │
       ├───────────────┐
       ▼               ▼
    CLEAN           THREAT
       │               │
       ▼               ▼
   S3 clean/      S3 quarantine/
       │               │
       └───────┬───────┘
               ▼
          DynamoDB
        Scan Results
               │
               ▼
          SNS Alerts
               │
               ▼
             Email
```

---

## 🔐 Security Analysis

SecureFileGuard uses multiple indicators to calculate a threat score.

### Detection mechanisms

* **SHA-256 hashing** for file identification
* **File extension analysis**
* **Magic-byte/signature detection**
* **File type identification**
* **Entropy analysis**
* **Threat scoring**
* **Suspicious indicator tracking**

### Threat scoring

| Detection                              | Score |
| -------------------------------------- | ----: |
| Suspicious executable/script extension |   +40 |
| PE/ELF executable signature            |   +50 |
| High entropy ≥ 7.5                     |   +20 |

Classification:

| Score | Result      |
| ----: | ----------- |
|  0–39 | CLEAN       |
| 40–69 | SUSPICIOUS  |
|   70+ | QUARANTINED |

---

#  Architecture

<img width="1408" height="768" alt="01-architecture" src="https://github.com/user-attachments/assets/97b1932f-f666-4efa-a8e9-59935826b386" />

The platform uses an event-driven serverless architecture to separate file uploading, processing, storage, detection, and alerting.

### Main components

* **Amazon API Gateway** — exposes the upload API
* **AWS Lambda** — handles uploads and security scanning
* **Amazon S3** — stores uploaded, clean, and quarantined files
* **Amazon SQS** — decouples S3 events from the scanner
* **Amazon SQS DLQ** — handles repeated processing failures
* **Amazon DynamoDB** — stores scan results and audit information
* **Amazon SNS** — sends security alerts
* **Amazon CloudWatch** — collects Lambda logs
* **AWS IAM** — controls least-privilege permissions
* **Terraform** — provisions the infrastructure

---

#  AWS Infrastructure

## Amazon S3

The S3 bucket is configured with:

* Block Public Access
* Versioning
* Server-side encryption using SSE-S3
* Lifecycle management
* Separate storage prefixes

```text
securefileguard-216453078762/
├── uploads/
├── clean/
└── quarantine/
```

<img width="1900" height="611" alt="02-s3-bucket" src="https://github.com/user-attachments/assets/e93ffa34-915c-4ae9-9a17-c7190108a690" />
<img width="1892" height="335" alt="03-s3-encryption" src="https://github.com/user-attachments/assets/a05d7197-0bcf-4ca9-a846-10999090edba" />
<img width="1881" height="713" alt="03-s3-public-access" src="https://github.com/user-attachments/assets/703294b1-c669-4ec7-b9cf-fd7ecfa880aa" />
<img width="1887" height="364" alt="04-s3-lifecycle" src="https://github.com/user-attachments/assets/079efbcb-8771-4867-8447-95eff18cd333" />

---

## AWS Lambda

Two Lambda functions are used:

### Upload Lambda

Responsible for:

* Validating file names
* Preventing path traversal
* Enforcing the 5 MB upload limit
* Generating UUID-based object names
* Uploading files to S3

### Scanner Lambda

Responsible for:

* Reading uploaded files
* Calculating SHA-256
* Detecting file signatures
* Calculating entropy
* Generating threat scores
* Moving files to `clean/` or `quarantine/`
* Recording results in DynamoDB
* Sending SNS alerts

<img width="1910" height="387" alt="05-lambda-functions" src="https://github.com/user-attachments/assets/a27c116e-fbbe-438f-8556-b114301d8c8d" />

<img width="1888" height="704" alt="06-scanner-lambda-config" src="https://github.com/user-attachments/assets/003f845b-0a59-4b56-bec7-4f3197ce26af" />

<img width="1583" height="326" alt="07-lambda-environment" src="https://github.com/user-attachments/assets/39267038-4633-49fa-99b5-f3b663387483" />

---

#  Upload API

SecureFileGuard exposes an HTTP API through Amazon API Gateway.

```text
POST /upload
```

Example request:

```json
{
  "file_name": "security-report.txt",
  "content": "Security validation test"
}
```

The API returns a queued response after successfully storing the file.

<img width="1901" height="525" alt="08-api-gateway" src="https://github.com/user-attachments/assets/e50b6d55-22b5-4822-9f39-a5ac5ed1a0d8" />

<img width="1597" height="377" alt="09-api-upload-success" src="https://github.com/user-attachments/assets/b5bee4ab-e004-4653-b19c-deb09ad0e9c7" />

---

#  File Security Scanning

When a file is uploaded to:

```text
S3 → uploads/
```

S3 generates an event that sends a message to SQS.
The Scanner Lambda processes the message and analyzes the file.

### Example clean result

```text
File: security-report.txt
Status: CLEAN
Threat Score: 0
```

<img width="1871" height="818" alt="10-clean-file-detection" src="https://github.com/user-attachments/assets/928e1309-c1ec-4f2b-8af3-b49bf7af43ea" />

Clean files are stored under:


# 🚨 Threat Detection & Quarantine

For suspicious files, SecureFileGuard evaluates multiple indicators.

Example:

```text
File: uuid-test.exe
Extension: .exe
Detected Type: PE/Windows executable
Threat Score: 90
Status: QUARANTINED
```

<img width="1877" height="827" alt="11-threat-detection" src="https://github.com/user-attachments/assets/5e77161e-82e6-4f40-9f8b-b64163c097b0" />

The suspicious file is copied to:

```text
quarantine/
```

<img width="1854" height="570" alt="12-quarantine" src="https://github.com/user-attachments/assets/f9a0f346-fc12-4880-994e-68102aa764f1" />

Detailed analysis information is stored for auditing.

<img width="1907" height="556" alt="13-clean-files" src="https://github.com/user-attachments/assets/a68428cc-bd86-480e-b3d6-64ea1a21d902" />

---

# 🗄️ DynamoDB Audit Records

Every processed file generates a scan record in:

UUID-based scan IDs prevent duplicate files from overwriting previous scan records.

---

# 📧 Security Alerts

When a file is classified as `SUSPICIOUS` or `QUARANTINED`, the Scanner Lambda publishes an alert to Amazon SNS.

<img width="1906" height="606" alt="16-sns-security-alerts" src="https://github.com/user-attachments/assets/665407d7-1740-43c9-b6f2-ea4386e34356" />

The alert is delivered through the confirmed SNS email subscription.

<img width="1479" height="415" alt="17-security-alert-email" src="https://github.com/user-attachments/assets/d599fee6-e2d5-4710-91df-f0d86c7d1139" />

---

#  Reliability with SQS & DLQ

Amazon SQS is used to decouple S3 events from the Scanner Lambda.

The security queue provides:

* Asynchronous processing
* Retry handling
* Visibility timeout
* Message retention
* Dead-Letter Queue support

Configuration:

```text
Security Queue
├── Visibility Timeout: 720 seconds
├── Message Retention: 4 days
└── Maximum Receive Count: 3
        │
        ▼
Security DLQ
└── Message Retention: 14 days
```

<img width="1876" height="834" alt="18-sqs-dlq" src="https://github.com/user-attachments/assets/7fd72943-81cc-4c49-b6d9-7689dbd91653" />

A real failure scenario was tested during development to verify that repeatedly failed messages could reach the DLQ.

---

#  Monitoring

AWS CloudWatch Logs are used to monitor both Lambda functions.

The scanner logs include:

* Scanner execution
* Processed file
* Threat status
* Threat score
* SHA-256 hash
* Processing errors

Log retention is configured for **7 days**.

<img width="1684" height="366" alt="21-cloudwatch-retention" src="https://github.com/user-attachments/assets/e5979bea-f239-4800-8a79-98e24c7e862f" />

---

#  IAM Least Privilege

SecureFileGuard follows the principle of least privilege.
The Scanner Lambda does not receive unrestricted S3 permissions.

It also receives only the AWS permissions required for:

* SQS message processing
* DynamoDB scan-result storage
* SNS alert publishing
* CloudWatch logging

<img width="1878" height="837" alt="22-iam-least-privilege" src="https://github.com/user-attachments/assets/fa418035-7762-4eb4-ab29-0d020c4f4377" />

---

#  Infrastructure as Code

All major AWS infrastructure is managed using Terraform.

Terraform provisions:

* S3
* Lambda
* API Gateway
* SQS
* SQS DLQ
* DynamoDB
* SNS
* IAM
* CloudWatch log groups

#  Input Validation

## Path Traversal Protection

The Upload Lambda rejects unsafe filenames such as:

## File Size Protection

Uploads larger than **5 MB** are rejected.

```text
Maximum file size = 5 MB
```

<img width="1597" height="441" alt="25-file-size-validation" src="https://github.com/user-attachments/assets/a74de511-cc46-4924-84b2-1f6bab923da8" />

---

#  Testing

The project was tested across multiple security and infrastructure scenarios.

| Test                           | Result |
| ------------------------------ | ------ |
| Clean file detection           | ✅ PASS |
| Executable signature detection | ✅ PASS |
| Suspicious extension detection | ✅ PASS |
| Threat scoring                 | ✅ PASS |
| File quarantine                | ✅ PASS |
| DynamoDB scan recording        | ✅ PASS |
| SNS security alert             | ✅ PASS |
| API upload                     | ✅ PASS |
| UUID-based scan IDs            | ✅ PASS |
| Path traversal protection      | ✅ PASS |
| 5 MB size validation           | ✅ PASS |
| IAM least privilege            | ✅ PASS |
| SQS retry handling             | ✅ PASS |
| Dead-Letter Queue              | ✅ PASS |
| CloudWatch logging             | ✅ PASS |
| CloudWatch retention           | ✅ PASS |

---

#  Project Structure

```text
SecureFileGuard/
│
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── iam.tf
│   ├── s3.tf
│   ├── sqs.tf
│   ├── dynamodb.tf
│   ├── sns.tf
│   ├── lambda.tf
│   └── api_gateway.tf
│
├── lambda/
│   ├── upload/
│   │   └── lambda_function.py
│   │
│   └── scanner/
│       └── lambda_function.py
│
├── test/
│   ├── clean.txt
│   ├── synthetic.exe
│   └── alert-test.exe
│
├── docs/
│   └── screenshots/
│
└── README.md
```

---


#  Key Skills Demonstrated

**Cloud Security**

* IAM least privilege
* S3 security
* Encryption
* Input validation
* Threat detection
* Quarantine workflows

**AWS**

* Lambda
* S3
* API Gateway
* SQS
* DynamoDB
* SNS
* CloudWatch
* IAM

**DevOps / Infrastructure**

* Terraform
* Infrastructure as Code
* Event-driven architecture
* Monitoring
* Failure handling
* Retry and DLQ design

**Security Engineering**

* SHA-256 hashing
* Magic-byte analysis
* Entropy analysis
* Threat scoring
* Security alerting
* Audit logging

---

# Author
Udeesha Jayendra
