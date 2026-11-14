# Invoicely Lite – Product Support Engineer Assignment

## Submission details

- Name: Madhumitha
- Date submitted: 14 November 2026
- Repository link: https://github.com/madhumitharaha/pse-support-lab-submission
- Total time spent: Approximately 2 hours

All customer communications below are drafts. No external emails were sent.

## 1. Ticket triage and acknowledgement drafts

### 1. Oasis Builders – Upload stuck
- Received: 16 Nov 2026, 09:05 GST
- Priority: P2
- Reason: Repeated uploads, duplicate warnings and month-end impact.
- Proposed acknowledgement: 09:10 GST

Hello Rashid,

We understand that repeated upload attempts have left you uncertain about which invoices were processed. Please do not upload the same workbook again until we reconcile the invoice statuses. We are checking the import and invoice-level records and will confirm which documents were submitted and which need further action.

Regards,
Invoicely Lite Support

### 2. API restart alert / Priya Nair
- Received: 16 Nov 2026, 09:42 GST; Priya forward at 09:50 GST
- Priority: P1
- Reason: Critical API restart alert, intermittent 502 reports and invoices reportedly stuck in Validated. Marina Bay go-live is on 18 November.
- Proposed acknowledgement: 09:55 GST

Hi Priya,

I have taken ownership of this incident. We are investigating the API restart alert alongside the reported 502 responses and invoices stalled in Validated. The API is currently healthy in our lab checks, but the exact cause of the reported restarts is not confirmed. We will continue checking processing status and service health before confirming go-live readiness.

Regards,
Support On-call

### 3. Al Noor Fuel – BR-CO-15
- Received: 16 Nov 2026, 10:15 GST
- Priority: P2
- Reason: Three rejected invoices and an active VAT audit review.
- Proposed acknowledgement: 10:20 GST

Dear Fatima,

We reviewed the three rejected invoices. The ASP response reports a difference between the inclusive total and the sum of the exclusive total and VAT. Our investigation found differences of AED 0.01, AED 0.02 and AED 0.01. The portal validation passed, but the ASP rejected the invoices.

Please retain the original workbook and avoid manually changing quantities or prices. We recommend technical reconciliation before resubmission.

Kind regards,
Invoicely Lite Support

### 4. Bluepeak ERP / Sahara Medical
- Received: 16 Nov 2026, 11:30 GST
- Priority: P2
- Reason: Buyer reports missing invoices; 40 invoices were in the connector batch and resending may create duplicates.
- Proposed acknowledgement: 11:35 GST

Hi Daniel,

HTTP 201 confirms that the import request was accepted by Invoicely; it does not confirm that every invoice reached the buyer. Our controlled lab replay showed 8 Accepted and 32 still Submitted without ASP message references. These outstanding documents need reconciliation.

Please hold off on resending the batch until the invoice statuses and ASP references are checked.

Regards,
Invoicely Lite Support

### 5. Ghaf Tree Café – Date format
- Received: 16 Nov 2026, 12:10 GST
- Priority: P3
- Reason: Four invoices rejected due to a correctable date-format issue.
- Proposed acknowledgement: 12:15 GST

Hello,

We identified that the invoice issue dates and payment due dates use DD/MM/YYYY, while the portal expects YYYY-MM-DD. Please convert both date columns to the required format, validate the corrected workbook and then upload it again. For example, 13/11/2026 should be 2026-11-13.

Kind regards,
Invoicely Lite Support

Note: The inbox does not specify formal SLA targets. Proposed acknowledgement times above are draft exercise-clock times, not actual sent times.

## 2. Detailed investigations

### Investigation A – Al Noor Fuel: BR-CO-15

Affected invoices:

| Invoice | Actual inclusive | Expected inclusive | Difference |
|---|---:|---:|---:|
| ANF-2611-0042 | AED 15,022.42 | AED 15,022.43 | AED 0.01 |
| ANF-2611-0044 | AED 11,817.57 | AED 11,817.59 | AED 0.02 |
| ANF-2611-0046 | AED 5,610.99 | AED 5,611.00 | AED 0.01 |

Status history: Parsed → Validated → Submitted → Rejected.

Database records confirm rejected_by = asp.

Code findings:
- services/api/internal/pint/pint.go, lines 157–185: net, VAT and gross totals are aggregated and rounded.
- services/asp-mock/main.go, lines 225–229: ASP compares inclusive total with exclusive total plus VAT.
- services/asp-mock/main.go, line 188: cents conversion uses math.Round.

Finding:
The API's gross calculation and the ASP's exclusive-plus-VAT comparison use different calculation paths. The resulting rounding differences explain the BR-CO-15 rejections observed in this lab.

