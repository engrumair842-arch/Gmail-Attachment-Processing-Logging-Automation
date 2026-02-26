# 📧 Gmail Attachment → Processing & Logging Automation

A Zapier automation that intelligently processes email attachments (PDFs and images), extracts structured receipt data using OpenAI Vision, stores files in Google Drive, and logs records into Google Sheets.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Diagram Summary](#workflow-diagram-summary)
- [Prerequisites](#prerequisites)
- [Step-by-Step Workflow](#step-by-step-workflow)
  - [Step 1 – Gmail Trigger](#step-1--gmail-trigger)
  - [Step 2 – File Type Filter](#step-2--file-type-filter)
  - [Step 3 – Split into Paths](#step-3--split-into-paths)
  - [Path A – PDF Processing](#path-a--pdf-processing)
  - [Path B – Image Processing](#path-b--image-processing)
- [Data Extraction Schema](#data-extraction-schema)
- [Folder Structure](#folder-structure)
- [Google Sheets Columns](#google-sheets-columns)

---

## Overview

This automation activates whenever a new email attachment is received in Gmail. It validates the file type, routes it through the appropriate processing path, uses GPT-4 Vision to extract key receipt fields, saves the file to Google Drive, and logs a structured record to Google Sheets.

**Supported file types:** `.pdf`, `.jpg`, `.jpeg`, `.png`

---

## Workflow Diagram Summary

```
Gmail Trigger (New Attachment)
        │
        ▼
  [Filter] Valid MIME type?
  (jpg / jpeg / png / pdf only)
        │
        ▼
  [Paths] Is it a PDF or Image?
    ┌────┴────┐
    │         │
  Path A    Path B
  (PDF)    (Image)
    │         │
    ▼         ▼
Convert    Sub-Path Split
to Image    ┌──┴──┐
    │      Path C  Path D
    │     (Image)  (PDF → Terminate)
    ▼         │
Analyze    Analyze
 Image      Image
    │         │
    ▼         ▼
Parse JSON  Parse JSON
    │         │
    ▼         ▼
Upload to  Upload to
  Drive      Drive
    │         │
    ▼         ▼
Log to     Log to
  Sheets    Sheets
```

---

## Prerequisites

The following app connections must be configured in Zapier before activating this workflow:

- **Gmail** – Connected account with access to the target inbox
- **Zapier Filter** – Built-in; no external connection required
- **Zapier Paths** – Built-in; no external connection required
- **CloudConvert** – API key for PDF-to-image conversion
- **OpenAI** – API key with access to GPT-4 Vision (`gpt-4o` or compatible model)
- **Code by Zapier** – Built-in JavaScript runner
- **Google Drive** – Connected account with write access to the target folder
- **Google Sheets** – Connected account with access to the target spreadsheet

---

## Step-by-Step Workflow

### Step 1 – Gmail Trigger

| Field | Value |
|---|---|
| **App** | Gmail |
| **Trigger Event** | New Attachment |
| **Purpose** | Fires the automation whenever an email with an attachment arrives |

---

### Step 2 – File Type Filter

| Field | Value |
|---|---|
| **App** | Zapier Filter |
| **Condition** | Attachment MIME type contains `jpg`, `jpeg`, `png`, or `pdf` |
| **Purpose** | Prevents unsupported file types from continuing through the workflow |

---

### Step 3 – Split into Paths

| Field | Value |
|---|---|
| **App** | Zapier Paths |
| **Path A** | File is a PDF |
| **Path B** | File is PNG or JPEG |

---

### Path A – PDF Processing

#### Step 4 – IF File is PDF

| Field | Value |
|---|---|
| **Condition** | Attachment contains `pdf` |
| **Result** | Continue to PDF processing |

#### Step 5 – Convert PDF to Image

| Field | Value |
|---|---|
| **App** | CloudConvert |
| **Action** | Convert File |
| **Input** | PDF attachment from Gmail |
| **Output** | Image file (compatible with GPT Vision) |

#### Step 6 – Analyze Image (PDF Path)

| Field | Value |
|---|---|
| **App** | OpenAI |
| **Action** | Analyze Image (ChatGPT Vision) |
| **Input** | Converted image from CloudConvert |

**Prompt used:**

```
You are a bookkeeping assistant.

Analyze this receipt image and extract: vendor (clean name), receipt_date (YYYY-MM-DD),
total_amount (numeric only), payment_type, and category
(ONE of: Fuel, Lodging, Meals, Supplies, Travel, Parking, Tolls, Other).

Return ONLY valid JSON with no extra text.
Do not use markdown, code fences, or backticks.
Use null for missing values.

JSON format:
{"vendor":null,"receipt_date":null,"total_amount":null,"payment_type":null,"category":null}
```

#### Step 7 – Parse JSON (Code by Zapier)

| Field | Value |
|---|---|
| **App** | Code by Zapier (JavaScript) |
| **Purpose** | Cleans and parses the raw GPT response into structured fields |

**Code:**

```javascript
const raw = inputData.image_analysis;
const cleaned = raw.replace(/```json|```/g, '').trim();
const data = JSON.parse(cleaned);
return {
  vendor: data.vendor || '',
  receipt_date: data.receipt_date || '',
  total_amount: data.total_amount || '',
  payment_type: data.payment_type || '',
  category: data.category || ''
};
```

**Example output:**

```json
{
  "vendor": "Marriott Hotel",
  "receipt_date": "2026-02-20",
  "total_amount": "245.67",
  "payment_type": "Credit Card",
  "category": "Lodging"
}
```

#### Step 8 – Upload File to Google Drive

| Field | Value |
|---|---|
| **App** | Google Drive |
| **Action** | Upload File |
| **Destination** | `Drive → Receipts → 2026 → February` |

#### Step 9 – Log to Google Sheets

| Field | Value |
|---|---|
| **App** | Google Sheets |
| **Action** | Create Spreadsheet Row |
| **Data Mapped** | Fields from Code by Zapier + Google Drive link + Gmail trigger metadata |

---

### Path B – Image Processing

#### Step 10 – IF File is an Image

| Field | Value |
|---|---|
| **Condition** | Attachment contains `jpg`, `png`, or `jpeg` |
| **Result** | Continue to image processing |

#### Step 11 – Split into Sub-Paths

| Field | Value |
|---|---|
| **App** | Zapier Paths |
| **Path C** | File ends with `jpg`, `jpeg`, or `png` → proceed to image analysis |
| **Path D** | File ends with `pdf` → terminate cleanly |

#### Step 12 – IF File is JPG/JPEG/PNG (Path C)

| Field | Value |
|---|---|
| **Condition** | Attachment MIME type ends with `jpg`, `jpeg`, or `png` |
| **Result** | Continue to image analysis |

#### Step 13 – Analyze Image (Image Path)

Same configuration as [Step 6](#step-6--analyze-image-pdf-path). Uses the same OpenAI Vision prompt to extract receipt data from the native image attachment.

#### Step 14 – Parse JSON (Code by Zapier)

Same code as [Step 7](#step-7--parse-json-code-by-zapier). Cleans and structures the GPT response output.

#### Step 15 – Upload File to Google Drive

Same configuration as [Step 8](#step-8--upload-file-to-google-drive).

| Field | Value |
|---|---|
| **Destination** | `Drive → Receipts → 2026 → February` |

#### Step 16 – Log to Google Sheets

Same configuration as [Step 9](#step-9--log-to-google-sheets).

#### Step 17 – IF File is PDF (Path D — Terminate)

| Field | Value |
|---|---|
| **Condition** | Attachment MIME type ends with `pdf` |
| **Result** | Terminate workflow cleanly (PDF already handled via Path A) |

#### Step 18 – Terminate (Code by Zapier)

| Field | Value |
|---|---|
| **App** | Code by Zapier (JavaScript) |
| **Purpose** | Stops execution cleanly to avoid errors when a PDF reaches Path D |

**Code:**

```javascript
return { terminated: true };
```

---

## Data Extraction Schema

The following fields are extracted from every processed receipt:

| Field | Format | Description |
|---|---|---|
| `vendor` | String | Clean business/vendor name |
| `receipt_date` | `YYYY-MM-DD` | Date on the receipt |
| `total_amount` | Numeric | Total charge (no currency symbols) |
| `payment_type` | String | e.g., Credit Card, Cash, Debit |
| `category` | Enum | One of: `Fuel`, `Lodging`, `Meals`, `Supplies`, `Travel`, `Parking`, `Tolls`, `Other` |

---

## Folder Structure

Processed files are stored in Google Drive using the following structure:

```
My Drive/
└── Receipts/
    └── 2026/
        └── February/
            ├── receipt_001.pdf
            ├── receipt_002.jpg
            └── ...
```

---

## Google Sheets Columns

Each processed receipt creates a new row with the following mapped columns:

| Column | Source |
|---|---|
| Vendor | Code by Zapier |
| Receipt Date | Code by Zapier |
| Total Amount | Code by Zapier |
| Payment Type | Code by Zapier |
| Category | Code by Zapier |
| File Link | Google Drive (Upload step) |
| Email Subject | Gmail Trigger |
| Email Date | Gmail Trigger |
| Sender | Gmail Trigger |

---

> **Note:** This workflow handles mixed attachment scenarios by using nested path logic. PDFs are converted to images before analysis, while native image attachments are analyzed directly. Any PDF that enters the image branch (Path B) is safely terminated via Path D to prevent duplicate processing.
