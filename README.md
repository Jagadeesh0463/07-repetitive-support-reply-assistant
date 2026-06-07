# 07 — Repetitive Support Reply Assistant

> RAG-powered support reply assistant built with n8n — detects customer intent, retrieves answers from a Qdrant vector knowledge base using Gemini embeddings, personalizes replies with Groq LLaMA, and routes unknown queries to human review via Gmail Draft. Includes a PDF knowledge base ingestion pipeline.

---

## Table of Contents

- [Overview](#overview)
- [Two-Workflow Architecture](#two-workflow-architecture)
- [Workflow 07A — AI Support Reply Assistant](#workflow-07a--ai-support-reply-assistant)
- [Workflow 07B — Support KB Ingestion Pipeline](#workflow-07b--support-kb-ingestion-pipeline)
- [Node Breakdown — 07A](#node-breakdown--07a)
- [Node Breakdown — 07B](#node-breakdown--07b)
- [Human-in-the-Loop Logic](#human-in-the-loop-logic)
- [RAG Architecture](#rag-architecture)
- [Tech Stack](#tech-stack)
- [Setup Instructions](#setup-instructions)
- [Environment Placeholders](#environment-placeholders)
- [Knowledge Base Structure](#knowledge-base-structure)
- [Test Cases](#test-cases)
- [Project Structure](#project-structure)
- [Author](#author)

---

## Overview

Support teams handling repetitive emails face the same problem every day — the same 5–6 questions about pricing, refunds, billing, order status, and account access arrive repeatedly, consuming time that could go toward complex issues.

This two-workflow system solves that by:

- Monitoring Gmail inbox in real time for incoming support emails
- Detecting customer intent automatically using an AI agent
- Searching a company knowledge base (Qdrant vector store) for approved answers
- Personalizing a professional draft reply using Groq LLaMA
- Auto-sending confident replies for repetitive known queries
- Routing unknown or uncertain queries to human review as a Gmail Draft

**Problem statement source:** NexusQ Enterprise SaaS Automation — Problem 7 of 12

**Why this matters:** Getting response times under an hour reduces follow-up emails. Automation works best on repetitive, structured queries first — this workflow targets exactly those.

---

## Two-Workflow Architecture

This solution intentionally uses two separate workflows — one for real-time query handling and one for knowledge base management. This separation means the KB can be updated independently without touching the live reply pipeline.

```
┌─────────────────────────────────────────────────────────────┐
│              WORKFLOW 07B — KB INGESTION                     │
│  (Run once manually whenever KB content is updated)          │
│                                                              │
│  Manual Trigger → Load PDFs → Extract Text                  │
│       → Default Data Loader + Text Splitter                  │
│       → Qdrant Vector Store (insert) + Gemini Embeddings     │
└─────────────────────────────────────────────────────────────┘
                          ↓ populates
                   [ Qdrant: support_kb ]
                          ↓ queried by
┌─────────────────────────────────────────────────────────────┐
│              WORKFLOW 07A — REPLY ASSISTANT                  │
│  (Runs automatically every minute)                           │
│                                                              │
│  Gmail Trigger → Customer Support AI Agent                   │
│       → Qdrant Retrieval (RAG) + Groq LLaMA                 │
│       → Risk Check                                           │
│            ↓ TRUE (unknown)     ↓ FALSE (confident)          │
│       Human Review Draft    Auto Send Reply                  │
│       (Gmail Drafts)        (to customer)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## Workflow 07A — AI Support Reply Assistant

### Flow

```
Gmail Trigger
      ↓
Customer Support AI Agent
      ↓  (uses Qdrant RAG tool + Groq LLaMA)
Risk Check (IF node)
      ↓ TRUE                        ↓ FALSE
(contains "human review"        (confident KB answer)
 or similar phrases)
      ↓                              ↓
Human Review Draft             Send a message
(saved to Gmail Drafts)        (auto-reply to customer)
```

### Routing Logic

| Condition | Path | Action |
|---|---|---|
| AI cannot find answer in KB | TRUE branch | Gmail Draft created for human to review and send |
| AI finds confident answer in KB | FALSE branch | Reply sent automatically to customer |

---

## Workflow 07B — Support KB Ingestion Pipeline

### Flow

```
Start KB Ingestion (Manual)
      ↓
Load Knowledge Base PDFs
      ↓
Extract from File (PDF text extraction)
      ↓ (main)
Qdrant Vector Store (insert mode)
      ↑ (ai_document)           ↑ (ai_embedding)
Default Data Loader       Embeddings Google Gemini
      ↑ (ai_textSplitter)
Recursive Character Text Splitter
(chunkSize: 500, chunkOverlap: 50)
```

### Why manual trigger?

KB ingestion runs on demand — only when new FAQ documents are added or existing ones are updated. Running it automatically would create duplicate vectors and degrade retrieval quality.

---

## Node Breakdown — 07A

### 1. Gmail Trigger
- **Type:** `n8n-nodes-base.gmailTrigger`
- **Poll frequency:** Every minute
- **Filter:** `category:primary -from:mailer-daemon -from:no-reply -from:noreply`
- **Purpose:** Monitors inbox for real customer support emails only — filters out newsletters, bounce emails, and automated no-reply messages to avoid wasting LLM calls

### 2. Groq Chat Model
- **Type:** `lmChatGroq`
- **Model:** `llama-3.1-8b-instant`
- **Purpose:** Language model powering the AI agent for intent detection and reply generation

### 3. Embeddings Google Gemini
- **Type:** `embeddingsGoogleGemini`
- **Model:** `models/gemini-embedding-001`
- **Vector dimensions:** 768
- **Purpose:** Converts customer query into a vector for semantic search against the Qdrant knowledge base

### 4. Qdrant Vector Store (retrieve-as-tool)
- **Type:** `vectorStoreQdrant`
- **Mode:** `retrieve-as-tool`
- **Collection:** `support_kb`
- **Tool description:** "Search company FAQ, billing, pricing, shipping, refund and subscription policies"
- **Purpose:** Acts as a RAG retrieval tool for the agent — searches for semantically similar approved answers from the KB

### 5. Customer Support AI Agent
- **Type:** `langchain.agent`
- **Input:** Subject + full email body (`$json.text || $json.snippet`)
- **Tools:** Qdrant Vector Store (RAG retrieval)
- **Instructions:**
  - Determine customer intent
  - Search the knowledge base
  - Generate a professional support email reply
  - Use ONLY information found in the knowledge base
  - If not found → respond that human review is required
  - Never invent pricing, refund, billing, shipping, or subscription policies
- **Hallucination guard:** Explicit instruction prevents LLM from inventing policy details not in the KB

### 6. Risk Check
- **Type:** `n8n-nodes-base.if`
- **caseSensitive:** false
- **ignoreCase:** true
- **Combinator:** OR
- **Conditions — routes to human review if output contains ANY of:**
  - `human review is required`
  - `requires human review`
  - `unable to find`
  - `not found in knowledge base`
- **True branch:** Human Review Draft
- **False branch:** Send a message (auto-reply)

### 7. Human Review Draft
- **Type:** `n8n-nodes-base.gmail`
- **Resource:** `draft`
- **Subject:** `Re: {original subject}`
- **Message:** Full AI-generated draft reply
- **Purpose:** Creates a Gmail Draft — human opens Gmail, reviews the draft, edits if needed, and sends. This is the cleanest human-in-the-loop pattern — no separate alert email, review and send in one step.

### 8. Send a message
- **Type:** `n8n-nodes-base.gmail`
- **sendTo:** `$node["Gmail Trigger"].json["From"].match(/<(.+)>/)?.[1]` — smart regex extracts actual customer email from the From header, handles both "Name \<email\>" and plain email formats
- **Subject:** `Re: {original subject}` — correct email threading
- **Message:** AI-generated reply from KB retrieval
- **Purpose:** Auto-sends reply directly to customer for high-confidence known queries

---

## Node Breakdown — 07B

### 1. Start KB Ingestion
- **Type:** `n8n-nodes-base.manualTrigger`
- **Purpose:** On-demand ingestion — run manually whenever KB documents are added or updated

### 2. Load Knowledge Base PDFs
- **Type:** `n8n-nodes-base.readWriteFile`
- **fileSelector:** `/home/node/.n8n-files/*.pdf`
- **Purpose:** Reads all PDF files from the n8n files directory — place your FAQ and policy PDFs here before running

### 3. Extract from File
- **Type:** `n8n-nodes-base.extractFromFile`
- **Operation:** PDF
- **Purpose:** Extracts raw text content from each PDF for embedding

### 4. Default Data Loader
- **Type:** `documentDefaultDataLoader`
- **Purpose:** Prepares extracted text as document chunks for the vector store — receives data from the main pipeline and passes chunks to Qdrant

### 5. Recursive Character Text Splitter
- **Type:** `textSplitterRecursiveCharacterTextSplitter`
- **chunkSize:** 500
- **chunkOverlap:** 50
- **Purpose:** Splits large PDF text into overlapping chunks of 500 characters — smaller chunks improve retrieval precision. Overlap ensures context is not lost at chunk boundaries.
- **Connection:** `ai_textSplitter` port → Default Data Loader

### 6. Embeddings Google Gemini
- **Type:** `embeddingsGoogleGemini`
- **Model:** `models/gemini-embedding-001`
- **Vector dimensions:** 768
- **Purpose:** Converts each text chunk into a 768-dimensional vector for semantic search
- **Connection:** `ai_embedding` port → Qdrant Vector Store

### 7. Qdrant Vector Store (insert mode)
- **Type:** `vectorStoreQdrant`
- **Mode:** `insert`
- **Collection:** `support_kb`
- **Purpose:** Stores embedded document chunks in Qdrant — the collection must be pre-created with vector size 768 and Cosine distance metric

---

## Human-in-the-Loop Logic

This workflow implements the pattern described in the problem statement:

> *"AI drafts, humans approve when needed"*

The AI agent determines routing by what it says in its reply:

- **Confident KB answer found** → reply auto-sent to customer immediately
- **KB answer not found** → AI states "human review is required" → Gmail Draft created → human reviews and sends manually

The Gmail Draft approach is intentionally chosen over an alert email — the human reviewer sees the AI-drafted reply directly in Gmail, can edit it in place, and sends with one click. No context switching required.

**Risk Check is case-insensitive** — catches all variations the LLM might use:

| LLM Output Phrase | Detected |
|---|---|
| "human review is required" | ✅ |
| "Human Review is Required" | ✅ |
| "requires human review" | ✅ |
| "unable to find" | ✅ |
| "not found in knowledge base" | ✅ |

---

## RAG Architecture

This is genuine Retrieval-Augmented Generation — not a hardcoded knowledge base.

```
Customer Email
      ↓
Gemini Embedding (768-dim vector)
      ↓
Qdrant Semantic Search → Top-K similar chunks retrieved
      ↓
Retrieved KB chunks passed to LLaMA agent as context
      ↓
LLaMA generates reply grounded in retrieved content only
      ↓
Hallucination guard: "Never invent pricing/billing/refund policies"
```

**Why RAG over hardcoded prompts:**

| Hardcoded KB in system prompt | RAG with Qdrant |
|---|---|
| Fixed at workflow creation | Updated by running 07B |
| Token limit constrained | Unlimited KB size |
| Retrieves everything always | Retrieves only relevant chunks |
| No semantic search | Semantic similarity matching |

---

## Tech Stack

| Tool | Purpose |
|---|---|
| n8n | Workflow automation |
| Groq API | LLM inference (LLaMA 3.1 8B) |
| Google Gemini API | Text embeddings (embedding-001, 768-dim) |
| Qdrant | Vector database for KB storage and retrieval |
| Gmail API | Email trigger, auto-reply, and draft creation |

---

## Setup Instructions

### Prerequisites
- n8n instance (local or cloud) — v2.17.7 or higher
- Groq API account — [console.groq.com](https://console.groq.com)
- Google Gemini API key — [aistudio.google.com](https://aistudio.google.com)
- Qdrant instance (local or cloud) — [qdrant.tech](https://qdrant.tech)
- Gmail account with OAuth2 configured in n8n

### Step 1 — Create Qdrant collection

Before ingesting documents, create the collection manually with correct dimensions:

```
Collection name:  support_kb
Vector size:      768
Distance metric:  Cosine
```

Gemini `embedding-001` produces 768-dimensional vectors. The collection must match exactly — mismatched dimensions cause a 422 error on ingestion.

### Step 2 — Place PDF documents

Copy your FAQ and policy PDF files to:
```
/home/node/.n8n-files/
```

Suggested PDF files:
```
Billing_FAQ.pdf
Refund_Policy.pdf
Pricing_Guide.pdf
Shipping_Policy.pdf
Account_Access_FAQ.pdf
Subscription_FAQ.pdf
```

### Step 3 — Import both workflows

1. Open your n8n instance
2. Click **New Workflow** → **Import from file**
3. Import `07A-ai-support-reply-assistant.json`
4. Repeat for `07B-support-kb-ingestion-pipeline.json`

### Step 4 — Configure credentials

Set up the following credentials in n8n Settings → Credentials:

| Credential | Type | Used by |
|---|---|---|
| Gmail account | Gmail OAuth2 | Gmail Trigger, Human Review Draft, Send a message |
| Groq account | Groq API | Groq Chat Model |
| Qdrant account | Qdrant API | Qdrant Vector Store (both workflows) |
| Google Gemini API | Google PaLM API | Embeddings Google Gemini (both workflows) |

### Step 5 — Replace placeholders

Open each node and replace the following placeholders:

| Placeholder | Replace with |
|---|---|
| `{{GMAIL_CREDENTIAL_ID}}` | Your Gmail credential ID from n8n |
| `{{GROQ_CREDENTIAL_ID}}` | Your Groq credential ID from n8n |
| `{{QDRANT_CREDENTIAL_ID}}` | Your Qdrant credential ID from n8n |
| `{{GEMINI_CREDENTIAL_ID}}` | Your Gemini credential ID from n8n |
| `{{WEBHOOK_ID_1}}` | Auto-generated by n8n on import |
| `{{WEBHOOK_ID_2}}` | Auto-generated by n8n on import |
| `{{N8N_INSTANCE_ID}}` | Auto-generated by n8n on import |

### Step 6 — Run KB ingestion

1. Open Workflow 07B
2. Click **Execute Workflow** (manual trigger)
3. Verify it completes successfully — PDFs are now embedded in Qdrant

### Step 7 — Activate 07A

1. Open Workflow 07A
2. Toggle to **Active**
3. It will begin polling Gmail every minute

---

## Environment Placeholders

All sensitive values have been removed and replaced with placeholder tokens. **Never commit real credentials, API keys, or email addresses to GitHub.**

```
{{GMAIL_CREDENTIAL_ID}}     → n8n Gmail OAuth2 credential ID
{{GROQ_CREDENTIAL_ID}}      → n8n Groq API credential ID
{{QDRANT_CREDENTIAL_ID}}    → n8n Qdrant API credential ID
{{GEMINI_CREDENTIAL_ID}}    → n8n Google Gemini API credential ID
{{WEBHOOK_ID_1}}            → Auto-assigned by n8n on import
{{WEBHOOK_ID_2}}            → Auto-assigned by n8n on import
{{N8N_INSTANCE_ID}}         → Auto-assigned by n8n on import
```

---

## Knowledge Base Structure

Place PDF files in `/home/node/.n8n-files/` covering these topics for best results:

| Topic | Example content |
|---|---|
| Billing | How to update payment method, download invoices, billing cycles |
| Refund | Refund policy window, how to request, processing time |
| Pricing | Plan tiers, features per plan, upgrade/downgrade process |
| Shipping | Delivery timelines, tracking, international shipping |
| Account Access | Password reset, login issues, account recovery |
| Subscription | How to cancel, pause, or modify subscription |

The AI agent is instructed to never invent information not present in the KB — if a topic is not covered, it will automatically route to human review.

---

## Test Cases

Use these emails to verify both routing paths after setup:

### Test 1 — Auto Reply (Password Reset)
```
Subject: How do I reset my password?
Body:    Hi, I forgot my password and cannot log into
         my account. Can you help me reset it?
```
**Expected:** KB finds password reset info → confident answer → Auto reply sent to sender

---

### Test 2 — Auto Reply (Refund Policy)
```
Subject: Refund request - bought 3 days ago
Body:    I purchased the premium plan 3 days ago and
         it is not working as expected.
         I want a full refund within the 30 day window.
```
**Expected:** KB finds refund policy → confident answer → Auto reply sent to sender

---

### Test 3 — Human Review Draft (Unknown query)
```
Subject: Partnership and reseller program inquiry
Body:    Hello, we are interested in becoming a reseller
         partner. We need information about partnership
         tiers, commission structures, and co-marketing
         opportunities.
```
**Expected:** KB has no partnership info → "human review is required" → Draft saved in Gmail Drafts

---

### Test 4 — Human Review Draft (Custom enterprise)
```
Subject: Custom enterprise pricing negotiation
Body:    We are a large enterprise with 500 employees
         and need a custom pricing agreement with
         dedicated support and SLA guarantees.
```
**Expected:** KB has no custom enterprise info → Human review draft created

---

### Test 5 — Filter test (Non-support email)
```
From:    no-reply@newsletter.com
Subject: Weekly digest
```
**Expected:** Gmail filter blocks this — workflow does not trigger

---

## Project Structure

```
07-repetitive-support-reply-assistant/
│
├── 07A-ai-support-reply-assistant.json       ← Query/reply workflow (GitHub-safe)
├── 07B-support-kb-ingestion-pipeline.json    ← KB ingestion workflow (GitHub-safe)
├── 07A-ai-support-reply-assistant.png        ← Workflow 07A screenshot
├── 07B-support-kb-ingestion-pipeline.png     ← Workflow 07B screenshot
└── README.md                                 ← This file
```

---

## Author

**S Jagadeesh**

---
