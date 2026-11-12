# 🤖 ITC Assistant – AI Chatbot & RAG System (Module 03)

An intelligent, multilingual AI Assistant built for **IT Community (ITC)** at Saad Dahleb University of Blida (Blida 1). This project fulfills **Module 03 (AI Chatbot)** of the *ITC Full-Stack & AI Challenge*, powered by **n8n**, **Supabase (pgvector)**, **Google Gemini**, and **PostgreSQL**.

---

## 🎯 Completed Modules & Features

- **Module 03: AI Chatbot & RAG System**
  - **Automated Ingestion Pipeline:** Admin sends Markdown knowledge-base files via Telegram, custom JS parser splits documents by headers and extracts metadata (`source`, `id`, `title`, `category`), embeds with OpenAI, and stores them in Supabase Vector Store.
  - **Grounded RAG Retrieval:** Strictly answers using verified club documents (`workshops.md`, `events.md`, `departments.md`, `faq_and_rules.md`).
  - **Multilingual & Darija Support:** Automatically detects and responds in Arabic, Algerian Darija, French, or English.
  - **Persistent Conversation Memory:** Uses PostgreSQL Chat Memory to maintain context across user messages.
  - **Guardrails & Security:** Anti-hallucination rules (replies "I don't know" when context is missing), prompt-injection protection, and zero exposed API keys.
  - **Automated Error Handling & AI Diagnosis:** Dedicated error workflow catches failures, diagnoses the root cause using Gemini, logs it to Postgres, and sends instant alerts via Telegram.

---

## 🏗️ System Architecture

```text
1) INGESTION  (Admin → Knowledge Base)

Admin (Telegram)
      │  Markdown files
      ▼
n8n Ingestion Pipeline
(JS Chunking + OpenAI Embeddings)
      │
      ▼
Supabase Vector Store


2) CHAT  (User → Answer)

User (Chat Widget)
      │
      ▼
n8n AI Agent
 ├──◄►  Supabase Vector Store
 ├──◄►  Google Gemini LLM
 └──◄►  Postgres Chat Memory


3) ERROR HANDLING  (On Failure / Crash)

n8n AI Agent
      │  on failure / crash
      ▼
Error Handler Workflow
      │
      ▼
AI Diagnosis
 ├──►  Postgres Logs
 └──►  Telegram Alert
```

---

## 📂 Repository Structure

```text
ITC-AI-Assistant/
├── workflows/
│   ├── ITC-Assistant.json                # Main RAG Ingestion & AI Chat Agent workflow
│   └── Error-Handler-ITC-Assistant.json  # Automated error monitoring & AI diagnosis workflow
├── knowledge-base/
│   ├── workshops.md                      # Club technical workshops catalog
│   ├── events.md                         # Hackathons, competitions & gatherings
│   ├── departments.md                    # ITC departments structure & joining tasks
│   └── faq_and_rules.md                  # FAQs, code of conduct & assistant guardrails
├── .env.example                          # Environment variables template (no real secrets)
└── README.md                             # Project documentation
```

---

## 🚀 How to Run Locally (Setup Guide)

### Prerequisites:
- A self-hosted or cloud n8n instance.
- A Supabase project with pgvector enabled and a `documents` table.
- A PostgreSQL database for chat memory and `error_logs`.
- API keys for Google Gemini, OpenAI (for embeddings), and a Telegram Bot.

### Environment Setup:
1. Copy `.env.example` to `.env` and fill in your credentials inside the n8n Credentials manager.

### Import Workflows:
1. Open your n8n dashboard -> Workflows -> Import from File.
2. Import both JSON files from the `workflows/` folder and link your credentials.

### Ingest Knowledge Base:
1. Activate the workflow and send the `.md` files from `knowledge-base/` to your Telegram bot to populate Supabase.

### Test the Assistant:
1. Open the Chat Trigger URL in n8n and start asking questions about ITC!
