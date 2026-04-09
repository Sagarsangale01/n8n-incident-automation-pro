# 🚀 Step-by-Step Testing Guide

This guide provides exact commands and expected results to verify the production-grade features of the Incident Workflow.

## 🧹 Initial Cleanup
Reset your local environment by deleting only the temporary log files:
```powershell
Remove-Item submission/processed_ids.log -ErrorAction SilentlyContinue
Remove-Item submission/failures.json -ErrorAction SilentlyContinue
```

---

## 🟢 Test Case 1: Standard Success (Clean Run)
**Scenario:** A new incident is received, normalized, and both notifications are sent immediately.

1.  **Reset Mocks** (Ensure no failures are injected):
    ```powershell
    $env:SLACK_FAIL_429_N=0
    $env:MS_FAIL_500_N=0
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Run the following command:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{
           \"incidentId\": \"INC-2026\",
           \"severity\": \"P1\",
           \"title\": \"Production Check\",
           \"description\": \"Systems are nominal\",
           \"createdAt\": \"2026-02-25T17:20:00Z\",
           \"ownerEmail\": \"hiring-manager@example.com\"
         }'
    ```
4.  **Expected Result:** Every node in n8n turns **Green** instantly.

---

## 🆔 Test Case 2: Deduplication (Idempotency)
**Scenario:** Re-sending the same incident should not trigger notifications.

1.  Click **Execute Workflow** in n8n.
2.  Run the **exact same command** from Test Case 1 again.
3.  **Expected Result:** Flow stops at the **Not Duplicate?** node. The **Acknowledge Duplicate** node turns Green. Notifications are skipped.

---

## 🔄 Test Case 3: Slack Resilience (Rate Limiting)
**Scenario:** The Slack API returns a 429 error, and the workflow waits and retries.

1.  Restart the mock server with failure injection:
    ```powershell
    $env:SLACK_FAIL_429_N=2
    $env:MS_FAIL_500_N=0
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Send a **New Incident ID**:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{ \"incidentId\": \"INC-SLACK-RETRY\", \"severity\": \"P2\", \"title\": \"Slack Retry Test\", \"createdAt\": \"2026-02-25T17:30:00Z\" }'
    ```
4.  **Expected Result:** The workflow loops through **Backoff Wait** twice. The Slack node finally turns Green on the 3rd attempt.

---

## 📧 Test Case 4: Email Resilience (Server Errors)
**Scenario:** The Microsoft Email API returns a 500 error, and the workflow waits and retries.

1.  Restart the mock server with email failure injection:
    ```powershell
    $env:SLACK_FAIL_429_N=0
    $env:MS_FAIL_500_N=1
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Send a **New Incident ID**:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{ \"incidentId\": \"INC-EMAIL-RETRY\", \"severity\": \"P2\", \"title\": \"Email Retry Test\", \"createdAt\": \"2026-02-25T17:35:00Z\" }'
    ```
4.  **Expected Result:** The workflow survives the 500 error, waits, and succeeds on the 2nd attempt.

---

## 🔴 Test Case 5: Final Failure & Logging
**Scenario:** The API fails consistently (500), and the workflow logs the failure for auditing.

1.  Restart the mock server with 10 failures:
    ```powershell
    $env:SLACK_FAIL_500_N=10
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Send a **New Incident ID**:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{ \"incidentId\": \"INC-FATAL-01\", \"severity\": \"P3\", \"title\": \"Fatal Error Test\", \"createdAt\": \"2026-02-25T17:40:00Z\" }'
    ```
4.  **Expected Result:** After 5 failed retry loops, the **Persist Failure** node logs the details to `submission/failures.json`.