Customer action:
Retain the original workbook. Do not manually change quantities or prices. Reconcile the calculation before resubmitting the three rejected invoices.

Evidence:
- evidence/al-noor-code-evidence.txt
- evidence/al-noor-db-proof.txt
- evidence/al-noor-rejection-logs.txt

### Investigation B – Oasis Builders: upload and duplicate warnings

Historical database records showed an October upload with 8 invoices. The November customer complaint's original invoice-level outcomes were not established from those historical records.

Code finding:
services/api/internal/validate/validate.go, lines 73–80 checks whether the invoice number already exists for the same seller. If it exists, AE-BR-09 is returned.

Controlled replay:
The load generator adds a unique run tag to generated invoice numbers. The controlled Oasis replay completed with 8 Accepted invoices.

Finding and limitation:
The replay confirms that a newly generated Oasis batch can be accepted. It does not prove what happened during the customer's original repeated upload attempts. The original upload's complete status reconciliation remains open.

Customer action:
Do not upload the same batch again until the original invoice statuses are reconciled. Do not manually change invoice numbers.

Evidence:
- evidence/oasis-imports.txt
- evidence/oasis-invoice-status.txt
- evidence/oasis-replay-result.txt
- evidence/oasis-replay-db-proof.txt
- evidence/duplicate-validation-code.txt
- evidence/oasis-investigation-summary.txt

### Investigation C – API restart and peak traffic

The controlled lab load test attempted 240 peak uploads:
- 209 successful
- 31 failed
- 24 EOF errors
- 1 connection refused
- 6 DNS lookup failures

The load summary reported 213 Completed imports and 24 Processing imports. A database snapshot showed 24 Processing imports across four sellers.

API container checks:
- RestartCount: 3
- OOMKilled: false
- ExitCode: 0
- Current health: healthy
- Health-check failing streak: 0

Finding:
The lab reproduced upload failures during peak traffic and showed three API restarts. OOM termination was not indicated. The exact restart cause was not established. The lab load-test timestamps are separate from the exercise inbox timestamps, so they do not prove the cause of the reported 16 November incident.

Next action:
Engineering should correlate the reported incident window with API, proxy and infrastructure logs, and reconcile the invoices remaining in Processing. Continue monitoring before confirming Marina Bay go-live readiness.

Evidence:
- evidence/full-traffic-replay.txt
- evidence/post-load-service-health.txt
- evidence/processing-imports-after-load.txt
- evidence/processing-imports-summary.txt
- evidence/api-restart-investigation.txt
- evidence/api-restart-timeline.txt
- evidence/api-startup-health-details.txt

### Investigation D – Bluepeak / Sahara Medical

Controlled replay produced 40 invoices:
- 8 Accepted with ASP message IDs
- 32 Submitted without ASP message IDs
- 0 Rejected

Finding:
HTTP 201 indicates successful import hand-off, not confirmation that every invoice reached the buyer. The 32 outstanding Submitted invoices need ASP status reconciliation. The controlled replay does not establish the original customer batch's exact delivery outcome.

Customer action:
Do not resend until outstanding invoice references and ASP statuses are reconciled.

Evidence:
- evidence/sahara-invoice-status.txt
- evidence/bluepeak-sahara-investigation-summary.txt

### Investigation E – Ghaf Tree Café

Database evidence showed four rejected invoices:
- GTC-118
- GTC-119
- GTC-120
- GTC-121

All were rejected by portal validation with AE-BR-04 because issue dates and due dates used DD/MM/YYYY instead of YYYY-MM-DD.

Workaround:
Correct both date columns, validate the corrected workbook, and upload it again.

Evidence:
- evidence/ghaf-invoice-proof.txt
- evidence/ghaf-investigation-summary.txt

## 3. AI and tools disclosure

ChatGPT assisted with investigation, interpretation and drafting. Findings were checked against the local lab portal, source code, logs and PostgreSQL records.

Tools used:
- Docker Desktop and Docker Compose
- Local Invoicely Lite portal and Grafana
- Terminal and Go source inspection
- PostgreSQL and SQL queries
- Local API and ASP mock logs

No real customer emails were sent and no external company or ASP was contacted. All communications are proposed drafts.

## 4. Limitations and open questions

- Exact cause of the three API restarts remains undetermined.
- Oasis original November upload outcomes were not fully established.
- Bluepeak's original outstanding invoices require ASP reconciliation.
- No production fix was applied to the lab source code.
- Controlled replay results are not treated as proof of original customer outcomes.
