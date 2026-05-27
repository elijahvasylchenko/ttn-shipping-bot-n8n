# TTN Shipping Label Bot

> Telegram bot that automatically fetches orders from LP-CRM and generates
> printable PDF shipping labels (TTN) — built with n8n + Labelary API.

---

## What It Does

An operations manager types `/get` in Telegram — the bot pulls all pending
orders from LP-CRM, generates ZPL shipping labels for each, converts them
to a single PDF, and sends the file back. One command replaces 20+ minutes
of manual copy-paste.

**Full flow:**

```
/get command in Telegram
        ↓
   Admin check (secret code + DataTable whitelist)
        ↓
   LP-CRM API → fetch orders by status
        ↓
   LP-CRM API → fetch order details (IDs, TTNs, products)
        ↓
   Parse orders → build ZPL label code per order
        ↓
   Labelary API → convert ZPL to PDF
        ↓
   Send PDF to user in Telegram
```

---

## Architecture

```
Telegram Trigger
    ↓
Admin auth (2-layer):
├── Secret code check (IF node)
│   └── Upsert to DataTable → "Access granted"
└── DataTable lookup → whitelist check
        ↓ (admin only)
   Command router (Switch):
   ├── /start  → Welcome + log to Google Sheets
   ├── /help   → Help message
   ├── /get    → Label generation pipeline
   └── text    → Typing indicator (placeholder)

Label pipeline:
   Config node (API key, status code)
        ↓
   LP-CRM: getOrdersIdByStatus
        ↓
   LP-CRM: getOrdersByID (with retry, 5 attempts)
        ↓
   Code: parse orders → extract order_id, ttn, products
        ↓
   Loop Over Items (batch: 2)
        ↓
   Aggregate (collect all orders)
        ↓
   Code: build combined ZPL string for all labels
        ↓
   Labelary API: ZPL → PDF (4x4 inch, 8dpmm)
        ↓
   Send PDF document to Telegram user
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Automation | n8n |
| CRM | LP-CRM (lp-crm.biz) via REST API |
| Label rendering | Labelary API (labelary.com) |
| Label format | ZPL (Zebra Programming Language) |
| User registry | n8n DataTable |
| Logging | Google Sheets |
| Messaging | Telegram Bot API |

---

## Key Features

**2-layer admin authentication**
First layer: secret code check (user types a private code to register).
Second layer: DataTable whitelist — every message checks if sender is in
approved users list. Unknown users get "access denied" silently.

**Batch processing with retry**
LP-CRM API calls use `splitInBatches` (2 per batch) with retry logic
(5 attempts, 5s delay). Handles API rate limits and timeouts without
failing the whole pipeline.

**Combined ZPL for all orders**
All orders are aggregated into a single ZPL string, converted to one PDF
in a single Labelary call. User receives one file, not 30 separate ones.

**Label content per order:**
- Order number
- TTN tracking number (large, scannable font)
- Product list with quantities

---

## ZPL Label Structure

```zpl
^XA
^CI28
^CF0,40
^FO50,50^FDOrder: #12345^FS
^CF0,80
^FO50,150^FD20450000000000^FS        ← TTN (large)
^CF0,40
^FO50,280^FB600,4,,L^FDProducts: ...^FS
^XZ
```

Label size: **4×4 inch** at **8 dpmm** (203 DPI) — standard for Nova Poshta
and other Ukrainian carriers.

---

## LP-CRM API Endpoints Used

| Endpoint | Purpose |
|---|---|
| `POST /api/getOrdersIdByStatus.html` | Get order IDs for a given status code |
| `POST /api/getOrdersByID.html` | Get full order details by IDs |

**Configuration** (in `Config: API key + status` node):

```json
{
  "key": "YOUR_LP_CRM_API_KEY",
  "status": "12"
}
```

Status `12` = "Ready to ship" in default LP-CRM setup. Change to match
your workflow.

---

## Google Sheets Logging

On `/start`, the bot logs:

| Column | Value |
|---|---|
| User ID | Telegram chat ID |
| Username | Telegram username |
| Date | Timestamp |

Replace `YOUR_GOOGLE_SHEET_ID` in the `Append row in sheet` node with
your actual Google Sheets ID.

---

## Setup

### Requirements

- n8n instance (self-hosted or cloud)
- Telegram Bot Token
- LP-CRM account + API key
- Google Sheets OAuth2 credential (for logging)

### Installation

1. Import `workflow/ttn-bot-workflow.json` into n8n
2. Set up credentials:
   - `Telegram API` → your bot token
   - `Google Sheets OAuth2` → your Google account
3. Replace placeholders:
   - `YOUR_LP_CRM_API_KEY` in `Config: API key + status` node
   - `YOUR_LP_CRM_DOMAIN` in HTTP Request nodes
   - `YOUR_ADMIN_TG_ID` in `Allow admin only` node
   - `YOUR_GOOGLE_SHEET_ID` in `Append row in sheet` node
   - `YOUR_DATATABLE_ID` → create a DataTable in n8n with fields:
     `tg_id` (number), `name` (string), `is_main` (boolean)
4. Set your secret registration code in `Admin secret code check` node
5. Activate workflow
6. Send secret code to bot — you'll be added to admin whitelist

### DataTable Schema

Create in n8n → DataTables:

```
Table name: Bot Users
Fields:
  - tg_id     (number, unique)
  - name      (string)
  - is_main   (boolean)
