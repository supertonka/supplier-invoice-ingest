# 📦 Supplier Invoice Ingest Pipeline

### n8n Automation · Supabase (PostgreSQL) · Africa/Johannesburg

> An end-to-end automated pipeline that ingests supplier invoices via **Webhook** or **Google Drive**, validates and normalises each row against strict business rules, deduplicates against a live database, persists clean records to Supabase, and dispatches a formatted HTML email alert — all without human intervention. A separate scheduled retry pipeline automatically re-attempts failed invoices and escalates exhausted cases.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Workflows](#workflows)
3. [Node-by-Node Breakdown — Main Workflow](#node-by-node-breakdown--main-workflow)
4. [Node-by-Node Breakdown — Retry Workflow](#node-by-node-breakdown--retry-workflow)
5. [Triggers & How to Run](#triggers--how-to-run)
6. [CSV Format & Field Mapping](#csv-format--field-mapping)
7. [Database Schema & Setup](#database-schema--setup)
8. [Validation Rules](#validation-rules)
9. [Deduplication Logic](#deduplication-logic)
10. [Binary Field Normalisation](#binary-field-normalisation)
11. [Dry-Run Mode](#dry-run-mode)
12. [Retry Pipeline](#retry-pipeline)
13. [Email Alerts](#email-alerts)
14. [n8n Credentials & Configuration](#n8n-credentials--configuration)
15. [Test Evidence](#test-evidence)
16. [Key Design Decisions](#key-design-decisions)

---

## Architecture Overview

### Main Ingest Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        MAIN INGEST PIPELINE                              │
│                                                                          │
│  [Webhook — Receive CSV Upload] ──┐                                      │
│                                   ├──▶ Respond — Acknowledged            │
│  [Google Drive — Watch Folder]    │                                      │
│  [Google Drive — Download CSV] ───┘                                      │
│                                   │                                      │
│                                   ▼                                      │
│                    ⚙️ Config — Pipeline Settings                          │
│                    (DRY_RUN flag)                                         │
│                                   │                                      │
│                                   ▼                                      │
│                    🔐 Hash File — SHA-256 + Extract                       │
│                    (computes source_hash, extracts sourceFileName)        │
│                                   │                                      │
│                                   ▼                                      │
│                    🔧 Normalize — Binary Field Name                       │
│                    (renames any binary field to 'file')                   │
│                                   │                                      │
│                                   ▼                                      │
│                    📄 Extract from File (Extract From CSV)                │
│                    (converts binary CSV → JSON rows)                      │
│                                   │                                      │
│                                   ▼                                      │
│                    📋 Parse CSV — Map Fields + Fill Math                  │
│                    (field mapping, VAT calculation, row numbering)        │
│                                   │                                      │
│                                   ▼                                      │
│                    🔍 Validate Row — Business Rules                       │
│                    (5 validation rules, errors + warnings)                │
│                                   │                                      │
│                    🔀 Route — Valid vs Invalid                             │
│                         │                  │                             │
│                      valid             invalid                           │
│                         │                  │                             │
│                         ▼                  ▼                             │
│               🔎 Dedup Check        💾 Insert —                          │
│               — Query DB            supplier_invoices_failures            │
│                         │                                                │
│                         ▼                                                │
│               🏷️ Tag — Duplicate or New                                   │
│               (within-batch + cross-run dedup)                           │
│                         │                                                │
│               🔀 Route — Insert or Skip Duplicate                        │
│                    │              │                                      │
│                 insert         duplicate                                 │
│                    │              │                                      │
│                    ▼              ▼                                      │
│            🧪 Route —      📌 Mark Duplicate                             │
│            Live vs Dry Run  — Collect                                    │
│               │       │                                                  │
│            live     dry-run                                              │
│               │       │                                                  │
│               ▼       ▼                                                  │
│    💾 Insert —    💾 Insert —                                            │
│    supplier_      supplier_                                              │
│    invoices       invoices_staging                                       │
│               │                                                          │
│               ▼                                                          │
│    📊 Aggregate — Metrics Summary                                        │
│               │                                                          │
│               ▼                                                          │
│    ✉️ Build — HTML Email Body                                             │
│               │                                                          │
│               ▼                                                          │
│    📧 Send — Ingest Alert Email                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Retry Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        RETRY PIPELINE                                    │
│                                                                          │
│  ⏰ Schedule — Every 6 Hours                                              │
│               │                                                          │
│               ▼                                                          │
│  🔍 Query — Fetch Pending Retries                                        │
│  (SELECT WHERE retry_count <= 3)                                         │
│               │                                                          │
│  🔀 Route — Has Retries or Empty                                         │
│       │              │                                                   │
│    has rows       no rows → end silently                                 │
│       │                                                                  │
│       ▼                                                                  │
│  ⚙️ Prepare — Parse & Validate Retry                                      │
│  (rebuilds record from raw_payload, checks can_retry flag)               │
│               │                                                          │
│  🔀 Route — Retryable or Exhausted                                       │
│       │                    │                                             │
│   retryable            exhausted                                         │
│       │                    │                                             │
│       ▼                    ▼                                             │
│  💾 Re-insert —    📝 Update — Mark Exhausted                            │
│  supplier_invoices                                                       │
│       │       │                                                          │
│   success   error                                                        │
│       │       │                                                          │
│       ▼       ▼                                                          │
│  🗑️ Delete —  📝 Update — Retry                                          │
│  Remove from  Failed, Increment                                          │
│  Failures     Count                                                      │
│               │                                                          │
│               ▼                                                          │
│  📊 Aggregate — Retry Metrics                                            │
│               │                                                          │
│               ▼                                                          │
│  ✉️ Build — Retry Email Body                                              │
│               │                                                          │
│               ▼                                                          │
│  📧 Send — Retry Alert Email                                              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Workflows

| File | Description |
| --- | --- |
| `supplier-ingest.json` | Main ingest pipeline — Webhook + Google Drive triggers, validation, dedup, insert, email |
| `supplier-retry.json` | Scheduled retry pipeline — re-attempts failed invoices every 6 hours |

---

## Node-by-Node Breakdown — Main Workflow (supplier-ingest.json)

<img width="1774" height="922" alt="main ingest workflow + executions" src="https://github.com/user-attachments/assets/a83d8e70-a7b9-41da-8a67-0241ed9741ee" />

Screenshot showing entire '`supplier-ingest.json`' workflow with sticky-notes.

### 📥 Webhook — Receive CSV Upload

- Listens for `POST` requests at `/webhook/supplier-invoice-upload`
- Accepts `multipart/form-data` file uploads (binary CSV)
- Immediately fires **✅ Respond — Acknowledged** in parallel to return a `200` response without waiting for processing to complete

### ✅ Respond — Acknowledged

- Returns `{"status":"received","message":"Invoice file received and queued for processing"}` instantly
- Ensures the HTTP client is not left waiting during the full pipeline execution

### 📁 Google Drive — Watch Folder

- Polls a designated Google Drive folder every minute
- Triggers when a **new file is created** in the watched folder
- Outputs file metadata including `id`, `name`, `mimeType`

### 📥 Google Drive — Download CSV

- Downloads the file detected by the Watch Folder trigger
- Uses `{{ $json.id }}` — the dynamic file ID from the trigger — to always fetch the correct file
- Outputs binary file data for downstream processing

### ⚙️ Config — Pipeline Settings

- Single source of truth for pipeline configuration
- Contains the `DRY_RUN` boolean flag:
  - `false` → inserts into `supplier_invoices` (live/production)
  - `true` → inserts into `supplier_invoices_staging` (safe testing)
- Positioned at the start so the flag propagates through every downstream node via the Hash node

### 🔐 Hash File — SHA-256 + Extract

- Reads binary file data from the **Webhook node** or **Google Drive Download** node as a fallback
- Computes a **SHA-256 hash** of the raw file contents — stored as `source_hash` for idempotency
- Extracts `sourceFileName` from binary metadata
- Reads `DRY_RUN` from Config and attaches the `dry_run` flag to all downstream items
- Passes binary data through for the Extract from File node

### 🔧 Normalize — Binary Field Name

- Solves a cross-source compatibility issue:
  - Webhook binary field = `file`
  - Google Drive binary field = `data`
- Renames whichever binary field is present to always be `file`
- Ensures **Extract from File** always finds the correct field name regardless of trigger source

```javascript
const firstKey = Object.keys(binaryData)[0];
normalizedBinary.file = binaryData[firstKey];
```

### 📄 Extract from File

- n8n native node that converts the binary CSV into structured JSON rows automatically
- Input Binary Field: `file` (normalised by the previous node)
- Outputs one JSON item per CSV row — handles encoding, line endings, and quoted fields natively

### 📋 Parse CSV — Map Fields + Fill Math

- Maps CSV column names to unified schema field names (e.g. `amount_excl` → `amount_excl_vat`)
- Applies VAT math fill logic:
  - If `vat` is missing: `vat = Math.round(amount_excl × vat_rate / 100 × 100) / 100`
  - If `amount_incl_vat` is missing: `amount_incl_vat = Math.round((amount_excl_vat + vat) × 100) / 100`
- Default VAT rate: **15%** (South Africa)
- Attaches `sourceFileName`, `sourceHash`, `dry_run`, and `_rowNum` to every row
- Reads metadata directly from the Hash node: `$('🔐 Hash File — SHA-256 + Extract').first()`

### 🔍 Validate Row — Business Rules

- Runs all 5 business validation rules (see [Validation Rules](#validation-rules))
- Returns `valid: true/false`, `validation_notes`, `_errorCount`, `_warningCount` per row
- Rows with errors → `valid: false` → routed to failures table
- Rows with warnings only → `valid: true` → proceed with warning in `validation_notes`

### 🔀 Route — Valid vs Invalid

- Routes valid rows to the dedup path
- Routes invalid rows to `💾 Insert — supplier_invoices_failures`

### 🔎 Dedup Check — Query DB

- Runs a `SELECT EXISTS(...)` query per row against `supplier_invoices`
- **"Always Output Data"** enabled — returns `{is_duplicate: true/false}` even when no match found
- Runs once per item (not execute-once) to check every row individually

### 🏷️ Tag — Duplicate or New

- Combines DB dedup results with validated row data
- Implements **within-batch deduplication** using a JavaScript `Set`
- Marks each row as `status: 'pending_insert'` or `status: 'duplicate'`
- Attaches `is_duplicate` boolean and appropriate `validation_notes`

### 🔀 Route — Insert or Skip Duplicate

- `pending_insert` rows → Live vs Dry Run router
- `duplicate` rows → **📌 Mark Duplicate — Collect**

### 📌 Mark Duplicate — Collect

- Collects duplicate rows and passes them to **📊 Aggregate — Metrics Summary**
- Ensures duplicates are counted and shown in the email even though they are not inserted

### 🧪 Route — Live vs Dry Run

- Reads `dry_run` from `$json.dry_run`
- `dry_run = false` → **true branch** → `supplier_invoices`
- `dry_run = true` → **false branch** → `supplier_invoices_staging`

### 💾 Insert — supplier_invoices

- Inserts validated, non-duplicate rows into the live table
- Uses `ON CONFLICT (supplier_number, invoice_number) DO NOTHING` as DB-level safety net
- Sets `status = 'inserted'` and `ingest_timestamp = NOW() AT TIME ZONE 'Africa/Johannesburg'`

### 💾 Insert — supplier_invoices_staging

- Identical insert query targeting `supplier_invoices_staging`
- Only active when `DRY_RUN: true`
- Full pipeline runs normally — data goes to staging instead of live

### 💾 Insert — supplier_invoices_failures

- Inserts invalid rows into the failures table
- Stores full row as `raw_payload JSONB` for retry processing
- Sets `retry_count = 0` on initial insert
- Stores full error details in `validation_notes`

### 📊 Aggregate — Metrics Summary

- Reads all items from **🏷️ Tag — Duplicate or New** to count totals
- Guard prevents double email: only blocks duplicate-triggered execution when inserts also fired
- Counts: `totalProcessed`, `insertedCount`, `duplicateCount`, `failedCount`
- Builds `issueRows` array (max 20) for the email issues table
- Computes `ingestTime` in SAST (UTC+2)

### ✉️ Build — HTML Email Body

- Generates fully formatted HTML email with colour-coded status header
- Constructs individual execution URL using `$execution.id` for direct link to the run
- Green (all success), Orange (duplicates/warnings), Red (errors/failures)

### 📧 Send — Ingest Alert Email

- Sends HTML email via Gmail
- Subject: `Supplier Ingest: {inserted} ok, {duplicates} dup, {failed} failed — {filename}`

---

## Node-by-Node Breakdown — Retry Workflow (supplier-retry.json)

<img width="1770" height="925" alt="retry logic workflow + executions" src="https://github.com/user-attachments/assets/243257a5-ba7c-46e3-a10f-c62db6f86fd7" />


Screenshot showing entire '`supplier-retry.json`' workflow with sticky-notes.

### ⏰ Schedule — Every 6 Hours

- Automatically triggers the retry pipeline every 6 hours (production) / Use 1 Minute for testing.
- Can also be triggered manually via "Execute workflow" on the canvas

### 🔍 Query — Fetch Pending Retries

- Fetches rows from `supplier_invoices_failures` where `retry_count <= 3`
- Orders by `created_at ASC` (oldest failures first)
- Limit: 50 rows per run

### 🔀 Route — Has Retries or Empty

- No rows → workflow ends silently (no email)
- Rows exist → proceeds to Prepare node

### ⚙️ Prepare — Parse & Validate Retry

- Parses `raw_payload` JSONB back into individual structured fields
- Merges failure record with parsed payload
- Increments `retry_count` by 1
- Sets `can_retry` flag:

```javascript
const MAX_RETRIES = 3;
const canRetry = !!(record.invoice_number && record.supplier_number &&
                 record.supplier_name && record.department &&
                 record.amount_excl_vat && record.invoice_date) &&
                 record.retry_count <= MAX_RETRIES;
```

### 🔀 Route — Retryable or Exhausted

- `can_retry = true` → attempt re-insert
- `can_retry = false` → mark as exhausted

### 💾 Re-insert — supplier_invoices

- Attempts to insert the failure row back into `supplier_invoices`
- **Success branch** → Delete from Failures
- **Error branch** → Update — Retry Failed, Increment Count

### 🗑️ Delete — Remove from Failures

- Deletes successfully re-inserted row from `supplier_invoices_failures`
- Keeps the failures table clean after successful retries
- Also serves as the trigger for **📊 Aggregate — Retry Metrics** to ensure single email per run

### 📝 Update — Mark Exhausted

- Updates `validation_notes` to `'Max retries (3) reached — manual intervention required'`
- Row remains in `supplier_invoices_failures` for manual review

### 📝 Update — Retry Failed, Increment Count

- Updates `retry_count` and `last_retry` timestamp
- Row will be picked up again on the next scheduled run

### 📊 Aggregate — Retry Metrics

- Counts re-inserted from **🗑️ Delete — Remove from Failures**
- Counts exhausted from **📝 Update — Mark Exhausted**
- Counts retry-failed from **📝 Update — Retry Failed, Increment Count**
- Single email guaranteed — Aggregate reads from Delete node as primary trigger

### ✉️ Build — Retry Email Body

- Generates HTML email with retry summary
- Red "ACTION REQUIRED" header when exhausted rows exist
- Includes table of exhausted invoices requiring manual intervention

### 📧 Send — Retry Alert Email

- Subject: `Invoice Retry: {reinserted} re-inserted, {exhausted} exhausted — {timestamp} SAST`

---

## Triggers & How to Run

### Trigger 1: Webhook (HTTP Upload)

**Production URL** *(workflow must be Published)*:

```
POST https://johnson-fullstack.app.n8n.cloud/webhook/supplier-invoice-upload
```

**Test URL** *(click "Execute workflow" in n8n first)*:

```
POST https://johnson-fullstack.app.n8n.cloud/webhook-test/supplier-invoice-upload
```

**curl:**

```bash
curl -X POST https://johnson-fullstack.app.n8n.cloud/webhook/supplier-invoice-upload \
  -F "file=@supplier_batch.csv"
```

**PowerShell:**

```powershell
curl.exe -X POST https://johnson-fullstack.app.n8n.cloud/webhook/supplier-invoice-upload `
  -F "file=@supplier_batch.csv"
```

> Note: Use `webhook-test` URL only when manually clicking "Execute workflow" in n8n. Use `webhook` URL when the workflow is Published.

---

### Trigger 2: Google Drive (Folder Watch)

1. Create a folder in Google Drive (e.g. `supplier-invoices`)
2. Open **📁 Google Drive — Watch Folder** node in n8n
3. Connect your Google Drive credential
4. Select your watched folder from the dropdown
5. Set **Watch For** to `File Created`
6. Publish the workflow
7. Upload any CSV to the watched folder — pipeline triggers within 1 minute

The **📥 Google Drive — Download CSV** node uses `{{ $json.id }}` to dynamically download the exact file that triggered the watch node, ensuring the correct file is always fetched.

Both triggers merge at **⚙️ Config — Pipeline Settings** and follow an identical pipeline from that point forward. The **🔧 Normalize — Binary Field Name** node handles the binary field name difference between the two sources transparently.

---

### Retry Pipeline

Runs automatically every 6 hours. To trigger manually, set 'trigger interval' to seconds then open the retry workflow canvas and click **"Execute workflow"**.

---

## CSV Format & Field Mapping

### Expected Headers

```csv
supplier_number,supplier_name,invoice_number,department,invoice_date,amount_excl,vat_rate
```

### Sample CSV (`supplier_batch.csv`)

```csv
supplier_number,supplier_name,invoice_number,department,invoice_date,amount_excl,vat_rate
S009,OfficeCo,OC-22119,Ops,2025-10-28,2175.00,15
S009,OfficeCo,OC-22120,Sales,2025-10-29,450.00,15
S011,PaperMart,PM-77891,Ops,2025-11-01,1020.00,15
S011,PaperMart,PM-77891,Ops,2025-11-01,1020.00,15
```

> Row 4 is an intentional duplicate of Row 3 — used to verify within-batch deduplication.

### Sample Invalid CSV (`supplier_batch_invalid.csv`)

```csv
supplier_number,supplier_name,invoice_number,department,invoice_date,amount_excl,vat_rate
S009,OfficeCo,OC-99999,Ops,2027-01-01,2175.00,15
```

> invoice_date is in the future — triggers validation failure, row goes to `supplier_invoices_failures`.

### Field Mapping

| CSV Column | DB Column | Notes |
| --- | --- | --- |
| `supplier_number` | `supplier_number` | Required |
| `supplier_name` | `supplier_name` | Required |
| `invoice_number` | `invoice_number` | Required |
| `department` | `department` | Required |
| `invoice_date` | `invoice_date` | Required, YYYY-MM-DD |
| `amount_excl` | `amount_excl_vat` | Required |
| `vat_rate` | *(calculation only)* | Optional, defaults to 15% |
| `vat` | `vat` | Auto-calculated: `round(amount_excl × vat_rate / 100, 2)` |
| `amount_incl` | `amount_incl_vat` | Auto-calculated: `amount_excl_vat + vat` |

### Math Fill Logic

```
If vat is missing AND vat_rate is present:
    vat = Math.round(amount_excl × vat_rate / 100 × 100) / 100

If amount_incl_vat is missing:
    amount_incl_vat = Math.round((amount_excl_vat + vat) × 100) / 100
```

Rounding method: standard half-up to 2 decimal places using `Math.round(value × 100) / 100`.

---

## Database Schema & Setup

### Step 1: Run SQL in Supabase SQL Editor

```sql
-- Main invoices table
CREATE TABLE IF NOT EXISTS supplier_invoices (
  id                UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  invoice_number    TEXT NOT NULL,
  supplier_number   TEXT NOT NULL,
  supplier_name     TEXT NOT NULL,
  department        TEXT NOT NULL,
  amount_excl_vat   NUMERIC(12,2) NOT NULL,
  vat               NUMERIC(12,2) NOT NULL,
  amount_incl_vat   NUMERIC(12,2) NOT NULL,
  invoice_date      DATE NOT NULL,
  source_file_name  TEXT,
  source_hash       TEXT,
  ingest_timestamp  TIMESTAMPTZ DEFAULT NOW(),
  status            TEXT CHECK (status IN ('inserted','duplicate','failed')) NOT NULL,
  validation_notes  TEXT,
  UNIQUE (supplier_number, invoice_number)
);

-- Failures / retry table
CREATE TABLE IF NOT EXISTS supplier_invoices_failures (
  id                UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  invoice_number    TEXT,
  supplier_number   TEXT,
  supplier_name     TEXT,
  raw_payload       JSONB,
  validation_notes  TEXT,
  source_file_name  TEXT,
  source_hash       TEXT,
  retry_count       INTEGER DEFAULT 0,
  last_retry        TIMESTAMPTZ,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

-- Staging table for dry-run mode (identical schema to live table)
CREATE TABLE IF NOT EXISTS supplier_invoices_staging
  (LIKE supplier_invoices INCLUDING ALL);
```

### Step 2: Connect Postgres to n8n

1. In n8n go to **Settings → Credentials → Add → Postgres**
2. Fill in your Supabase connection details:
  - **Host:** `db.<your-project-ref>.supabase.co`
  - **Port:** `5432`
  - **Database:** `postgres`
  - **User:** `postgres`
  - **Password:** your Supabase database password
3. Name the credential exactly: `Postgres account`
4. All database nodes in both workflows use this credential name

---

## Validation Rules

Implemented in **🔍 Validate Row — Business Rules** (Run Once for All Items mode):

| #   | Rule | Type | Example |
| --- | --- | --- | --- |
| 1   | All required fields present | ERROR | `Missing required field: department` |
| 2   | `amount_incl_vat = amount_excl_vat + vat` (±0.01 tolerance) | ERROR | `Math check failed: 1200 ≠ 1000 + 150` |
| 3   | Derived VAT rate must equal 15% (SA) | WARNING | `VAT rate mismatch: derived 10%, expected 15%` |
| 4   | `invoice_date` not in the future (Africa/Johannesburg) | ERROR | `Invoice date is in the future: 2027-01-01` |
| 5   | All amounts must be ≥ 0 | ERROR | `amount_excl_vat cannot be negative` |

**Timezone handling:**

```javascript
const nowJHB = new Date(new Date().getTime() + 2 * 60 * 60 * 1000);
const todayJHB = nowJHB.toISOString().split('T')[0];
```

**Routing logic:**

- Rows with any **errors** → `supplier_invoices_failures` (not inserted)
- Rows with **warnings only** → proceed to dedup + insert (warning preserved in `validation_notes`)

---

## Deduplication Logic

Three levels of deduplication enforced:

### Level 1: Within-Batch (JavaScript Set)

Tracks `(supplier_number, invoice_number)` pairs seen in the current CSV batch. First occurrence proceeds; all subsequent occurrences are marked duplicate immediately — before any DB query.

```javascript
const batchKey = `${validated.supplier_number}|${validated.invoice_number}`;
const existsInBatch = seenInBatch.has(batchKey);
if (!isDuplicate) seenInBatch.add(batchKey);
```

### Level 2: Cross-Run (Database Query)

Before tagging each row, queries the live `supplier_invoices` table:

```sql
SELECT EXISTS(
  SELECT 1 FROM supplier_invoices
  WHERE supplier_number = '...' AND invoice_number = '...'
) as is_duplicate;
```

<img width="786" height="483" alt="DEDUP execute once is ON" src="https://github.com/user-attachments/assets/fdbb0c57-b30e-4545-aa07-7dcb02712cd6" />


Screenshot showing 'Always Output Data' toggled on.

### Level 3: Database Constraint (Final Safety Net)

```sql
UNIQUE (supplier_number, invoice_number)
```

Combined with `ON CONFLICT (supplier_number, invoice_number) DO NOTHING` — ensures no duplicates even under race conditions or edge cases.

---

## Binary Field Normalisation

Different trigger sources output binary data under different field names:

- **Webhook** (`multipart/form-data`) → binary field named `file`
- **Google Drive Download** → binary field named `data`

Without normalisation, the **Extract from File** node would fail depending on which trigger fired.

The **🔧 Normalize — Binary Field Name** node solves this by renaming whatever binary field exists to always be `file`:

```javascript
const firstKey = Object.keys(binaryData)[0];
normalizedBinary.file = binaryData[firstKey];
```

This makes the pipeline fully source-agnostic — both triggers work identically with a single shared pipeline. No branching required.

---

## Dry-Run Mode

A `DRY_RUN` config flag enables safe testing without touching production data.

### How to Enable

Open **⚙️ Config — Pipeline Settings** and set:

<img width="938" height="548" alt="DRY_RUN true" src="https://github.com/user-attachments/assets/8cad18a9-abab-4cab-8f2d-1481b869901f" />


Screenshot showing DRY_RUN toggled true.

```javascript
DRY_RUN: true   // Routes all inserts to supplier_invoices_staging
DRY_RUN: false  // Routes all inserts to supplier_invoices (default)
```

### How It Propagates

The flag is read in the Hash node and attached to every row as `dry_run`. It flows through every node in the pipeline until it reaches **🧪 Route — Live vs Dry Run**:

```
DRY_RUN: false → condition "dry_run is false" = TRUE  → supplier_invoices
DRY_RUN: true  → condition "dry_run is false" = FALSE → supplier_invoices_staging
```

### Evidence

When `DRY_RUN: true` is set and `supplier_batch.csv` is submitted:

<img width="961" height="548" alt="staging populated - Copy" src="https://github.com/user-attachments/assets/8e20b0af-a981-4c92-b024-83594e32ac2f" />


**Screenshot A** shows `supplier_invoices_staging` populated with 3 rows.

<img width="963" height="477" alt="normal NOT populated - Copy" src="https://github.com/user-attachments/assets/c47aa8cb-6e73-47a4-8dff-e76a37d2ced6" />


**Screenshot B** shows `supplier_invoices` is empty — production data is completely unaffected.

<img width="1920" height="925" alt="4 processed 3 inserted 1 duplicates 0 failed" src="https://github.com/user-attachments/assets/5bb84dc6-40f1-498e-a010-28d5471db2e5" />


The email alert is sent in both modes with identical metrics and format. The `source_file_name`, `source_hash`, and all counts are preserved in the staging table for full audit traceability.

---

## Retry Pipeline

### How It Works

Every 6 hours the retry workflow:

1. Fetches all rows from `supplier_invoices_failures` where `retry_count <= 3`
2. Parses `raw_payload` JSONB back into structured fields
3. Checks `can_retry` — requires all required fields AND `retry_count <= MAX_RETRIES (3)`
4. Attempts re-insert into `supplier_invoices`
5. On success: deletes row from `supplier_invoices_failures`
6. On failure: increments `retry_count` for the next attempt
7. On exhaustion: marks row with "manual intervention required"
8. Sends a single email summary after every run

### Retry States

| State | Condition | Action |
| --- | --- | --- |
| **Re-inserted** | Insert succeeds | Row deleted from failures, counted as re-inserted in email |
| **Retry Failed** | Insert throws DB error | `retry_count` incremented, `last_retry` updated, retried next run |
| **Exhausted** | `retry_count > 3` or missing required fields | Marked exhausted, stays in failures for manual review, email shows ACTION REQUIRED |

### can_retry Logic

```javascript
const MAX_RETRIES = 3;
const canRetry = !!(record.invoice_number && record.supplier_number &&
                 record.supplier_name && record.department &&
                 record.amount_excl_vat && record.invoice_date) &&
                 record.retry_count <= MAX_RETRIES;
```

### Single Email Guarantee

The **📊 Aggregate — Retry Metrics** node reads from **🗑️ Delete — Remove from Failures** as its primary trigger. This ensures exactly one email is sent per run regardless of how many paths fired.

---

## Email Alerts

### Main Ingest Email

**Subject:**

```
Supplier Ingest: {inserted} ok, {duplicates} dup, {failed} failed — {filename}
```

**Header colours:**

| Result | Colour | Label |
| --- | --- | --- |
| All successful | 🟢 Green | COMPLETED SUCCESSFULLY |
| Duplicates present | 🟠 Orange | COMPLETED WITH WARNINGS |
| Failures present | 🔴 Red | COMPLETED WITH ERRORS |

**Body includes:**

- Source file name + ingest timestamp (SAST)
- 4 metric cards: Total Processed, Inserted, Duplicates, Failed
- Issues table (max 20 rows): Invoice #, Supplier #, Supplier Name, Status badge, Reason
- Footer link to n8n workflow for direct access

**Single email per run** is guaranteed by a guard in the Aggregate node that prevents double-triggering when both the Insert and Mark Duplicate nodes fire in the same execution.

### Retry Email

**Subject:**

```
Invoice Retry: {reinserted} re-inserted, {exhausted} exhausted — {timestamp} SAST
```

**Header colours:**

| Result | Colour | Label |
| --- | --- | --- |
| All re-inserted | 🟢 Green | RETRIES SUCCESSFUL |
| Retries attempted | 🔵 Blue | RETRIES ATTEMPTED |
| Exhausted rows | 🔴 Red | ACTION REQUIRED |

**Body includes:**

- Run timestamp (SAST)
- Re-inserted, Retry Updated, Exhausted metric cards
- Exhausted invoices table for manual intervention
- Footer link to n8n workflow

### Email Node Configuration

1. In n8n go to **Settings → Credentials → Add → Gmail OAuth2** (or SMTP)
2. Open **📧 Send — Ingest Alert Email** → select credential → set `To` address
3. Repeat for **📧 Send — Retry Alert Email** in the retry workflow

---

## n8n Credentials & Configuration

| Credential Name | Type | Used By |
| --- | --- | --- |
| `Postgres account` | Postgres | All DB nodes in both workflows |
| `Google Drive account` | Google OAuth2 | Watch Folder + Download CSV nodes |
| `Gmail account` | Gmail OAuth2 | Send email nodes in both workflows |

### Importing & Setting Up

1. Open n8n → **"Add workflow"** → **"Import from file"**
2. Import `supplier-ingest.json`
3. Import `supplier-retry.json`
4. Update all credentials to your own in every node
5. Open **📁 Google Drive — Watch Folder** → select your watched folder
6. Update `To` email addresses in both **Send** nodes
7. Click **"Publish"** on both workflows

---

## Test Evidence

### Test 1 — Successful First Run

- **Input:** `supplier_batch.csv` (4 rows, 1 intentional within-batch duplicate)
- **Expected:** 3 inserted, 1 duplicate, 0 failed
- **Email subject:** `Supplier Ingest: 3 ok, 1 dup, 0 failed — supplier_batch.csv`
- **Evidence:** Email screenshot + Supabase `supplier_invoices` showing 3 rows with `status = 'inserted'`

<img width="1920" height="925" alt="4 processed 3 inserted 1 duplicates 0 failed" src="https://github.com/user-attachments/assets/4b8260eb-02f2-4cab-b325-7bc743717c48" />


Screenshot of email showing 4 `processed`, 3 `inserted`, 1 `duplicates` and 0 `failed` operations.

<img width="906" height="674" alt="SQL Query 1" src="https://github.com/user-attachments/assets/6504e139-5092-40b7-83c7-cf56de44846a" />


Screenshot showing 3 rows of supplier invoice data.

### Test 2 — Full Duplicate Run

- **Input:** Same `supplier_batch.csv` submitted again (all 3 rows already in DB + 1 within-batch dup)
- **Expected:** 0 inserted, 4 duplicates, 0 failed
- **Evidence:** Email screenshot showing 0 inserted, 4 duplicates

<img width="1920" height="919" alt="4 processed 0 inserted 4 duplicates 0 failed" src="https://github.com/user-attachments/assets/b34e1b5a-82e0-4e0b-b195-e28f6ade34b3" />

Screenshot of email showing 4 `processed`, 0 `inserted`, 4`duplicates` and 0 `failed` operations.

### Test 3 — Validation Failure

- **Input:** `supplier_batch_invalid.csv` (`invoice_date: 2027-01-01` — future date)
- **Expected:** 0 inserted, 0 duplicates, 1 failed
- **Evidence:** Email showing `ERROR: Invoice date is in the future: 2027-01-01` + Supabase `supplier_invoices_failures` screenshot

<img width="1920" height="920" alt="0 processed 0 inserted 0 duplicates 1 failed with reason" src="https://github.com/user-attachments/assets/ae57e87d-7ee5-4a11-ac6d-c126795a329a" />

Screenshot of email showing 0 `processed`, 0 `inserted`, 0 `duplicates `and 1 `failed` operations.

<img width="1920" height="390" alt="failed table" src="https://github.com/user-attachments/assets/a71e0a4b-bed4-4f56-b4ec-0cf85d33e281" />

Screenshot showing a single failed entry in '`supplier_invoices_failures`'.

### Test 4 — Dry-Run Mode

- **Config:** `DRY_RUN: true` set in ⚙️ Config — Pipeline Settings
- **Input:** `supplier_batch.csv`
- **Expected:** 3 rows written to `supplier_invoices_staging`, `supplier_invoices` remains empty
- **Evidence:**
  - Screenshot A: `supplier_invoices_staging` with 3 rows populated
  - Screenshot B: `supplier_invoices` empty — production data unaffected
  - The email metrics are identical to a live run; only the destination table differs

<img width="938" height="548" alt="DRY_RUN true" src="https://github.com/user-attachments/assets/9eebe64b-9d6c-49db-b01b-51c2e5532c69" />

Screenshot showing variable `DRY_RUN` set to '`true`'.

<img width="1920" height="535" alt="staging table 3 rows" src="https://github.com/user-attachments/assets/30eec52e-c897-48f4-9e4d-937c586b5a76" />

Screenshot showing multiple entries into `supplier_invoices_staging` table.

<img width="963" height="477" alt="normal NOT populated" src="https://github.com/user-attachments/assets/b9673b7d-cf01-42b9-9e78-61995e5c858c" />

Screenshot showing no entries in the `supplier_invoices` table.

### Test 5 — Google Drive Trigger

- **Action:* Uploaded `supplier_batch.csv` directly to the watched Google Drive folder ![file in g drivePNG]

<img width="957" height="505" alt="file in g drive" src="https://github.com/user-attachments/assets/61610703-782f-4462-af0a-a322e7e728a4" />

Screenshot showing `supplier_batch.csv` file inside folder named `supplier-invoices-inbox` in Google Drive.

- **Expected:** Pipeline triggers automatically within 1 minute, identical results to Test 1

<img width="1659" height="755" alt="auto triggered g branch" src="https://github.com/user-attachments/assets/bc66318f-0bce-47a0-a1f2-26d411c0b775" />

Screenshot of showing completed workflow triggered by `Google Drive Watch Folder`.

- **Evidence:** Email received without any manual webhook call — triggered purely by the Drive upload

- Note: The **🔧 Normalize — Binary Field Name** node handles the `data` → `file` binary field rename transparently so the same pipeline processes both sources

### Test 6 — Retry Success

- **Setup:** Valid row inserted directly into `supplier_invoices_failures` with `retry_count: 0`

<img width="959" height="474" alt="sql false insert" src="https://github.com/user-attachments/assets/55a6667f-0bd7-4e57-9b03-28a611995d40" />

Screenshot showing SQL query of invalid entry into `supplier_invoices_failures`.

- **Action:** Retry workflow triggered manually
- **Expected:** Row re-inserted into `supplier_invoices`, deleted from `supplier_invoices_failures`
- **Evidence:** Retry email showing `1 re-inserted, 0 exhausted`

<img width="1920" height="912" alt="1 re-inserted 0 retry-updated 0 exhausted" src="https://github.com/user-attachments/assets/ee804685-5eff-4d4c-9d97-c8a38207d486" />


Screenshot of email showing 1 `re-inserted`, 0 `retry updated` and 0 `exhausted` operations.

### Test 7 — Retry Exhaustion

- **Setup:** Row inserted into `supplier_invoices_failures` with `retry_count: 3`
- **Action:** Retry workflow triggered
- **Expected:** Row marked exhausted, remains in failures table for manual review
- **Evidence:** Retry email showing `0 re-inserted, 1 exhausted` with red "ACTION REQUIRED" header and exhausted invoices table

<img width="1920" height="918" alt="0 re-inserted 0 retry-updated 1 exhausted" src="https://github.com/user-attachments/assets/faab7f6b-986e-4f3e-bdd7-eba761cd12fc" />

Screenshot of email showing 0 `re-inserted`, 0 `retry updated` and 1 `exhausted` operations.

### DB Query Evidence

Run the following in Supabase SQL Editor to see per-row status and validation notes:

```sql
SELECT invoice_number, supplier_number, status, validation_notes
FROM supplier_invoices
ORDER BY ingest_timestamp DESC;
```
<img width="906" height="674" alt="SQL Query 1" src="https://github.com/user-attachments/assets/c8ede384-0760-4ae7-94a0-b9d66d1ed1b7" />

Screenshot of results shows each input row mapped to its final `status` and any `validation_notes`.

---

## Key Design Decisions

| Decision | Rationale |
| --- | --- |
| **Immediate webhook response** | `Respond — Acknowledged` fires instantly so the HTTP client gets `200` without waiting for the full pipeline |
| **SHA-256 file hashing** | `source_hash` stored with every row — same file can be detected and skipped for idempotency |
| **Binary field normalisation node** | Single node makes the pipeline source-agnostic — no branching needed for Webhook vs Google Drive |
| **Extract from File node** | Replaces manual base64 decoding — handles encoding, BOM, and quoted fields natively |
| **Three-level deduplication** | Within-batch (Set) + cross-run (DB query) + constraint (UNIQUE key) — no duplicate can slip through |
| **`DRY_RUN` config flag** | Single flag in one node controls entire pipeline routing — easy to find, easy to toggle |
| **`ON CONFLICT DO NOTHING`** | Prevents crashes on edge cases while the UNIQUE constraint acts as final guard |
| **Retry as separate workflow** | Clean separation of concerns — ingest pipeline not polluted with retry logic |
| **`raw_payload JSONB`** | Stores the complete original row in failures so retries have full data even if original file is gone |
| **`can_retry` flag** | Computed in Prepare node — decouples retry eligibility logic from the routing node |
| **Aggregate guard** | Prevents double email when Mark Duplicate and Insert both feed into Aggregate in same run |
| **Africa/Johannesburg timezone** | All timestamps and date comparisons use SAST — `NOW() AT TIME ZONE 'Africa/Johannesburg'` |
| **Execution link in email** | Each email footer links directly to the n8n workflow for traceability and easy access |

---

*Supplier Invoice Ingest Pipeline · n8n Automation Exercise · Africa/Johannesburg · March 2026*
