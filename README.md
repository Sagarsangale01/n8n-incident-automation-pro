# n8n Incident Notification Workflow - Production Grade

![n8n](https://img.shields.io/badge/n8n-v1.0+-FF6C37?logo=n8n&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue.svg)
![Reliability](https://img.shields.io/badge/Reliability-Production--Grade-green)

A robust, offline-first incident notification system built with n8n. This project demonstrates enterprise-level reliability patterns including data normalization, idempotency, custom retry loops, and fail-safe logging.

---

## 🏗️ Workflow Architecture

The workflow follows a linear-but-resilient pipeline designed for high availability:

### 1. Data Integrity (Validation)
Incoming payloads are validated in the **Validate & Normalize** node. It ensures all mandatory fields (`incidentId`, `severity`, etc.) exist and maps severity levels (P1-P4) to numeric values for downstream logic.

### 2. Idempotency (Deduplication)
The workflow implements a **Post-Success Marking** strategy.
- **Check**: Before notifying, it checks `submission/processed_ids.log`.
- **Action**: If a duplicate is found, it terminates at the **Acknowledge Duplicate** node.
- **Mark**: The incident is only added to the log **after** successful delivery to both Slack and Email.

### 3. Reliability (Retry Loop)
Instead of relying on simple "Retry on Failure" toggles, this workflow uses a **Manual Backoff Loop**:
- **Triggers**: Only retries on **429** (Rate Limit) and **5xx** (Server Error).
- **Control**: An **Increment Retry** node tracks attempts.
- **Circuit Breaker**: Stops after **5 attempts** to prevent infinite execution.
- **Backoff**: Incorporates a 2-second wait between attempts.

### 4. Observability (Audit Trail)
If a delivery fails after all retries, the incident is routed to the **Persist Failure** node, which appends a detailed error record to `submission/failures.json`.

---

## 🛠️ Getting Started

### 1. Prerequisites
- **Node.js**: v18 or later.
- **n8n**: Any local installation.
- **Terminal**: PowerShell is recommended for the filesystem state checks.

### 2. Installation
```powershell
npm install
```

### 3. Running the Mocks
In a dedicated terminal:
```powershell
# Optional: Set failure injection
$env:SLACK_FAIL_429_N=2
$env:MS_FAIL_500_N=1

npm run mocks
```

---

## 📂 Project Structure

```text
├── submission/
│   ├── workflow.json           # Main n8n Production Workflow
│   ├── TECHNICAL_DOCUMENTATION.md # Design & Node architecture
│   ├── NOTES.md                # Implementation & Dedupe logic
│   └── failures.json           # Local failure audit log (auto-generated)
├── fixtures/                   # Sample incident payloads
├── mocks/                      # Offline API mock servers
└── README.md                   # Project landing page
```

---

## 🧪 Verification & Test Cases

| Case | Scenario | Command | Expected Result |
| :--- | :--- | :--- | :--- |
| **01** | **Standard** | Use `curl.exe` with a new `incidentId`. | All nodes finish Green. |
| **02** | **Dedupe** | Run the **same** command twice. | 2nd run stops at **Acknowledge Duplicate**. |
| **03** | **Retry** | Set `SLACK_FAIL_429_N=2`. | Node loops through **Backoff Wait** twice before success. |
| **04** | **Failure** | Set `SLACK_FAIL_500_N=10`. | After 5 retries, flow hits **Persist Failure**. |

---

## ✒️ Author Notes
This implementation prioritizes **offline safety**. We use filesystem-based state checks (`processed_ids.log`) to ensure the deduplication logic remains functional even in n8n's "Test Mode" where internal memory is reset between runs.