```

---

## Commands

| Command | What it does |
|---|---|
| `YOUR_SECRET_CODE` | Register as admin (one-time setup) |
| `/start` | Welcome message + log user to Google Sheets |
| `/help` | Help text (customize in `Telegram3` node) |
| `/get` | Generate and send TTN labels PDF |

---

## Customization

| What to change | Where |
|---|---|
| LP-CRM order status to fetch | `Config` node → `status` field |
| Label size | Labelary URL — change `4x4` to your size |
| Label DPI | Labelary URL — change `8dpmm` |
| Label content/layout | `Build ZPL labels` Code node |
| Admin chat IDs | `Allow admin only` node |
| Batch size | `Loop Over Items` node → `batchSize` |

---

## Project Status

Working prototype. Built for internal ops automation at an e-commerce
company using Nova Poshta / LP-CRM stack.

Currently adapted as a **reusable template** — swap LP-CRM credentials
and status code for any client using the same CRM.

---

## Related Projects

- [AI Booking Bot for Beauty Studios](https://github.com/elijahvasylchenko/ai-booking-bot-telegram)
  — n8n + Claude + Cal.com + Telegram booking automation

---

## Author

**Illia Vasylchenko** — AI Automation Engineer  
Founder @ [SystemFlow](https://elijahvasylchenko.github.io/services/)  
[elijah.vasylchenko@gmail.com](mailto:elijah.vasylchenko@gmail.com) · [t.me/ilya_vasylchenko](https://t.me/ilya_vasylchenko)TTN Shipping Label Bot

Telegram bot that automatically fetches orders from LP-CRM and generates
printable PDF shipping labels (TTN) — built with n8n + Labelary API.


What It Does
An operations manager types /get in Telegram — the bot pulls all pending
orders from LP-CRM, generates ZPL shipping labels for each, converts them
to a single PDF, and sends the file back. One command replaces 20+ minutes
of manual copy-paste.
Full flow:
/get command in Telegram
        ↓
   Admin check (secret code + DataTable whitelist)
        ↓
   LP-CRM API → fetch orders by status
        ↓
   LP-CRM API → fetch order details (IDs, TTNs, products)
        ↓
   Parse orders → build ZPL label code per order
        ↓
   Labelary API → convert ZPL to PDF
        ↓
   Send PDF to user in Telegram

Architecture
Telegram Trigger
    ↓
Admin auth (2-layer):
├── Secret code check (IF node)
│   └── Upsert to DataTable → "Access granted"
└── DataTable lookup → whitelist check
        ↓ (admin only)
   Command router (Switch):
   ├── /start  → Welcome + log to Google Sheets
   ├── /help   → Help message
   ├── /get    → Label generation pipeline
   └── text    → Typing indicator (placeholder)

Label pipeline:
   Config node (API key, status code)
        ↓
   LP-CRM: getOrdersIdByStatus
        ↓
   LP-CRM: getOrdersByID (with retry, 5 attempts)
        ↓
   Code: parse orders → extract order_id, ttn, products
        ↓
   Loop Over Items (batch: 2)
        ↓
   Aggregate (collect all orders)
        ↓
   Code: build combined ZPL string for all labels
        ↓
   Labelary API: ZPL → PDF (4x4 inch, 8dpmm)
        ↓
   Send PDF document to Telegram user

Tech Stack
LayerToolAutomationn8nCRMLP-CRM (lp-crm.biz) via REST APILabel renderingLabelary API (labelary.com)Label formatZPL (Zebra Programming Language)User registryn8n DataTableLoggingGoogle SheetsMessagingTelegram Bot API

Key Features
2-layer admin authentication
First layer: secret code check (user types a private code to register).
Second layer: DataTable whitelist — every message checks if sender is in
approved users list. Unknown users get "access denied" silently.
Batch processing with retry
LP-CRM API calls use splitInBatches (2 per batch) with retry logic
(5 attempts, 5s delay). Handles API rate limits and timeouts without
failing the whole pipeline.
Combined ZPL for all orders
All orders are aggregated into a single ZPL string, converted to one PDF
in a single Labelary call. User receives one file, not 30 separate ones.
Label content per order:

Order number
TTN tracking number (large, scannable font)
Product list with quantities


ZPL Label Structure
zpl^XA
^CI28
^CF0,40
^FO50,50^FDOrder: #12345^FS
^CF0,80
^FO50,150^FD20450000000000^FS        ← TTN (large)
^CF0,40
^FO50,280^FB600,4,,L^FDProducts: ...^FS
^XZ
Label size: 4×4 inch at 8 dpmm (203 DPI) — standard for Nova Poshta
and other Ukrainian carriers.

LP-CRM API Endpoints Used
EndpointPurposePOST /api/getOrdersIdByStatus.htmlGet order IDs for a given status codePOST /api/getOrdersByID.htmlGet full order details by IDs
Configuration (in Config: API key + status node):
json{
  "key": "YOUR_LP_CRM_API_KEY",
  "status": "12"
}
Status 12 = "Ready to ship" in default LP-CRM setup. Change to match
your workflow.

Google Sheets Logging
On /start, the bot logs:
ColumnValueUser IDTelegram chat IDUsernameTelegram usernameDateTimestamp
Replace YOUR_GOOGLE_SHEET_ID in the Append row in sheet node with
your actual Google Sheets ID.

Setup
Requirements

n8n instance (self-hosted or cloud)
Telegram Bot Token
LP-CRM account + API key
Google Sheets OAuth2 credential (for logging)

Installation

Import workflow/ttn-bot-workflow.json into n8n
Set up credentials:

Telegram API → your bot token
Google Sheets OAuth2 → your Google account


Replace placeholders:

YOUR_LP_CRM_API_KEY in Config: API key + status node
YOUR_LP_CRM_DOMAIN in HTTP Request nodes
YOUR_ADMIN_TG_ID in Allow admin only node
YOUR_GOOGLE_SHEET_ID in Append row in sheet node
YOUR_DATATABLE_ID → create a DataTable in n8n with fields:
tg_id (number), name (string), is_main (boolean)


Set your secret registration code in Admin secret code check node
Activate workflow
Send secret code to bot — you'll be added to admin whitelist

DataTable Schema
Create in n8n → DataTables:
Table name: Bot Users
Fields:
  - tg_id     (number, unique)
  - name      (string)
  - is_main   (boolean)

Commands
CommandWhat it doesYOUR_SECRET_CODERegister as admin (one-time setup)/startWelcome message + log user to Google Sheets/helpHelp text (customize in Telegram3 node)/getGenerate and send TTN labels PDF

Customization
What to changeWhereLP-CRM order status to fetchConfig node → status fieldLabel sizeLabelary URL — change 4x4 to your sizeLabel DPILabelary URL — change 8dpmmLabel content/layoutBuild ZPL labels Code nodeAdmin chat IDsAllow admin only nodeBatch sizeLoop Over Items node → batchSize

Project Status
Working prototype. Built for internal ops automation at an e-commerce
company using Nova Poshta / LP-CRM stack.
Currently adapted as a reusable template — swap LP-CRM credentials
and status code for any client using the same CRM.

Related Projects

AI Booking Bot for Beauty Studios
— n8n + Claude + Cal.com + Telegram booking automation


Author
Illia Vasylchenko — AI Automation Engineer
Founder @ SystemFlow
elijah.vasylchenko@gmail.com · t.me/ilya_vasylchenko
