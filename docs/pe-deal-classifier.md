# PE Deal Email Classifier & NDA Request - Documentation

## Overview

An n8n workflow that automates private equity deal email screening. It classifies inbound emails, evaluates deals against investment criteria, drafts response emails, and presents everything in a human-in-the-loop review before sending.

## Workflow Flow

```
New Email --> Fetch Full Email --> Classify Email --> Evaluate Deal --> Draft NDA Request --> Draft Pass Email --> Review Decision --> Pursue or Pass? --> Send Reply
```

### Node-by-Node Breakdown

#### 1. New Email (Gmail Trigger)
- **Type**: `n8n-nodes-base.gmailTrigger` v1.3
- **Config**: Polls every minute for unread emails
- **Output**: Basic email metadata including message `id`
- **Note**: Does NOT reliably extract email body for forwarded/multipart MIME emails

#### 2. Fetch Full Email (Gmail Get)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `get`
- **Config**: `simple: false` to get both `textPlain` and `textHtml` fields
- **Input**: `{{ $json.id }}` from trigger
- **Why needed**: The Gmail Trigger often returns empty body fields for forwarded or multipart/mixed emails (especially large ones 500KB+). Re-fetching by ID guarantees complete content.

#### 3. Classify Email (Text Classifier)
- **Type**: `@n8n/n8n-nodes-langchain.textClassifier` v1.1
- **Categories**:
  - **PE Deal Marketing** - Emails marketing businesses for sale/acquisition (deal teasers, CIM offers, NDA requests)
  - **Not PE Deal** - Everything else (newsletters, spam, meetings, operational)
- **Input text**: Combines `from`, `subject`, `textPlain`, and `textHtml` fields
- **Fallback**: Routes to "other" category
- **LLM**: Gemini Flash (`models/gemini-3-flash-preview`)

#### 4. Evaluate Deal (Basic LLM Chain)
- **Type**: `@n8n/n8n-nodes-langchain.chainLlm` v1.9
- **System prompt**: PE deal screening analyst role
- **Evaluates against criteria**:
  - Target EBITDA: $5M+ preferred
  - Preferred sectors: Industrial services
  - Flexible for interesting sectors with rollup potential
- **Output fields**: Deal name, EBITDA, sector, rollup potential, recommendation (INTERESTED/PASS)
- **LLM**: Gemini Flash (`models/gemini-3-flash-preview`)

#### 5. Draft NDA Request (Basic LLM Chain)
- **Type**: `@n8n/n8n-nodes-langchain.chainLlm` v1.9
- **System prompt**: PE investor assistant, professional tone
- **Output**: Email body text only (no subject/greeting/signature)
- **LLM**: Gemini Pro (`models/gemini-3-pro-preview`)

#### 6. Draft Pass Email (Basic LLM Chain)
- **Type**: `@n8n/n8n-nodes-langchain.chainLlm` v1.9
- **System prompt**: Gracious decline, non-specific reasoning, 3-4 sentences max
- **Output**: Email body text only
- **LLM**: Gemini Pro (`models/gemini-3-pro-preview`)

#### 7. Review Decision (Gmail sendAndWait)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `sendAndWait`
- **Approval type**: `double` (Approve + Reject buttons)
- **Email content**: HTML with AI recommendation, both draft emails, and instructions
- **Output**: `{ data: { approved: true } }` or `{ data: { approved: false } }`

#### 8. Pursue or Pass? (If Node)
- **Type**: `n8n-nodes-base.if` v2.3
- **Condition**: `$json.data.approved` is `true` (boolean check)
- **True branch** (output 0): Routes to Send NDA Request
- **False branch** (output 1): Routes to Send Pass

#### 9. Send NDA Request / Send Pass (Gmail Reply)
- **Type**: `n8n-nodes-base.gmail` v2.2
- **Operation**: `reply`
- **Replies to**: Original email thread via `$('Fetch Full Email').item.json.id`
- **Message**: The pre-drafted text from the corresponding LLM Chain node

## Credentials Required

| Credential | Type | Used By |
|-----------|------|---------|
| Gmail OAuth2 | `gmailOAuth2` | New Email, Fetch Full Email, Review Decision, Send NDA Request, Send Pass |
| Google Gemini API | `googlePalmApi` | Gemini (Classifier), Gemini (Evaluator), Gemini (Drafter), Gemini (Pass) |

## Expression Reference

Key expressions used throughout the workflow:

| Expression | Used In | Purpose |
|-----------|---------|---------|
| `{{ $json.id }}` | Fetch Full Email | Get message ID from trigger |
| `{{ $json.from }}` | Classify Email | Sender address |
| `{{ $json.subject }}` | Classify Email | Email subject line |
| `{{ $json.textPlain }}` | Classify Email, LLM nodes | Plain text body |
| `{{ $json.textHtml }}` | Classify Email, LLM nodes | HTML body |
| `{{ $('Fetch Full Email').item.json.from }}` | LLM nodes, Review | Reference original email sender |
| `{{ $('Fetch Full Email').item.json.id }}` | Send NDA/Pass | Reply to original thread |
| `{{ $('Evaluate Deal').item.json.text }}` | Review Decision | AI recommendation text |
| `{{ $('Draft NDA Request').item.json.text }}` | Review, Send NDA | NDA request draft |
| `{{ $('Draft Pass Email').item.json.text }}` | Review, Send Pass | Pass email draft |
| `{{ $json.data.approved }}` | Pursue or Pass? | User's approval decision |

## Issues Encountered & Solutions

### 1. Empty email body from Gmail Trigger
**Problem**: Forwarded emails with multipart/mixed MIME encoding (500KB+) returned empty `textPlain` and `textHtml` fields from the Gmail Trigger.

**Root cause**: The Gmail Trigger's `simple:true` mode cannot reliably extract body content from complex MIME structures.

**Solution**: Added a Gmail Get node ("Fetch Full Email") after the trigger that re-fetches the complete email by message ID with `simple: false`.

### 2. Approved reviews sending wrong email
**Problem**: Clicking "Approve" on the review email sent the pass email instead of the NDA request.

**Root cause**: The If node was checking `$json.response` equals `"Approve"` (string comparison), but Gmail sendAndWait with `approvalType: "double"` outputs `{ data: { approved: true } }` - a boolean nested under `data`, not a string `response` field. Since `$json.response` didn't exist, the condition always evaluated to false.

**Solution**: Changed the If node condition to check `$json.data.approved` as a boolean (`true`/`false`).

### 3. Gmail Trigger typeVersion mismatch
**Problem**: Validation errors about outdated node versions.

**Solution**: Updated Gmail nodes from v2.1 to v2.2, If node from v2.2 to v2.3.

### 4. If node missing condition metadata
**Problem**: If node validation failed with missing `conditions.options.version` and `typeValidation`.

**Solution**: Added `"version": 3` and `"typeValidation": "strict"` to the conditions options.

### 5. Code node property name
**Problem**: An intermediate attempt to use a Code node for MIME parsing used `jsCodeV2` instead of the correct property name.

**Solution**: The correct property is `jsCode` (checked via `get_node` schema). Ultimately this approach was replaced by the Gmail Get node.

## Recommended Improvements

1. **Self-email filter**: Add `-from:your@email.com` to the Gmail Trigger filter to prevent classifying your own sendAndWait review emails
2. **Error handling**: Add error workflow or try/catch for Gemini API failures
3. **Deal tracking**: Add a database or spreadsheet node to log all screened deals and decisions
4. **NDA follow-up**: Create a second workflow to monitor for NDA replies and trigger next steps
