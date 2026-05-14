# Jira-AI: UOB Chatbot + Jira Integration Plan

## What Are We Building?

We're adding a **customer service layer** to the existing UOB-AI chatbot. Right now, the chatbot answers academic questions (calendar, regulations). But when it **can't help** or the student **needs human support**, there's nowhere to go.

**The goal:** When the chatbot can't solve a student's problem, it automatically creates a Jira ticket and connects the student to a human support agent — all through n8n as the middleware.

---

## Why This Architecture?

### Why n8n? (and not direct Jira API calls)

| Approach | Pros | Cons |
|----------|------|------|
| **Direct Jira API in Python** | Simple, no extra tools | Tightly coupled, hard to change, no visual flow |
| **n8n as middleware** | Visual workflows, easy to modify, handles retries, logs everything | Extra service to run |
| **Zapier/Make** | Easy setup | Costs money, not self-hosted, data leaves your control |

**We chose n8n because:**
1. **Free & self-hosted** — student data stays on your server (important for university)
2. **Visual workflows** — non-developers can modify the flow later
3. **500+ integrations** — tomorrow you can add Slack, Email, Teams without code changes
4. **Retry & error handling** — if Jira is down, n8n queues and retries automatically
5. **Decoupled** — the chatbot doesn't need to know Jira exists; n8n handles the bridge

---

## System Architecture

### High-Level Overview

```
+------------------+       +------------------+       +------------------+
|                  |       |                  |       |                  |
|    Student       |       |    UOB-AI        |       |    n8n           |
|    (Browser)     +------>+    Chatbot       +------>+    Middleware    |
|                  |  SSE  |    (FastAPI)     | HTTP  |    (Workflows)  |
|                  |<------+                  |       |                  |
+------------------+       +--------+---------+       +--------+---------+
                                    |                          |
                                    |                          |
                           +--------v---------+       +--------v---------+
                           |                  |       |                  |
                           |   Supabase       |       |   Jira Cloud     |
                           |   (PostgreSQL)   |       |   (Atlassian)    |
                           |                  |       |                  |
                           +------------------+       +------------------+
```

### Detailed Data Flow

```
Student asks: "I paid my fees but it still shows unpaid"
        |
        v
+---------------------------------------+
|           UOB-AI Chatbot              |
|                                       |
|  1. Sanitize input                    |
|  2. Detect language (Arabic)          |
|  3. Search FAQ (no match)             |
|  4. Search RAG (no relevant context)  |
|  5. AI classifies as SUPPORT_NEEDED   |
|                                       |
|  Response to student:                 |
|  "I can't help with payment issues.   |
|   I'm creating a support ticket for   |
|   you. A human agent will follow up." |
|                                       |
|  6. Send webhook to n8n ----------+   |
+---------------------------------------+
                                    |
                                    v
+---------------------------------------+
|              n8n Workflow              |
|                                       |
|  Trigger: Webhook received            |
|       |                               |
|       v                               |
|  Classify ticket priority             |
|  (payment = HIGH)                     |
|       |                               |
|       v                               |
|  Create Jira issue                    |
|  - Project: UOB-SUPPORT               |
|  - Type: Service Request              |
|  - Priority: High                     |
|  - Labels: [payment, arabic]          |
|  - Description: conversation context  |
|       |                               |
|       v                               |
|  Send confirmation back to chatbot    |
|  (ticket ID: UOB-123)                |
|       |                               |
|       v                               |
|  (Optional) Notify agent on Slack     |
|  (Optional) Send email to student     |
+---------------------------------------+
                                    |
                                    v
+---------------------------------------+
|           Jira Cloud                  |
|                                       |
|  New issue: UOB-123                   |
|  Status: Open                         |
|  Assigned to: Support Team            |
|  Student can check status via chatbot |
+---------------------------------------+
```

---

## The Three Workflows

### Workflow 1: Escalation (Chatbot -> Jira)

**When:** The chatbot can't answer OR the student asks for human help

```
+-------------+     +-----------+     +------------+     +----------+     +-----------+
|  Webhook    |     | Classify  |     | Create     |     | Notify   |     | Reply to  |
|  from       +---->+ Priority  +---->+ Jira       +---->+ Support  +---->+ Chatbot   |
|  Chatbot    |     | & Route   |     | Ticket     |     | Team     |     | (ticket#) |
+-------------+     +-----------+     +------------+     +----------+     +-----------+
```

