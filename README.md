# Jira-AI

Bilingual AI chatbot widget for the University of Bahrain website (`uob.edu.bh`) with automatic tech support ticketing via Jira.

## What It Does

```
Student asks a question on uob.edu.bh
                |
                v
        AI tries to answer
         (FAQ / RAG search)
                |
        +-------+-------+
        |               |
     Answer found    No answer
        |               |
        v               v
   Bot responds     n8n sends to Jira
                        |
                        v
                  Ticket created
                  Tech Support team sees it
```

**Two jobs:**

1. **Answer questions** (primary) — academic calendar, regulations, deadlines, GPA rules
2. **Tech support ticketing** (secondary) — when the AI can't answer, it sends the message to n8n. n8n handles everything: decides if it's a ticket, categorizes it, and creates it in Jira

## How It Works

### The Widget

A chat widget embedded on `uob.edu.bh` (WordPress). Students see a floating chat button on every page — click it, ask a question, get an answer instantly.

```
+----------------------------------+
|  uob.edu.bh (any page)          |
|                                  |
|  [Page content...]               |
|                                  |
|                    +----------+  |
|                    | Chat     |  |
|                    | Widget   |  |
|                    |          |  |
|                    | Ask me   |  |
|                    | anything |  |
|                    +----------+  |
+----------------------------------+
```

### The AI (UOB-AI)

The chatbot backend ([UOB-AI](https://github.com/xAbdullaShaker/UOB-AI)) handles all the intelligence:

- Bilingual (Arabic + English) with Gulf dialect support
- FAQ matching (49 Q&A pairs) for instant answers
- RAG pipeline for regulation and calendar questions
- Streams responses in real-time via SSE

### The Ticketing (n8n + Jira)

When the AI **can't answer**, it sends the message to n8n. **n8n does all the logic:**

```
Chatbot can't answer
        |
        v
   Sends message to n8n (webhook)
        |
        v
   n8n decides:
   - Is it a tech issue? → Create Jira ticket
   - What category? → Labels it (bug, access, outage, etc.)
   - What priority? → Sets it (critical, high, medium, low)
   - Creates the ticket in Jira
   - Returns ticket ID to chatbot
        |
        v
   Tech Support Team sees the ticket
```

**The chatbot code stays simple** — it only answers questions. All ticketing logic lives in n8n workflows, which you can edit visually without touching code.

### What Does NOT Get a Ticket

| Issue | What happens instead |
|-------|---------------------|
| "When is exam week?" | Bot answers from FAQ |
| "How do I change my major?" | Bot answers from regulations |
| "I want to pay fees" | Bot redirects to Finance Office |
| "What's my GPA?" | Bot redirects to student portal |

## Architecture

```mermaid
graph TB
    subgraph Student["Student Browser"]
        UOB[uob.edu.bh]
        Widget[Chat Widget]
    end
    
    subgraph AWS["AWS (me-south-1 Bahrain)"]
        Nginx[Nginx :443]
        FastAPI[UOB-AI Chatbot\nFastAPI :8005]
        n8n[n8n :5678]
        Aurora[(Aurora PostgreSQL\npgvector)]
    end
    
    Jira[Jira Cloud]
    
    UOB --> Nginx
    Widget -->|HTTPS| Nginx
    Nginx --> FastAPI
    FastAPI -->|vector search| Aurora
    FastAPI -->|webhook| n8n
    n8n -->|create ticket| Jira
    n8n -->|ticket ID| FastAPI

    style Widget fill:#4CAF50,color:#fff
    style FastAPI fill:#2196F3,color:#fff
    style n8n fill:#FF6D00,color:#fff
    style Aurora fill:#FF9800,color:#fff
    style Jira fill:#0052CC,color:#fff
```

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Chatbot** | Python, FastAPI, OpenAI GPT-4.1 | AI backend |
| **Frontend** | React, Vite | Chat UI (widget) |
| **Database** | Amazon Aurora PostgreSQL + pgvector | Embeddings & ticket tracking |
| **Automation** | n8n (self-hosted) | Webhook middleware |
| **Ticketing** | Jira Cloud | Tech support tickets |
| **Web Server** | Nginx | SSL, reverse proxy |
| **Hosting** | AWS EC2 (me-south-1 Bahrain) | Everything runs here |

## Repo Structure

```
Jira-AI/
|-- README.md                    # You're here
|-- PLAN.md                      # Detailed implementation plan
|-- ARCHITECTURE.md              # Mermaid diagrams
|
|-- n8n/
|   |-- docker-compose.yml       # Run n8n
|   |-- .env.example             # Credentials template
|   |-- workflows/
|       |-- escalation.json      # Chatbot -> Jira ticket
|       |-- status-check.json    # Student checks ticket status
|       |-- jira-notify.json     # Jira update -> notification
|
|-- chatbot-plugin/
|   |-- n8n_client.py            # Sends webhooks to n8n (simple POST)
|   |-- models.py                # Data models
|
|-- widget/
|   |-- uob-chat.js              # Embed script for WordPress
|   |-- widget.css               # Floating button styles
|
|-- db/
|   |-- migration.sql            # Aurora schema
|
|-- tests/
    |-- test_n8n_client.py
```

## How n8n Works Here

n8n runs on the same server as the chatbot. The chatbot code does **zero ticketing logic** — n8n handles everything:

**1. Chatbot can't answer a question**

**2. Chatbot sends the raw message to n8n:**
```json
{
  "session_id": "uuid-abc-123",
  "message": "البوابة ما تفتح معي، يطلع خطأ 500",
  "language": "ar",
  "page_url": "https://www.uob.edu.bh/student-portal"
}
```

**3. n8n does all the thinking (visual workflow):**
```
[Webhook]         receives the message
     |
     v
[Classify]        is it a tech issue? what category? what priority?
     |
     v
[Create Jira]     creates the ticket with the right labels
     |
     v
[Respond]         sends ticket ID back to chatbot
```

**4. Chatbot tells the student:**
```
"تم فتح تذكرة UOB-456. فريق الدعم التقني بيتابعون الموضوع."
```

All tickets go to **one Tech Support team**. Labels (outage, access, bug, etc.) help them prioritize.

**Why this is better:** If you want to change how tickets are categorized, change the priority rules, or add Slack notifications — you edit the n8n workflow visually. No code changes, no redeployment.

## Example Conversations

### Bot answers (no ticket needed)

```
Student: When does spring semester start?
Bot:     Spring semester 2025/2026 begins on January 25, 2026.
```

### Tech issue -> ticket created

```
Student: البوابة ما تفتح معي، يطلع خطأ 500
Bot:     يبدو إن فيه مشكلة تقنية في البوابة.
         تم فتح تذكرة رقم UOB-456
         فريق الدعم التقني بيتابعون الموضوع.
```

### Non-tech issue -> redirect

```
Student: I paid my fees but it still shows unpaid
Bot:     For payment issues, please contact the Finance Office:
         - Visit: Administration Building, Ground Floor
         - Email: finance@uob.edu.bh
```

## Related Repos

- [UOB-AI](https://github.com/xAbdullaShaker/UOB-AI) — The chatbot backend (FAQ + RAG pipeline)

## Docs

- [PLAN.md](PLAN.md) — Full implementation plan with phases, code examples, and security details
- [ARCHITECTURE.md](ARCHITECTURE.md) — All system diagrams (Mermaid)
