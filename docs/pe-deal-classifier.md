# PE Deal Email Classifier & NDA Request - Documentation

## Overview

An n8n workflow that automates private equity deal email screening. It classifies inbound emails, extracts PDF attachment content (teasers/CIMs), evaluates deals against investment criteria, drafts both response emails (NDA request and pass), and presents everything in a human-in-the-loop review before sending a threaded reply with the sender's full name.

## Workflow Flow

```
New Email --> Fetch Full Email --> Has PDF? --[yes]--> Extract PDF --> Classify Email --[PE Deal]--> Analyze & Draft --> Review Decision --> Pursue or Pass? --> Send Threaded Reply
                                           \--[no]-----------------> Classify Email
                                                                    \--[Not Deal/Other]--> Skip Non-Deal
```

### Node-by-Node Breakdown

#### 1. New Email (Gmail Trigger)
- **Type**: `n8n-nodes-base.gmailTrigger` v1.3
- **Config**: Polls every minute for unread emails
- **Output**: Basic email metadata including message `id`
- **Note**: Does NOT reliably extract email body for forwarded/multipart MIME emails
- **TODO**: Add sender filter to exclude your own email address to prevent self-email feedback loop from sendAndWait review emails

#### 2. Fetch Full Email (Gmail Get)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `get`
- **Config**: `simple: false` to get structured `from`/`to` objects plus `textPlain` and `textHtml` fields
- **Input**: `{{ $json.id }}` from trigger
- **Why needed**: The Gmail Trigger often returns empty body fields for forwarded or multipart/mixed emails (especially large ones 500KB+). Re-fetching by ID guarantees complete content.
- **Output format** (with `simple: false`):
  - `from`: `{ value: [{address: "email@example.com", name: "Name"}], text: "\"Name\" <email@example.com>" }`
  - `textPlain` / `textHtml`: Full email body content

#### 3. Has PDF? (IF Node)
- **Type**: `n8n-nodes-base.if` v2.3
- **Condition**: `{{ Object.keys($binary || {}).some(k => ($binary[k].mimeType || '').includes('pdf')) }}`
- **True branch** (output 0): Routes to Extract PDF
- **False branch** (output 1): Routes directly to Classify Email (bypasses extraction)
- **Why needed**: The Extract PDF node errors if no PDF binary exists. This IF node cleanly routes around extraction for emails without PDF attachments.

#### 4. Extract PDF (Extract From File)
- **Type**: `n8n-nodes-base.extractFromFile` v1.1
- **Operation**: `pdf`
- **Binary key**: Dynamic expression finds the first PDF attachment: `{{ Object.keys($binary).find(k => ($binary[k].mimeType || '').includes('pdf')) }}`
- **Options**: `joinPages: true` (single concatenated string), `keepSource: "json"` (preserves original email JSON fields)
- **Output**: All original email fields plus `$json.text` with extracted PDF content

#### 5. Classify Email (Text Classifier)
- **Type**: `@n8n/n8n-nodes-langchain.textClassifier` v1.1
- **Categories**:
  - **PE Deal Marketing** (output 0) — Emails marketing businesses for sale/acquisition
  - **Not PE Deal** (output 1) — Everything else (newsletters, spam, meetings, operational)
  - **Other/fallback** (output 2) — Unclassifiable emails
- **Input text**: Combines `from.text`, `subject`, and `textPlain || textHtml` (prefers plain text, falls back to HTML)
- **LLM**: Gemini Flash (`models/gemini-3-flash-preview`)
- **Note**: Receives input from both the Extract PDF path (with `$json.text`) and the no-PDF path (without it)

#### 6. Skip Non-Deal (No Operation)
- **Type**: `n8n-nodes-base.noOp` v1
- **Connected to**: Classify Email outputs 1 and 2
- **Purpose**: Explicit handling of non-deal emails so they don't silently disappear

#### 7. Analyze & Draft (Basic LLM Chain with Structured Output)
- **Type**: `@n8n/n8n-nodes-langchain.chainLlm` v1.9
- **Config**: `hasOutputParser: true` — enables structured JSON output
- **System prompt**: PE deal screening analyst role
- **Single-pass analysis** — evaluates the deal AND drafts both response emails in one LLM call
- **PDF content**: Conditionally appends extracted PDF text using a try-catch IIFE: `{{ (() => { try { ... $('Extract PDF').item.json.text ... } catch(e) { return ''; } })() }}`
- **Evaluates against criteria**:
  - Target EBITDA: $5M+ preferred
  - Preferred sectors: Industrial services
  - Flexible for interesting sectors with rollup potential
