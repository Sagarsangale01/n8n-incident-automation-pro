# Submission Notes - Incident Workflow

## 1. How to Run the Workflow

1.  **Environment Setup**:
    - Ensure Node.js 18+ is installed.
    - Run `npm install` in the root directory.
    - Start the mock APIs: `npm run mocks`.
2.  **n8n Import**:
    - Import `submission/workflow.json` into your n8n instance.
3.  **Triggering**:
    - Use the Webhook URL (POST /incident).
    - To test with sample data:
      ```powershell
      npm run demo -- fixtures/incidents/INC-10001.json
      ```

---

## 2. Implementation Details

### Normalization & Validation

- **Node**: `Validate & Normalize` (Code Node)
- **Validation**: Enforces strict existence of `incidentId`, `severity`, `title`, and `createdAt`. Throws a detailed error if any field is missing.
- **Severity Mapping**: Standardizes `P1-P4` to numeric levels `1-4`.
- **Truncation**: Incident descriptions are truncated to 240 chars for clean notification delivery.

### Reliability & Retries

- **Mechanism**: **Dual Manual Loop Lanes** (Enterprise Resilience Pattern).
- **Nodes**: Separate visual loops for `Slack Notify` and `Send Email` for maximum transparency.
- **Triggers**: Only retries on **429** (Rate Limit) and **5xx** (Server Error).
- **Settings**:
  - **Max Attempts**: 5 per service.
  - **Wait/Backoff**: Exponential backoff via `Wait` node (`2s`, `4s`, `8s`, `16s`, capped at `30s`).
- **Resilience**: Every file-based node includes **Self-Healing directory creation** to prevent path errors.

### Idempotency & Deduplication

- **DedupeKey Formula**: `incidentId:severity:createdAt`
- **Mechanism**:
  - **Check**: The `Check Deduplication` node verifies the key against `submission/processed_ids.log`.
  - **Auto-Provisioning**: Automatically creates the `submission/` folder if missing.
  - **Mark**: The `Mark Processed` node only appends the key **after** successful email delivery.
- **Why**: This filesystem-based state ensures idempotency works even in n8n's "Test Mode" where internal memory is reset.

### Error Handling & Persistence

- **Routing**: If notifications fail after all 5 attempts, the flow takes the failure branch.
- **Persistence**: Details (Incident ID, error, timestamp) are appended to a JSON audit log at **`submission/failures.json`**.
- **Portability**: File operations use Node.js commands in `Execute Command` nodes (cross-platform), not PowerShell-specific syntax.

### Email Recipient Rule

- **Recipient Source**: Email is sent to `ownerEmail` from the incident payload (no fallback address).

---

## 3. Compliance Checklist

- [x] JS-heavy Normalization & Validation.
- [x] Severity Mapping (P1->1, etc.).
- [x] Dedupe formula documented.
- [x] Retries on 429/5xx only (Max 5).
- [x] Graph-like Email body structure.
- [x] Local failure persistence.
- [x] Fully Offline (using mocks).
