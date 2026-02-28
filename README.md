# PE Deal Email Classifier & NDA Request

An n8n workflow that automates private equity deal email screening. It classifies inbound emails, evaluates deals against investment criteria, drafts response emails, and presents everything in a human-in-the-loop review before sending.

## How It Works

```
New Email --> Fetch Full Email --> Classify Email --> Evaluate Deal --> Draft NDA Request --> Draft Pass Email --> Review Decision --> Pursue or Pass? --> Send Reply
```

### Flow Overview

1. **New Email** (Gmail Trigger) - Polls for unread emails every minute
2. **Fetch Full Email** (Gmail Get) - Re-fetches the full email body by message ID (handles forwarded/multipart emails that the trigger can't fully parse)
3. **Classify Email** (Text Classifier + Gemini) - AI classifies the email as "PE Deal Marketing" or "Not PE Deal" using extensive example descriptions
4. **Evaluate Deal** (LLM Chain + Gemini) - Screens the deal against investment criteria and provides a recommendation (INTERESTED or PASS)
5. **Draft NDA Request** (LLM Chain + Gemini) - Drafts a professional email requesting the NDA
6. **Draft Pass Email** (LLM Chain + Gemini) - Drafts a polite decline email
7. **Review Decision** (Gmail sendAndWait) - Sends a review email with the AI recommendation and both draft emails, with Approve/Reject buttons
8. **Pursue or Pass?** (If node) - Routes based on user's decision
9. **Send NDA Request** or **Send Pass** (Gmail Reply) - Sends the appropriate reply to the original email thread

### Investment Criteria

- Target EBITDA: $5M+ preferred
- Preferred sectors: Industrial services
- Flexible for interesting sectors with rollup/platform potential

## Architecture

### Nodes (14 total)

| Node | Type | Purpose |
|------|------|---------|
| New Email | Gmail Trigger | Poll for unread emails |
| Fetch Full Email | Gmail (Get) | Fetch complete email body |
| Classify Email | Text Classifier | AI email classification |
| Gemini (Classifier) | Google Gemini Chat Model | LLM for classifier |
| Evaluate Deal | Basic LLM Chain | Deal screening against criteria |
| Gemini (Evaluator) | Google Gemini Chat Model | LLM for evaluator |
| Draft NDA Request | Basic LLM Chain | Draft NDA request email |
| Gemini (Drafter) | Google Gemini Chat Model | LLM for NDA drafter |
| Draft Pass Email | Basic LLM Chain | Draft decline email |
| Gemini (Pass) | Google Gemini Chat Model | LLM for pass drafter |
| Review Decision | Gmail (sendAndWait) | Human review with Approve/Reject |
| Pursue or Pass? | If | Route based on approval |
| Send NDA Request | Gmail (Reply) | Send NDA request reply |
| Send Pass | Gmail (Reply) | Send decline reply |

### AI Models

- **Classifier & Evaluator**: `models/gemini-3-flash-preview`
- **NDA Drafter & Pass Drafter**: `models/gemini-3-pro-preview`

### Key Design Decisions

- **Gmail Get after Trigger**: The Gmail Trigger's `simple:true` mode often returns empty body fields for forwarded/multipart MIME emails. Adding a Gmail Get node to re-fetch by ID ensures the full email body is always available.
- **Both drafts generated upfront**: Both NDA request and pass emails are drafted before the review step, so the user can see exactly what will be sent regardless of their decision.
- **sendAndWait with double approval**: Uses Approve/Reject buttons (not free-text) for clear binary decisions. The output is `{ data: { approved: true/false } }`, checked with a boolean condition.
- **Linear flow**: All AI steps run sequentially in a single path rather than branching early, keeping the workflow simple and ensuring all data is available at the review step.

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Gmail OAuth2 credentials
- Google Gemini API key

### Installation

1. Import `workflows/pe-deal-classifier.json` into your n8n instance
2. Configure Gmail OAuth2 credentials on all Gmail nodes
3. Configure Google Gemini API credentials on all Gemini nodes
4. Update the `sendTo` email address in the "Review Decision" node to your email
5. Activate the workflow

### Credential Setup

The workflow requires two credential types:
- **Gmail OAuth2** - For reading emails, sending reviews, and sending replies
- **Google Gemini API** - For all AI classification, evaluation, and drafting

## Troubleshooting

### Common Issues & Fixes

#### Emails not classified as PE deals
**Cause**: The Gmail Trigger may not extract the full email body for forwarded or multipart MIME emails.
**Fix**: The workflow uses a Gmail Get node after the trigger to re-fetch complete email content. Ensure the Fetch Full Email node has `simple: false` to get both `textPlain` and `textHtml` fields.

#### Approved review sends wrong email (pass instead of NDA request)
**Cause**: The Gmail sendAndWait node with `approvalType: "double"` outputs `{ data: { approved: true } }`, not `{ response: "Approve" }`. If the If node checks the wrong field, it always evaluates to false.
**Fix**: The If node condition must check `$json.data.approved` as a boolean, not `$json.response` as a string.

#### Self-sent review emails getting classified
**Cause**: The workflow processes all unread emails, including the sendAndWait review emails you receive.
**Fix**: Add a filter to the Gmail Trigger: `-from:your@email.com` to exclude emails from your own address.

## File Structure

```
.
├── README.md                              # This file
├── CLAUDE.md                              # Claude Code configuration for n8n development
├── docs/
│   └── pe-deal-classifier.md              # Detailed workflow documentation
└── workflows/
    └── pe-deal-classifier.json            # Exportable n8n workflow (credentials removed)
```

## Development

This workflow was built using [Claude Code](https://claude.com/claude-code) with the [n8n-mcp](https://github.com/czlonkowski/n8n-mcp) MCP server for direct n8n API access and [n8n-skills](https://github.com/czlonkowski/n8n-skills) for node configuration guidance.