**Webhook payload (Chatbot sends this to n8n):**
```json
{
  "session_id": "uuid-abc-123",
  "student_message": "I paid my fees but it still shows unpaid",
  "language": "en",
  "conversation_history": [
    {"role": "user", "content": "I paid my fees but it still shows unpaid"},
    {"role": "assistant", "content": "I can help with academic questions..."}
  ],
  "category": "payment_issue",
  "timestamp": "2026-05-14T10:30:00Z"
}
```

**n8n creates Jira ticket:**
```json
{
  "fields": {
    "project": {"key": "UOBSUP"},
    "issuetype": {"name": "Service Request"},
    "summary": "Payment issue - fees showing unpaid after payment",
    "description": "Student reports paying fees but system shows unpaid.\n\nConversation:\n- Student: I paid my fees but it still shows unpaid\n- Bot: I can help with academic questions...\n\nLanguage: English\nSession: uuid-abc-123",
    "priority": {"name": "High"},
    "labels": ["payment", "chatbot-escalation", "english"]
  }
}
```

---

### Workflow 2: Status Check (Student asks about ticket)

**When:** Student asks "What's the status of my ticket?" or "UOB-123 شنو وضعه؟"

```
+-------------+     +------------+     +----------+     +-------------+
|  Webhook    |     | Extract    |     | Fetch    |     | Reply to    |
|  from       +---->+ Ticket ID  +---->+ from     +---->+ Chatbot     |
|  Chatbot    |     | from msg   |     | Jira API |     | (status)    |
+-------------+     +------------+     +----------+     +-------------+
```

**Response back to chatbot:**
```json
{
  "ticket_id": "UOB-123",
  "status": "In Progress",
  "assignee": "Ahmed (Finance Dept)",
  "last_update": "2026-05-14",
  "comment": "We're verifying the payment with the bank"
}
```

---

### Workflow 3: Jira Update -> Student Notification

**When:** A support agent updates the Jira ticket

```
+-------------+     +------------+     +-------------+     +-------------+
|  Jira       |     | Format     |     | Find        |     | Send        |
|  Webhook    +---->+ Update     +---->+ Student     +---->+ Notification|
|  (on change)|     | Message    |     | Session     |     | (email/SMS) |
+-------------+     +------------+     +-------------+     +-------------+
```

---

## How the Chatbot Decides to Escalate

We need to add a new routing step in the existing UOB-AI pipeline:

```
Current Pipeline:
  User Query -> Sanitize -> FAQ Search -> RAG Search -> Respond

New Pipeline:
  User Query -> Sanitize -> FAQ Search -> RAG Search
                                              |
                                    +---------+---------+
                                    |                   |
                              Has answer?          No answer?
                                    |                   |
                                    v                   v
                               Respond           Escalation Check
                                                        |
                                              +---------+---------+
                                              |                   |
                                        Student asks         Bot can't
                                        for human help       answer
                                              |                   |
                                              v                   v
                                         Create ticket      Offer to create
                                         immediately        ticket (ask first)
```

### Trigger Keywords (examples):

**English:**
- "I want to talk to someone"
- "human agent please"
- "this isn't helping"
- "I need help with [payment/registration/grades]"
- "create a ticket"

**Arabic:**
- "ابي اكلم شخص"
- "ابي موظف"
- "ما استفدت"
- "ساعدوني في [الدفع/التسجيل/الدرجات]"
- "افتحوا لي تذكرة"

### Categories for Auto-Routing:

| Category | Priority | Jira Label | Assigned To |
|----------|----------|------------|-------------|
| Payment/Fees | High | `payment` | Finance Team |
| Registration Issues | High | `registration` | Registrar |
| Grade Disputes | Medium | `grades` | Academic Affairs |
| IT/System Issues | Medium | `it-support` | IT Department |
| General Inquiry | Low | `general` | Student Services |
| Complaints | High | `complaint` | Quality Assurance |

---

## Tech Stack for Jira-AI

