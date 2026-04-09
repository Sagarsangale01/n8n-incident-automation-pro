# 🛡️ n8n Incident Notification Workflow - Production Grade

![n8n](https://img.shields.io/badge/n8n-v1.0+-FF6C37?logo=n8n&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue.svg)
![Reliability](https://img.shields.io/badge/Reliability-Production--Grade-green)

A professional, offline-first incident notification system built with n8n. This project demonstrates enterprise-level reliability patterns including data normalization, stateful idempotency, custom retry loops, and fail-safe logging.

---

## 📑 Table of Contents
* [Core Features](#-core-features)
* [Workflow Architecture](#-workflow-architecture)
* [Technical Documentation](#-technical-documentation)
* [Step-by-Step Testing Guide](#-step-by-step-testing-guide)
* [Getting Started](#-getting-started)
* [Verification & Test Cases](#-verification--test-cases)
* [Project Structure](#-project-structure)

---

## ✨ Core Features
* **💎 Robust Normalization**: JS-heavy validation and severity mapping.
* **🆔 Stateful Idempotency**: Filesystem-based deduplication that works in all n8n modes.
* **🔄 Advanced Retries**: Manual backoff loops with a 5-attempt circuit breaker.
* **📊 Audit Logging**: Persistent local failure auditing in JSON format.
* **🔌 Offline First**: Designed to run entirely against local mock APIs.

---

## 🏗️ Workflow Architecture

The workflow follows a linear-but-resilient pipeline designed for high availability:

### 1. Data Integrity (Validation)
Incoming payloads are validated in the **Validate & Normalize** node. It ensures all mandatory fields exist and maps severity levels (P1-P4) to numeric values.

### 2. Idempotency (Deduplication)
- **Check**: Before notifying, it checks `submission/processed_ids.log`.
- **Action**: If a duplicate is found, it terminates at the **Acknowledge Duplicate** node.

### 3. Reliability (Native Retries)
The workflow utilizes **n8n Native Built-in Retries** for both Slack and O365:
- **Resilience**: Automatically retries 5 times with a 2-second backoff.
- **Circuit Breaker**: Stops execution and logs to `failures.json` if all retries fail.
- **Cleanliness**: Keeps the canvas clean and readable without extra manual loop nodes.

---

## 📖 Technical Documentation

For a deep dive into the node-level design and implementation logic, please refer to:
* 📘 **[Technical Documentation](submission/TECHNICAL_DOCUMENTATION.md)**: Architecture diagrams and node breakdown.
* 📝 **[Implementation Notes](submission/NOTES.md)**: Dedupe formulas and retry settings.

---

## 🚀 Step-by-Step Testing Guide

To verify the "Production Grade" features of this workflow (Normalization, Deduplication, Retries, and Error Logging), please follow our:
* 🧪 **[Detailed Testing Guide](submission/TESTING_GUIDE.md)**

---

## 🛠️ Getting Started

### 1. Prerequisites
- **Node.js**: v18 or later.
- **n8n**: Local installation (Desktop or CLI).

### 2. Installation & Mocks
```powershell
# Install dependencies
npm install

# Start the offline mocks (in a separate terminal)
# Optional: Set failure injection
$env:SLACK_FAIL_429_N=2
npm run mocks
```

### 3. Workflow Import
- Export your n8n workflow as a JSON file or use the provided **[workflow.json](submission/workflow.json)**.
- Import it into n8n and set the Webhook path to `incident`.

---

## 🧪 Verification & Test Cases

| Case | Scenario | Mock Injection ($env) | Expected Result |
| :--- | :--- | :--- | :--- |
| **01** | **Success** | `SLACK_FAIL_429_N=0`, `MS_FAIL_500_N=0` | All nodes Redundant Green. |
| **02** | **Dedupe** | (Use same data as Case 01) | Stops at **Acknowledge Duplicate**. |
| **03** | **Slack Retry** | `SLACK_FAIL_429_N=2` | Loops through **Backoff Wait** twice. |
| **04** | **Email Retry** | `MS_FAIL_500_N=1` | Survives 500 error & succeeds on 2nd try. |
| **05** | **Failure** | `SLACK_FAIL_500_N=10` | After 5 retries, logs to **failures.json**. |

---

## 📂 Project Structure
```text
├── submission/
│   ├── workflow.json           # Main n8n Production Workflow
│   ├── TECHNICAL_DOCUMENTATION.md # Design & Node architecture
│   ├── NOTES.md                # Implementation & Dedupe logic
│   └── failures.json           # Local failure audit log
├── fixtures/                   # Sample incident payloads
├── mocks/                      # Offline API mock servers
└── README.md                   # Project landing page
```

---

## ✒️ Author Notes
This implementation prioritizes **offline safety**. We use filesystem-based state checks (`processed_ids.log`) to ensure the deduplication logic remains functional even in n8n's "Test Mode" where internal memory is reset between runs.
