# Email Automation (n8n Workflow)

An n8n workflow that reads pending contacts from a Google Sheet, verifies each email address through a three-stage validation pipeline, sends the email via Gmail if the address is deliverable, and writes the outcome back to the sheet.

## What it does

1. **Trigger** – runs automatically every day at 9:00 AM (`Schedule Trigger`).
2. **Fetch rows** – pulls all rows from the "Email Automation" Google Sheet (`Get Email`).
3. **Filter** – keeps only rows where `status = "Pending"` (`Pending Status`).
4. **Loop** – processes the pending rows one at a time (`Loop Over Items`).
5. **Validate the email address**, in stages, stopping early if a stage fails:
   - **Stage 1 – Format/MX check** via Mailboxlayer (`apilayer.net/api/check`). Passes only if `format_valid` and `mx_found` are both `true`.
   - **Stage 2 – Deliverability check** via Emailable (`api.emailable.com/v1/verify`). Passes only if `state = "deliverable"`.
   - **Stage 3 – Fallback check** via ZeroBounce, used when Emailable says the address is *not* deliverable. Passes only if ZeroBounce's `status` contains `"Valid"`.
6. **Send** – if an address passes verification (via Emailable or the ZeroBounce fallback), the workflow sends the email through Gmail using the row's `Email`, `Subject`, and `message` fields.
7. **Update the sheet** – after each attempt, the corresponding row is updated with the outcome:
   - `Send` – email was verified and sent successfully
   - `Email Failed` – verified, but the Gmail send did not confirm as sent
   - `Verification Failed` – the address failed format/MX, Emailable, or ZeroBounce checks
   Each update also stamps `sending date` and which service (`verified by`) confirmed the address.
8. **Loop continues** until every pending row has been processed.

## Google Sheet structure

The workflow reads and writes a sheet named **"Email Automation"** with (at least) these columns:

| Column | Purpose |
|---|---|
| `Email` | Recipient's email address |
| `Subject` | Email subject line |
| `message` | Email body (plain text) — note the column is referenced as `message ` (trailing space) in some nodes |
| `status` | Row state: `Pending`, `Send`, `Email Failed`, or `Verification Failed` |
| `sending date` | Timestamp the row was last processed |
| `verified by` | Which service verified the address (`Emailable`, `ZeroBounce`, or `None`) |

Only rows with `status = Pending` are picked up on each run.

## Nodes overview

| Node | Type | Role |
|---|---|---|
| Schedule Trigger | Schedule Trigger | Runs daily at 9 AM |
| Get Email | Google Sheets | Reads all rows |
| Pending Status | Filter | Keeps rows with `status = Pending` |
| Loop Over Items | Split in Batches | Iterates rows one by one |
| Mailboxlayer Format Checker | HTTP Request | Checks email format & MX records |
| If | If | Routes on format/MX validity |
| Emailable Verification | HTTP Request | Checks deliverability |
| If2 | If | Routes on `state = deliverable` |
| Validate an email | ZeroBounce node | Fallback validation |
| If1 | If | Routes on ZeroBounce `status` containing `Valid` |
| Send Email / Send Email1 | Gmail | Sends the email |
| If4 / If5 | If | Confirms Gmail send via `labelIds[0] = SENT` |
| Update Status To Send / To Send1 | Google Sheets | Marks row as sent |
| Update Status Email Failed / Failed1 | Google Sheets | Marks row as failed to send |
| Verification Failed / Failed1 | Google Sheets | Marks row as failed verification |
| No Operation nodes | NoOp | Loop-back / branch merge points |

## Required credentials & connections

- **Google Sheets OAuth2** – for reading and updating the tracking sheet
- **Gmail OAuth2** – for sending emails
- **Mailboxlayer (apilayer.net)** – API key for format/MX checking
- **Emailable** – API key for deliverability checking
- **ZeroBounce** – API credential (configured as an n8n credential, used as fallback)

## ⚠️ Security note

The **Mailboxlayer** and **Emailable** HTTP Request nodes currently have their API keys hardcoded directly in the query parameters inside the workflow JSON, rather than stored as n8n credentials. Anyone with access to this workflow file can see and reuse those keys. Before sharing, exporting, or committing this workflow anywhere:

- Move both keys into n8n credentials (or environment variables) and reference them with an expression instead of a literal string.
- Rotate the exposed keys, since they've already been visible in this file.

## Setup checklist

1. Import the workflow into n8n.
2. Connect/select credentials for Google Sheets, Gmail, and ZeroBounce.
3. Replace the hardcoded Mailboxlayer and Emailable API keys with secure credentials (see security note above).
4. Point the two Google Sheets "Get/Update" nodes at your own spreadsheet (update `documentId`/`sheetName` if you're not reusing the original sheet).
5. Ensure your sheet has the `Email`, `Subject`, `message`, `status`, `sending date`, and `verified by` columns, with rows to be sent marked `status = Pending`.
6. Activate the workflow, or run it manually to test on a few rows first.