```
+--------------------------------------------------+
|                   Jira-AI Repo                    |
|                                                   |
|  +--------------------+  +---------------------+ |
|  |  Chatbot Changes   |  |  n8n Configuration  | |
|  |  (Python module)   |  |  (JSON workflows)   | |
|  |                    |  |                     | |
|  |  - escalation.py   |  |  - escalation.json  | |
|  |  - jira_client.py  |  |  - status.json      | |
|  |  - categories.py   |  |  - notify.json      | |
|  +--------------------+  +---------------------+ |
|                                                   |
|  +--------------------+  +---------------------+ |
|  |  n8n Setup         |  |  Jira Setup         | |
|  |                    |  |                     | |
|  |  - docker-compose  |  |  - project config   | |
|  |  - env template    |  |  - issue types      | |
|  +--------------------+  |  - workflows         | |
|                          +---------------------+ |
+--------------------------------------------------+
```

### Files We'll Create:

```
Jira-AI/
|-- PLAN.md                      # This file (you're reading it)
|-- README.md                    # Setup guide
|
|-- n8n/
|   |-- docker-compose.yml       # Run n8n locally or on server
|   |-- .env.example             # n8n + Jira credentials template
|   |-- workflows/
|   |   |-- escalation.json      # Workflow 1: chatbot -> Jira
|   |   |-- status-check.json    # Workflow 2: check ticket status
|   |   |-- jira-notify.json     # Workflow 3: Jira -> notification
|
|-- chatbot-plugin/
|   |-- escalation.py            # Detects when to escalate
|   |-- jira_models.py           # Data models for tickets
|   |-- categories.py            # Category & priority mapping
|   |-- n8n_client.py            # Sends webhooks to n8n
|   |-- ticket_store.py          # Maps session_id <-> ticket_id
|   |-- __init__.py
|
|-- jira-setup/
|   |-- project-config.md        # How to set up Jira project
|   |-- issue-types.md           # Custom issue types & fields
|
|-- tests/
|   |-- test_escalation.py       # Unit tests
|   |-- test_categories.py
|   |-- test_n8n_client.py
```

---

## How Everything Connects

```
+------------------------------------------------------------------+
|                        YOUR SERVER                                |
|                                                                   |
|  +------------------+    webhook     +------------------+         |
|  |                  +--------------->+                  |         |
|  |  UOB-AI Chatbot  |               |  n8n             |         |
|  |  (FastAPI :8005) |<---------------+  (Docker :5678)  |         |
|  |                  |   ticket info  |                  |         |
|  +--------+---------+               +--------+---------+         |
|           |                                  |                   |
|           | serves                           | API calls         |
|           |                                  |                   |
+-----------|----------------------------------|-------------------+
            |                                  |
            v                                  v
     +------+------+                   +-------+--------+
     |             |                   |                |
     |  Student    |                   |  Jira Cloud    |
     |  Browser    |                   |  (Atlassian)   |
     |             |                   |                |
     +-------------+                   +----------------+
```

### Port Map:

| Service | Port | Purpose |
|---------|------|---------|
| Nginx | 80 | Public-facing, routes traffic |
| UOB-AI (FastAPI) | 8005 | Chatbot backend |
| n8n | 5678 | Workflow automation (internal) |
| PostgreSQL (Supabase) | 5432 | Chatbot data |

---

## Step-by-Step Implementation Plan

### Phase 1: Setup Infrastructure (Day 1)

- [ ] Set up Jira Cloud project (UOBSUP)
- [ ] Create issue types (Service Request, Bug, Complaint)
- [ ] Configure Jira workflows (Open -> In Progress -> Resolved -> Closed)
- [ ] Deploy n8n via Docker on the same server as UOB-AI
- [ ] Connect n8n to Jira (API token)

### Phase 2: Build Escalation Flow (Day 2-3)

- [ ] Create `escalation.py` — detect when chatbot should escalate
- [ ] Create `categories.py` — classify issue category & priority
- [ ] Create `n8n_client.py` — send webhook to n8n
- [ ] Build n8n Workflow 1 (escalation) — webhook -> Jira ticket
- [ ] Modify `api.py` in UOB-AI to call escalation logic
- [ ] Add new SSE event type: `{"type": "ticket", "id": "UOB-123"}`

### Phase 3: Build Status Check Flow (Day 4)

