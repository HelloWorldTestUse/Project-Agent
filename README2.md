# AI Inquiry Agent — Imported Product Support System

An AI agent that receives customer product inquiries in Korean, routes them to the appropriate foreign manufacturer, manages the back-and-forth translation, and delivers the final answer back to the customer.

---

## Overview

Company A distributes imported products and frequently receives customer inquiries that can only be answered by the foreign manufacturer. This agent automates the full inquiry lifecycle — from intake to resolution — across email (Gmail) and chat (KakaoTalk) channels.

```
Customer (KR) → AI Agent → Foreign Company (EN) → AI Agent → Customer (KR)
```

---

## Process Flow

### Step 1 — Customer inquiry received
- Log ticket with auto-generated ID and timestamp
- Detect channel (Gmail or KakaoTalk)
- Detect language (Korean, English, etc.)
- Extract product/order context from message

### Step 2 — Auto-classify inquiry
See [Classification Logic](#classification-logic) below.

### Step 3 — Translate & reformat inquiry
- Translate Korean → English (or target language)
- Reformat as a structured professional query
- Attach product context: model number, purchase date, symptom description

### Step 4 — Send to foreign company
- Route via email or chat based on pre-mapped contact info
- Attach ticket ID for thread tracking
- Set follow-up timer (default: 1 business day)

### Step 5 — Monitor & wait for reply
- Watch inbox/chat for response
- Auto-send polite follow-up if no reply by deadline

### Step 6 — Reply received from foreign company
- Translate reply → Korean
- Assess completeness: does it actually answer the original question?
- If incomplete → return to Step 4 for follow-up round

### Step 7 — Draft reply to customer in Korean
- Adapt tone for customer-facing communication
- Attach any docs, images, or files provided by foreign company
- Flag for human review if needed (see confidence thresholds)

### Step 8 — Human QA & approval *(optional)*
- Required for: warranty claims, safety-related answers, legal/returns statements
- Triggered automatically when AI confidence is below threshold

### Step 9 — Send answer to customer
- Deliver via original channel (Gmail or KakaoTalk)
- Mark ticket as resolved
- Request customer feedback

### Step 10 — Log to knowledge base
- Store resolved Q&A pair (product + question + answer)
- Used by Step 2 to auto-answer repeat inquiries in the future

---

## Classification Logic

Every incoming inquiry passes through six checks in order:

| # | Check | Action if triggered |
|---|-------|-------------------|
| 1 | **Safety risk** — injury, fire, electric shock keywords | Escalate to human immediately |
| 2 | **KB match** — same product + issue answered before | Auto-answer from knowledge base |
| 3 | **Product origin** — not an imported product | Route to Company A internal team |
| 4 | **Inquiry type** — classify into one of four types | Apply matching message template |
| 5 | **Urgency signals** — 빨리, 급하게, 환불, deadline language | Set priority flag, shorten timers |
| 6 | **Confidence score** — AI confidence below threshold | Flag for human review before routing |

### Inquiry types

- `technical_defect` — malfunction, defect, not working
- `usage_how_to` — how to use, setup, instructions
- `parts_compatibility` — spare parts, compatible accessories
- `warranty_returns` — warranty claim, return, refund

---

## Intake Fields

Every ticket captures the following at intake:

### Auto-captured
| Field | Description |
|-------|-------------|
| `channel` | Gmail, KakaoTalk, web form, etc. |
| `ticket_id` | Auto-generated unique ID |
| `timestamp` | Date and time of inquiry |
| `language_detected` | Korean, English, etc. |

### Customer identity
| Field | Description |
|-------|-------------|
| `customer_name` | Name from message or account |
| `contact` | Email address or KakaoTalk ID |
| `order_number` | Purchase order reference |
| `prior_tickets` | Related past inquiries for same customer |

### Product information
| Field | Description |
|-------|-------------|
| `product_name` | Product name as stated by customer |
| `model_number` | SKU or model number |
| `brand` | Brand name — used to look up foreign company contact |
| `country_of_origin` | Identifies which foreign company to contact |
| `purchase_date` | For warranty coverage check |

### Inquiry content
| Field | Description |
|-------|-------------|
| `inquiry_type` | One of the four types above |
| `raw_message` | Verbatim original Korean message (preserved before translation) |
| `attachments` | Photos, error codes, documents |

### Urgency & routing
| Field | Description |
|-------|-------------|
| `urgency_level` | `normal` or `high` |
| `kb_match` | Boolean — similar past case found? |
| `foreign_contact_route` | Pre-mapped email or chat contact for this brand |

### Normalized ticket format
All channels produce this shared format before the agent processes them:

```json
{
  "ticket_id": "TKT-20240415-001",
  "channel": "gmail",
  "customer_id": "...",
  "timestamp": "2024-04-15T09:30:00+09:00",
  "language": "ko",
  "raw_message": "제품이 작동하지 않습니다...",
  "attachments": [],
  "product": {
    "name": "...",
    "model": "...",
    "brand": "...",
    "purchase_date": "..."
  },
  "classification": {
    "type": "technical_defect",
    "urgency": "normal",
    "kb_match": false,
    "confidence": 0.92,
    "route_to_foreign": true
  }
}
```

---

## Channel Integration

### Gmail — native MCP (available now)

Connect the Gmail MCP from Claude settings. The agent gains access to:

- `search_messages` — poll inbox for new inquiries
- `read_thread` — read full conversation thread
- `read_message` — read individual message with attachments
- `create_draft` — draft reply for review or auto-send

**Setup:**
1. Connect Gmail MCP in Claude.ai settings
2. Designate a dedicated inbox (e.g. `inquiries@yourcompany.com`)
3. Configure the agent to poll for unread messages matching product inquiry criteria

### KakaoTalk — custom webhook bridge (build required)

KakaoTalk does not have a native MCP. A custom MCP server is needed:

```
KakaoTalk Channel → Webhook → Custom MCP Server → AI Agent
```

**Architecture:**
1. Customer messages your KakaoTalk Channel (business account)
2. KakaoTalk calls your configured webhook URL with the message payload
3. Your custom MCP server normalizes the payload into the shared ticket format
4. The agent reads from this MCP server identically to how it reads Gmail

**MCP tools to implement:**

```python
get_new_messages()    # poll for unhandled incoming messages
send_reply(id, text)  # send a reply to a specific conversation
mark_handled(id)      # mark a message as processed
get_thread(id)        # retrieve full conversation history
```

**KakaoTalk Business API reference:** https://developers.kakao.com/docs/latest/en/message/rest-api

---

## Knowledge Base

The knowledge base stores resolved Q&A pairs and grows over time. It is checked at Step 2 before any outreach to the foreign company.

**Stored per entry:**
- Product model + brand
- Original Korean question (normalized)
- Translated English question sent to foreign company
- Foreign company's English answer
- Final Korean answer delivered to customer
- Confidence score used at resolution time
- Ticket ID and resolution date

**Confidence thresholds:**
| Score | Action |
|-------|--------|
| ≥ 95% | Auto-send KB answer to customer |
| 80–94% | Draft KB answer, flag for human review |
| < 80% | Do not use KB match, escalate to foreign company |

---

## Human Review Triggers

The agent flags a ticket for human review when any of the following apply:

- Safety or injury keywords detected
- AI classification confidence < 80%
- Inquiry type is `warranty_returns` or involves legal language
- Translation confidence flagged as low by the translation model
- Customer has escalated tone (anger, threat of legal action)
- Follow-up round count exceeds 3 (unresolved after multiple exchanges)

---

## Suggested Tech Stack

| Component | Recommended option |
|-----------|-------------------|
| AI agent runtime | Claude API (claude-sonnet-4-6) |
| Gmail integration | Gmail MCP (native connector) |
| KakaoTalk integration | Custom MCP server (Python/Node.js + KakaoTalk Business API) |
| Translation | Claude API (same model, translation prompt) |
| Knowledge base | PostgreSQL with pgvector, or a vector DB (Pinecone, Weaviate) |
| Ticket queue | Simple DB table or a lightweight queue (Redis, Supabase) |
| Human review UI | Internal web dashboard or Slack notifications |

---

## Project Structure (suggested)

```
inquiry-agent/
├── README.md
├── agent/
│   ├── main.py              # agent entrypoint
│   ├── classify.py          # classification logic (6 checks)
│   ├── translate.py         # Korean ↔ English translation
│   └── kb.py                # knowledge base read/write
├── channels/
│   ├── gmail_mcp.py         # Gmail MCP client wrapper
│   └── kakao_mcp/
│       ├── server.py        # custom MCP server
│       ├── webhook.py       # KakaoTalk webhook receiver
│       └── normalizer.py    # normalize Kakao payload → ticket format
├── templates/
│   ├── technical_defect.txt
│   ├── usage_how_to.txt
│   ├── parts_compatibility.txt
│   └── warranty_returns.txt
└── config/
    ├── brands.json          # brand → foreign company contact mapping
    └── thresholds.json      # confidence and urgency thresholds
```

---

## Configuration

### `config/brands.json`
Maps product brands to their foreign company contact details:

```json
{
  "BrandName": {
    "company": "Foreign Co. Ltd.",
    "email": "support@foreignco.com",
    "language": "en",
    "timezone": "America/New_York",
    "reply_sla_hours": 24
  }
}
```

### `config/thresholds.json`

```json
{
  "kb_auto_answer": 0.95,
  "kb_human_review": 0.80,
  "classification_human_review": 0.80,
  "follow_up_after_hours": 24,
  "max_follow_up_rounds": 3
}
```

---

## Getting Started

1. Clone the repository
2. Connect Gmail MCP in Claude.ai settings
3. Set up KakaoTalk Business API and configure webhook URL
4. Populate `config/brands.json` with your product brand → supplier mappings
5. Initialize the knowledge base schema
6. Set environment variables:
   ```
   ANTHROPIC_API_KEY=...
   KAKAO_CHANNEL_SECRET=...
   KAKAO_ADMIN_KEY=...
   DB_URL=...
   ```
7. Run the agent: `python agent/main.py`

---

## Next Steps

- [ ] Connect Gmail MCP
- [ ] Build KakaoTalk webhook bridge MCP server
- [ ] Populate brand → foreign company contact mapping
- [ ] Set up knowledge base (vector DB recommended)
- [ ] Define human review workflow (Slack notification or dashboard)
- [ ] Write outbound message templates for each inquiry type
- [ ] Set confidence thresholds based on pilot testing