- **LLM**: Gemini Flash (`models/gemini-3-flash-preview`)

**Structured Output Schema** (via `outputParserStructured` v1.3):

| Field | Type | Description |
|-------|------|-------------|
| `dealName` | string | Name of the deal, fund, or company |
| `ebitda` | string | Estimated EBITDA or "Not disclosed" |
| `sector` | string | Industry/sector |
| `rollupPotential` | string | Assessment of rollup/platform potential |
| `recommendation` | string | `INTERESTED` or `PASS` |
| `reasoning` | string | Brief reasoning for the recommendation |
| `ndaDraft` | string | Draft email body requesting the NDA |
| `passDraft` | string | Draft email body graciously declining |

#### 8. Review Decision (Gmail sendAndWait)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `sendAndWait`
- **Approval type**: `double` (Approve + Reject buttons)
- **Email content**: Styled HTML with structured AI analysis table, both draft emails, and instructions
- **Output**: `{ data: { approved: true } }` or `{ data: { approved: false } }`

#### 9. Pursue or Pass? (If Node)
- **Type**: `n8n-nodes-base.if` v2.3
- **Condition**: `$json.data.approved` is `true` (boolean check)
- **True branch** (output 0): Routes to Send NDA Request
- **False branch** (output 1): Routes to Send Pass

#### 10. Send NDA Request / Send Pass (Gmail Send with Threading)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `send` (not reply — for proper cross-client threading)
- **Threading**: Uses `threadId` from original email + `Re:` subject prefix
- **Recipient**: RFC 5322 format with full name: `{{ $('Fetch Full Email').item.json.from.value[0].name + ' <' + $('Fetch Full Email').item.json.from.value[0].address + '>' }}`
- **Message body**: Draft text + quoted original email in `<blockquote>` with robust fallback: `textHtml || html || textPlain || text`
- **appendAttribution**: `false` (no n8n footer on professional emails)

## Credentials Required

| Credential | Type | Used By |
|-----------|------|---------|
| Gmail OAuth2 | `gmailOAuth2` | New Email, Fetch Full Email, Review Decision, Send NDA Request, Send Pass |
| Google Gemini API | `googlePalmApi` | Gemini (Classifier), Gemini (Analyzer) |

## Expression Reference

Key expressions used throughout the workflow:

| Expression | Used In | Purpose |
|-----------|---------|---------|
| `{{ $json.id }}` | Fetch Full Email | Get message ID from trigger |
| `Object.keys($binary \|\| {}).some(...)` | Has PDF? | Check if any PDF attachment exists |
| `Object.keys($binary).find(...)` | Extract PDF | Find the binary key of the first PDF |
| `{{ $json.textPlain \|\| $json.textHtml }}` | Classify Email | Email body (plain preferred, HTML fallback) |
| `{{ $('Fetch Full Email').item.json.from.text }}` | Analyze & Draft, Review | Original sender display name |
| `from.value[0].name + ' <' + from.value[0].address + '>'` | Send NDA/Pass | RFC 5322 formatted recipient |
| `{{ $('Fetch Full Email').item.json.threadId }}` | Send NDA/Pass | Thread the reply into the original conversation |
| `$('Extract PDF').item.json.text` (try-catch) | Analyze & Draft | Conditionally include PDF content |
| `{{ $('Analyze & Draft').item.json.output.* }}` | Review, Send nodes | Structured LLM output fields |
| `{{ $json.data.approved }}` | Pursue or Pass? | User's approval decision (boolean) |
| `textHtml \|\| html \|\| textPlain \|\| text` | Send NDA/Pass | Robust blockquote fallback chain |

## Architecture Decisions

### PDF extraction with IF guard
The Extract PDF node errors when no PDF binary exists. Rather than relying on `onError: continueRegularOutput` (which can produce unexpected output items), a "Has PDF?" IF node cleanly routes emails without PDFs directly to classification. The Analyze & Draft prompt uses a try-catch IIFE to safely reference the Extract PDF output, returning empty string when the node didn't run.