- [ ] Create `ticket_store.py` — track session <-> ticket mapping
- [ ] Add ticket status intent detection in chatbot
- [ ] Build n8n Workflow 2 (status check) — webhook -> Jira query -> response
- [ ] Display ticket status in chat UI

### Phase 4: Build Notification Flow (Day 5)

- [ ] Configure Jira webhook (on issue update)
- [ ] Build n8n Workflow 3 (notification) — Jira update -> email/notification
- [ ] (Optional) Add real-time notification to chat UI via WebSocket

### Phase 5: Testing & Polish (Day 6-7)

- [ ] Write unit tests for all new modules
- [ ] End-to-end testing (chatbot -> n8n -> Jira -> back)
- [ ] Test Arabic & English flows
- [ ] Test error cases (n8n down, Jira down, invalid input)
- [ ] Update UOB-AI frontend with ticket UI elements

---

## Example Conversations

### Scenario 1: Student needs help with payment (Arabic)

```
Student: دفعت الرسوم بس لين الحين يقول ما دفعت
Bot:     هالموضوع يحتاج متابعة من فريق الدعم.
         تبيني افتح لك تذكرة عشان يتواصلون معك؟

Student: ايه افتح لي
Bot:     تم فتح تذكرة رقم UOB-456
         فريق المالية بيتواصل معك قريب.
         تقدر تسأل عن وضع التذكرة في أي وقت.
```

### Scenario 2: Student checks ticket status (English)

```
Student: What's the status of my ticket UOB-456?
Bot:     Ticket UOB-456:
         Status: In Progress
         Assigned to: Finance Department
         Last update: "Verifying payment with the bank"
         Updated: 2 hours ago
```

### Scenario 3: Student explicitly asks for human (Arabic)

```
Student: ابي اكلم موظف
Bot:     اكيد! شنو الموضوع اللي تحتاج مساعدة فيه؟
         1. دفع / رسوم
         2. تسجيل مواد
         3. درجات
         4. مشاكل تقنية
         5. شي ثاني

Student: 1
Bot:     تم فتح تذكرة UOB-457 لفريق المالية.
         بيتواصلون معك خلال 24 ساعة.
```

---

## Integration Points with UOB-AI

### Changes needed in UOB-AI repo:

**1. `core.py` — Add escalation detection:**
```python
ESCALATION_KEYWORDS_EN = [
    "talk to someone", "human agent", "create ticket",
    "not helping", "real person", "support agent"
]

ESCALATION_KEYWORDS_AR = [
    "ابي اكلم شخص", "ابي موظف", "افتح تذكرة",
    "ما استفدت", "شخص حقيقي", "ابي دعم"
]

SUPPORT_CATEGORIES = [
    "payment", "registration", "grades",
    "it_support", "complaint", "general"
]
```

**2. `api.py` — Add webhook endpoint + escalation route:**
```python
# New endpoint for n8n to call back with ticket info
@app.post("/api/jira/callback")
async def jira_callback(data: dict):
    # n8n sends ticket ID back, we store it for the session
    pass

# In the chat stream handler, add escalation check:
# if should_escalate(message, faq_score, rag_score):
#     send_to_n8n(session_id, message, history, category)
```

**3. Frontend `App.jsx` — Add ticket UI:**
```
- Show ticket creation confirmation
- Show ticket status card
- Add "Talk to Human" button in sidebar
```

---

## Security Considerations

| Risk | Mitigation |
|------|-----------|
| Student data in Jira | Use Jira Cloud (SOC 2 compliant), minimal PII in tickets |
| n8n exposed to internet | Keep n8n on internal port (5678), only accessible from server |
| Webhook abuse | Authenticate webhooks with shared secret token |
| Jira API token leak | Store in `.env`, never commit, rotate regularly |
| Spam tickets | Rate limit ticket creation (max 3 per session per hour) |

---

## Why This Design Works for UOB

1. **Bilingual** — The chatbot already handles Arabic/English; tickets preserve the language
2. **Non-intrusive** — Students don't need to leave the chat; everything happens in the same window
3. **Scalable** — n8n can route to different departments automatically
4. **Trackable** — Every student issue gets a Jira ticket number they can reference
5. **Flexible** — Adding Slack/Email/Teams notifications is just one n8n node away
6. **Self-hosted** — University data stays on university servers
