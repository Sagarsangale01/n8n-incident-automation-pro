# 🚀 Step-by-Step Testing Guide

This guide provides exact commands and expected results to verify the production-grade features of the Incident Workflow.

## 🧹 Initial Cleanup
Reset your local environment to ensure a clean test state:
```powershell
Remove-Item submission/*.log -ErrorAction SilentlyContinue
Remove-Item submission/*.json -ErrorAction SilentlyContinue
```

---

## 🟢 Test Case 1: Standard Success
**Scenario:** A new incident is received, normalized, and both notifications are sent.

1.  Click **Execute Workflow** in n8n.
2.  Run the following command:
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
3.  **Expected Result:** Every node in n8n turns **Green**. Both Slack and Email mocks receive the normalized data.

---

## 🆔 Test Case 2: Deduplication (Idempotency)
**Scenario:** Re-sending the same incident should not trigger notifications.

1.  Click **Execute Workflow** in n8n.
2.  Run the **exact same command** from Test Case 1 again.
3.  **Expected Result:** Flow stops at the **Not Duplicate?** node. The **Acknowledge Duplicate** node turns Green. Notifications are skipped.

---

## 🔄 Test Case 3: Advanced Retries (Rate Limiting)
**Scenario:** The Slack API is "busy" (returns 429), and the workflow must wait and retry.

1.  Restart the mock server with failure injection:
    ```powershell
    $env:SLACK_FAIL_429_N=2
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Send a **New Incident ID**:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{
           \"incidentId\": \"INC-RETRY-01\",
           \"severity\": \"P2\",
           \"title\": \"Retry Simulation\",
           \"description\": \"Testing 429 backoff\",
           \"createdAt\": \"2026-02-25T17:30:00Z\"
         }'
    ```
4.  **Expected Result:** The workflow loops through **Backoff Wait** twice. The Slack node finally turns Green on the 3rd attempt.

---

## 🔴 Test Case 4: Persistent Failure & Logging
**Scenario:** The API fails consistently (500), and the workflow logs the failure for auditing.

1.  Restart the mock server with 10 failures (exceeding our 5 retries):
    ```powershell
    $env:SLACK_FAIL_500_N=10
    npm run mocks
    ```
2.  Click **Execute Workflow** in n8n.
3.  Send a **New Incident ID**:
    ```powershell
    curl.exe -X POST http://localhost:5678/webhook-test/incident `
         -H "Content-Type: application/json" `
         -d '{ \"incidentId\": \"INC-FAIL-01\", \"severity\": \"P3\", \"title\": \"Fatal Error Test\", \"createdAt\": \"2026-02-25T17:40:00Z\" }'
    ```
4.  **Expected Result:** After 5 failed retry loops, the **Persist Failure** node turns Green.
5.  **Verify:** Open `submission/failures.json` to see the logged error details.