### Extract PDF before classification (not after)
The text classifier node strips binary data from its output items. If PDF extraction happened after classification, the PDF attachments would be lost. Extracting before classification preserves the text as a JSON field that flows through the entire pipeline.

### Single-pass LLM analysis (3 calls to 1)
The previous version made 3 sequential LLM calls: evaluate deal, draft NDA, draft pass. Since none depend on each other's output, they are now consolidated into a single call with structured output parsing. This reduces latency by ~60% and halves token costs.

### Gemini Flash only
All LLM nodes use `models/gemini-3-flash-preview`. Flash is sufficient for deal classification, evaluation, and email drafting — no need for Pro.

### Gmail Send with threadId (not Reply)
Gmail's `reply` operation does not include the original message in the body and has limited threading control. Using `send` with explicit `threadId` and `Re:` subject prefix ensures:
- Proper threading across all email clients (Gmail, Outlook, Apple Mail)
- Original message quoted in the reply body via `<blockquote>`

### RFC 5322 sendTo format
Using `name <address>` format (e.g., "John Smith <john@example.com>") instead of bare email addresses ensures Gmail displays the recipient's full name in sent mail.

### textPlain || textHtml
LLM inputs use `textPlain || textHtml` instead of concatenating both. Plain text and HTML typically contain the same content — sending both doubles token usage for no benefit.

### Robust blockquote fallback chain
Reply blockquotes use `textHtml || html || textPlain || text` to handle variations in Gmail's field naming across different email types.

## Issues Encountered & Solutions

### 1. Empty email body from Gmail Trigger
**Problem**: Forwarded emails with multipart/mixed MIME encoding (500KB+) returned empty `textPlain` and `textHtml` fields from the Gmail Trigger.
**Solution**: Added a Gmail Get node ("Fetch Full Email") after the trigger that re-fetches the complete email by message ID with `simple: false`.

### 2. Approved reviews sending wrong email
**Problem**: Clicking "Approve" sent the pass email instead of the NDA request.
**Root cause**: The If node was checking `$json.response` equals `"Approve"` (string), but Gmail sendAndWait with `approvalType: "double"` outputs `{ data: { approved: true } }` — a boolean nested under `data`.
**Solution**: Changed the If node condition to check `$json.data.approved` as a boolean.

### 3. Self-email feedback loop risk
**Problem**: The sendAndWait review email arrives as an unread email, potentially retriggering the workflow in an infinite loop.
**Solution**: Add a sender filter to the Gmail Trigger to exclude your own email address. (TODO: configure with your actual email)

### 4. Extract PDF fails when no PDF attached
**Problem**: The Extract PDF node errors with "binary file 'no_pdf' not found" when emails have no PDF attachments.
**Root cause**: The dynamic binary key expression falls back to `'no_pdf'` which doesn't exist. `onError: continueRegularOutput` didn't cleanly pass through original data.
**Solution**: Added a "Has PDF?" IF node before Extract PDF that checks for PDF binary existence. Emails without PDFs bypass extraction entirely.

### 5. Reply emails show only email address
**Problem**: Sent replies displayed only the raw email address without the sender's name.
**Root cause**: `sendTo` used `from.value[0].address` (just the email).
**Solution**: Changed to RFC 5322 format: `from.value[0].name + ' <' + from.value[0].address + '>'`.

### 6. Quoted thread truncated in replies
**Problem**: The blockquoted original message in replies was incomplete.
**Root cause**: Field names may vary (`textHtml`/`html`/`textPlain`/`text`) depending on email type.
**Solution**: Updated blockquote expression with robust fallback chain: `textHtml || html || textPlain || text`.

## Recommended Improvements

1. **Error handling**: Add error workflow or try/catch for Gemini API failures
2. **Deal tracking**: Add a database or spreadsheet node to log all screened deals and decisions
3. **NDA follow-up**: Create a second workflow to monitor for NDA replies and trigger next steps
4. **Rate limiting**: Add batch processing for email bursts to avoid Gemini API rate limits
5. **Multi-PDF support**: Extract text from all PDF attachments, not just the first one
