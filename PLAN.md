# UOB-AI: Chatbot Widget + Jira Integration Plan

## What Are We Building?

A **bilingual AI chatbot widget** that lives on the University of Bahrain website (`uob.edu.bh`) and serves two purposes:

1. **Primary: Answer student questions** — academic calendar, regulations, deadlines, GPA rules (this already works in UOB-AI)
2. **Secondary: Tech support ticketing** — when a student has a technical issue (system down, login problems, portal bugs, app errors), the bot creates a Jira ticket via n8n and routes it to the IT/tech support team

**Jira is a tech support ticketing system here** — not general student services. It handles IT issues, system bugs, access problems, and technical requests. Academic issues like payments, grades, or registration go through other university channels.

The chatbot widget will be **embedded directly on uob.edu.bh** (a WordPress site using the Avada theme), so students don't need to visit a separate page.

---

## The Big Picture

```
uob.edu.bh (WordPress)                    Your Server (AWS)
+---------------------------+              +---------------------------+
|                           |              |                           |
|  [Avada Theme Pages]      |   HTTPS      |  FastAPI Backend (:8005)  |
|                           +------------->+                           |
|  +---------------------+ |              |  - FAQ/RAG pipeline       |
|  | Chat Widget (iframe) | |   SSE       |  - Aurora PostgreSQL      |
|  | or JS embed          |<--------------+  - Escalation logic       |
|  |                      | |              |                           |
|  | "Ask me anything!" | |              +------------+--------------+
|  +---------------------+ |                           |
|                           |                           | webhook
+---------------------------+                           v
                                           +---------------------------+
                                           |  n8n (:5678)              |
                                           |  - Escalation workflow    |
                                           |  - Status check workflow  |
                                           |  - Notification workflow  |
                                           +------------+--------------+
                                                        |
                                                        | API
                                                        v
                                           +---------------------------+
                                           |  Jira Cloud (Atlassian)   |
                                           |  - Support tickets        |
                                           |  - Team routing           |
                                           +---------------------------+
```

---

## Part 1: Widget on uob.edu.bh

### Why a Widget?

The UOB website is a **WordPress site** (Avada theme). We can't (and shouldn't) rebuild it. Instead, we inject a **chat widget** — a small floating button in the corner that opens a chat window.

### How to Embed on WordPress

There are **3 options**, from simplest to most robust:

#### Option A: JavaScript Embed (Recommended)

A single `<script>` tag that the WordPress admin adds to the site. This is how Intercom, Crisp, Drift, and every major chat widget works.

**WordPress admin adds this (one time):**
```html
<!-- UOB AI Chat Widget -->
<script>
  (function() {
    var w = document.createElement('script');
    w.src = 'https://your-server.com/widget/uob-chat.js';
    w.async = true;
    document.head.appendChild(w);
  })();
</script>
```

**What `uob-chat.js` does:**
1. Creates a floating button (bottom-right corner)
2. On click, opens an iframe pointing to your React chat app
3. Handles open/close animation
4. Passes the current page URL to the chatbot (context)
5. Works on mobile and desktop

**Why this is best:**
- WordPress admin only adds one line — no plugin needed
- Widget updates automatically (you change the JS on your server)
- No interference with Avada theme
- Works on every page of uob.edu.bh

#### Option B: iframe Embed

Simpler but less flexible. Add an iframe to specific pages:

```html
<iframe 
  src="https://your-server.com/chat" 
  style="position:fixed; bottom:20px; right:20px; width:400px; height:600px; border:none; border-radius:16px; z-index:9999;"
  allow="microphone"
></iframe>
```

**Downside:** No open/close toggle, always visible, harder to make responsive.

#### Option C: WordPress Plugin

Build a custom WordPress plugin that injects the widget. This is overkill for now but useful if UOB IT wants control over which pages show the widget.

### Widget Architecture

```
Student visits uob.edu.bh
         |
         v
WordPress loads page (Avada theme)
         |
         v
<script> tag loads uob-chat.js from your server
         |
         v
uob-chat.js creates:
  1. Floating button (bottom-right)
  2. Hidden iframe (your React chat app)
         |
         v
Student clicks button -> iframe opens
         |
         v
React app inside iframe talks to FastAPI via SSE
(same as current UOB-AI, just inside an iframe)
```

### Widget File Structure (new in UOB-AI repo)

```
UOB-AI/
|-- frontend/
|   |-- src/
|   |   |-- App.jsx           # Existing chat UI (works inside iframe)
|   |   |-- widget-mode.css   # Compact styles for widget mode
|   |
|-- widget/
|   |-- uob-chat.js           # The embed script (loaded by WordPress)
|   |-- widget.css             # Floating button + iframe container styles
```

### CORS Configuration

Since the widget on `uob.edu.bh` talks to your server, you need to allow cross-origin requests:

```python
# api.py - update CORS
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://www.uob.edu.bh",
        "https://uob.edu.bh",
        "http://localhost:5173",  # dev
    ],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Widget Security

| Risk | Solution |
|------|----------|
| Someone embeds your widget on a fake site | Check `Referer` header — only allow `uob.edu.bh` |
| DDoS via widget | Rate limiting already exists (30 msg/10 min per IP) |
| XSS through widget | iframe is sandboxed — can't access parent page DOM |
| Student data leaking to WordPress | Widget runs in iframe — no data shared with WordPress |

---

## Part 2: Amazon Aurora PostgreSQL (Replacing Supabase)

### Why Aurora Instead of Supabase?

| | Supabase | Aurora PostgreSQL |
|---|---|---|
| **Control** | Supabase manages it | You manage it (or AWS does) |
| **Performance** | Good | 3-5x faster than standard PostgreSQL |
| **Scaling** | Manual | Auto-scales storage (up to 128 TB) |
| **Read replicas** | Paid | Up to 15 read replicas |
| **pgvector** | Supported | Supported (extension) |
| **Compliance** | SOC 2 | SOC 2, ISO, HIPAA — better for university |
| **Location** | Fixed regions | Choose region (me-south-1 Bahrain!) |
| **Cost** | Free tier then pay | Pay per use (no free tier) |

**Key reason:** Aurora can run in **me-south-1 (Bahrain region)** — student data stays in Bahrain. This matters for university compliance.

### Aurora Setup

#### 1. Create Aurora Cluster

```
AWS Console -> RDS -> Create Database
  - Engine: Amazon Aurora PostgreSQL-Compatible
  - Version: 15.x or 16.x (must support pgvector)
  - Template: Production
  - DB Cluster Identifier: uob-ai-db
  - Master username: uob_admin
  - Instance: db.r6g.large (start small, scale later)
  - Region: me-south-1 (Bahrain)
  - VPC: your existing VPC
  - Public access: No (internal only)
  - Security group: allow port 5432 from your app server only
```

#### 2. Enable pgvector Extension

```sql
-- Connect to Aurora and run:
CREATE EXTENSION IF NOT EXISTS vector;

-- Verify
SELECT * FROM pg_extension WHERE extname = 'vector';
```

#### 3. Create Tables (migrate from Supabase schema)

```sql
-- FAQ embeddings
CREATE TABLE faq_embeddings (
    id SERIAL PRIMARY KEY,
    faq_id TEXT NOT NULL,
    question TEXT NOT NULL,
    answer_en TEXT,
    answer_ar TEXT,
    embedding vector(3072) NOT NULL
);

-- Calendar embeddings
CREATE TABLE calendar_chunks (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    source TEXT,
    embedding vector(3072) NOT NULL
);

-- Regulation embeddings (dual language)
CREATE TABLE regulation_chunks (
    id SERIAL PRIMARY KEY,
    content_en TEXT,
    content_ar TEXT,
    article_number TEXT,
    embedding_en vector(3072),
    embedding_ar vector(3072)
);

-- Ticket tracking (NEW - for Jira integration)
CREATE TABLE ticket_sessions (
    id SERIAL PRIMARY KEY,
    session_id TEXT NOT NULL,
    ticket_id TEXT NOT NULL,
    category TEXT,
    language TEXT DEFAULT 'en',
    student_email TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- HNSW indexes for fast similarity search
CREATE INDEX idx_faq_embedding ON faq_embeddings 
    USING hnsw (embedding vector_cosine_ops);

CREATE INDEX idx_calendar_embedding ON calendar_chunks 
    USING hnsw (embedding vector_cosine_ops);

CREATE INDEX idx_regulation_en_embedding ON regulation_chunks 
    USING hnsw (embedding_en vector_cosine_ops);

CREATE INDEX idx_regulation_ar_embedding ON regulation_chunks 
    USING hnsw (embedding_ar vector_cosine_ops);
```

#### 4. Update UOB-AI to Use Aurora

Replace `db.py` (Supabase client) with a direct PostgreSQL connection:

```python
# db_aurora.py
import asyncpg

AURORA_CONFIG = {
    "host": "uob-ai-db.cluster-xxxxx.me-south-1.rds.amazonaws.com",
    "port": 5432,
    "database": "uobai",
    "user": "uob_admin",
    "password": "from-env-variable",
    "ssl": "require"
}

pool = None

async def get_pool():
    global pool
    if pool is None:
        pool = await asyncpg.create_pool(**AURORA_CONFIG)
    return pool

async def find_top_faq_matches(embedding, top_k=3):
    pool = await get_pool()
    async with pool.acquire() as conn:
        rows = await conn.fetch("""
            SELECT faq_id, question, answer_en, answer_ar,
                   1 - (embedding <=> $1::vector) as similarity
            FROM faq_embeddings
            ORDER BY embedding <=> $1::vector
            LIMIT $2
        """, str(embedding), top_k)
        return rows

async def retrieve_calendar_chunks(embedding, top_k=4, min_score=0.35):
    pool = await get_pool()
    async with pool.acquire() as conn:
        rows = await conn.fetch("""
            SELECT content, source,
                   1 - (embedding <=> $1::vector) as similarity
            FROM calendar_chunks
            WHERE 1 - (embedding <=> $1::vector) >= $3
            ORDER BY embedding <=> $1::vector
            LIMIT $2
        """, str(embedding), top_k, min_score)
        return rows
```

### Aurora vs Supabase: What Changes in the Codebase

| File | Change |
|------|--------|
| `db.py` | Replace with `db_aurora.py` (asyncpg instead of supabase-py) |
| `core.py` | Update imports to use new db module |
| `api.py` | Add pool initialization on startup |
| `.env` | Replace `SUPABASE_URL`/`SUPABASE_KEY` with `AURORA_HOST`/`AURORA_PASSWORD` |
| `migrate_to_pgvector.py` | Update to push embeddings to Aurora |
| `requirements.txt` | Add `asyncpg`, remove `supabase` |

---

## Part 3: Jira Integration via n8n (Secondary Feature)

### Reminder: This is Optional

The chatbot's **main job** is answering questions. Jira is the **tech support ticketing system** for when:
- The student has a **technical issue** (system down, can't login, portal error, app crash)
- The student reports a **bug** on any university system
- The student needs **IT help** (password reset, access request)
- The student explicitly asks for tech support

**The bot should NOT create a Jira ticket for:**
- Questions it can answer from FAQ/RAG (academic questions)
- Non-technical issues (payment, grades, registration — these go through other channels)
- Vague messages ("hi", "hello")
- Repeated questions (try harder before escalating)

### The Three n8n Workflows

#### Workflow 1: Escalation (Chatbot -> Jira)

**When:** Student has a technical issue that needs IT support

```
Student: "I can't login to the student portal, it keeps showing error 500"
Bot:     [searches FAQ - no match]
         [detects: technical issue, system error]
         "Looks like a technical issue! I'll create a support ticket for the IT team.
          Can you describe what you were trying to do?"
Student: "I was trying to check my grades on the portal"
Bot:     [sends webhook to n8n with full context]
         "Done! Ticket UOB-456 created. IT support will investigate and contact you."
```

**n8n flow:**
```
Webhook -> Classify Priority -> Create Jira Issue -> Return Ticket ID
           |                                          |
           |  system down = CRITICAL                  |  also:
           |  login/access = HIGH                     |  - notify Slack
           |  bug report = MEDIUM                     |  - notify IT team
           |  general tech = LOW                      |  - send email
```

#### Workflow 2: Status Check (Student -> Jira via bot)

```
Student: "What happened with ticket UOB-456?"
Bot:     [detects ticket status intent]
         [sends webhook to n8n with ticket ID]
         "Ticket UOB-456:
          Status: In Progress
          Team: Finance Department
          Last update: Verifying payment with the bank (2 hours ago)"
```

#### Workflow 3: Notification (Jira -> Student)

```
Support agent updates ticket in Jira
  -> Jira fires webhook to n8n
  -> n8n looks up student email from ticket_sessions table
  -> n8n sends email: "Your ticket UOB-456 has been updated"
```

### Escalation Detection Logic

```python
# How the chatbot decides to create a tech support ticket

def should_create_ticket(message, faq_score, rag_chunks, language):
    """
    Returns: (should_create: bool, reason: str, category: str)
    
    ONLY creates tickets for TECHNICAL issues.
    Academic/admin questions are answered by the bot or redirected.
    """
    
    # 1. Student explicitly asks for tech support
    if has_tech_support_keywords(message, language):
        category = classify_tech_category(message, language)
        return True, "student_requested_tech_support", category
    
    # 2. Student reports a technical problem
    if is_technical_issue(message, language):
        category = classify_tech_category(message, language)
        return True, "technical_issue_detected", category
    
    # 3. Non-technical issue the bot can't answer -> redirect, don't ticket
    if faq_score < 0.40 and not rag_chunks:
        if is_academic_or_admin(message, language):
            return False, "redirect_to_department", None  # redirect, no ticket
        # Unknown issue -> offer tech support as option
        return False, "offer_ticket_option", None
    
    return False, None, None


TECH_KEYWORDS_EN = [
    "error", "bug", "crash", "can't login", "not working", "down",
    "slow", "loading", "500", "404", "password reset", "locked out",
    "printer", "system", "portal error", "app crash"
]

TECH_KEYWORDS_AR = [
    "خطأ", "ما يشتغل", "واقف", "ما أقدر أدخل", "بطيء", "الموقع طاح",
    "طابعة", "النظام", "كلمة السر", "الصفحة ما تفتح",
    "مشكلة تقنية", "دعم فني", "ما يفتح", "error"
]
```

### Tech Support Categories & Routing

All Jira tickets go to **one Tech Support team**. Categories are just labels to help them prioritize:

| Category | Detection Keywords | Priority | Jira Label |
|----------|-------------------|----------|------------|
| System Outage | نظام واقف, server down, 500 error, site down, الموقع ما يشتغل | Critical | `outage` |
| Login/Access | ما أقدر أدخل, can't login, password, locked out, access denied | High | `access` |
| Portal Bugs | خطأ, error, bug, glitch, الصفحة ما تفتح, not loading, crash | High | `bug` |
| Email/Apps | إيميل, email, outlook, teams, الايميل ما يشتغل | Medium | `email-apps` |
| Hardware | طابعة, printer, projector, بروجكتر, lab computer | Low | `hardware` |
| General Tech | مساعدة تقنية, tech help, IT help, everything else | Low | `general-tech` |

**One team handles everything.** Labels + priority help them decide what to fix first.

### What is NOT a Jira ticket (redirect instead):

| Student says | Bot responds |
|-------------|-------------|
| "I want to pay fees" | "For payment issues, visit the Finance Office or uob.edu.bh/finance" |
| "What's my GPA?" | "I can explain GPA rules! For your specific GPA, check the student portal." |
| "I want to register for courses" | "Here's how registration works: [FAQ/RAG answer]" |
| "I have a complaint about a professor" | "For academic complaints, contact the Dean's Office." |

### Webhook Payload (Chatbot -> n8n)

```json
{
  "session_id": "uuid-abc-123",
  "message": "I can't login to the student portal, error 500",
  "language": "en",
  "conversation_history": [
    {"role": "user", "content": "I can't login to the student portal, error 500"},
    {"role": "assistant", "content": "That sounds like a technical issue. Let me create a ticket..."}
  ],
  "category": "bug",
  "priority": "high",
  "affected_system": "student_portal",
  "error_details": "HTTP 500 on login page",
  "escalation_reason": "technical_issue",
  "page_url": "https://www.uob.edu.bh/student-portal",
  "timestamp": "2026-05-14T10:30:00Z"
}
```

**Note:** `page_url` is new — the widget captures which page the student was on when they asked. This helps the support team understand context.

---

## Part 4: Complete System Architecture

### How Everything Connects

```
+-----------------------------------------------------------------------+
|                         AWS (me-south-1 Bahrain)                      |
|                                                                       |
|  +------------------+                                                 |
|  | Aurora PostgreSQL |  <-- pgvector for embeddings                   |
|  | (uob-ai-db)      |  <-- ticket_sessions for Jira tracking         |
|  +--------+---------+                                                 |
|           ^                                                           |
|           | asyncpg (port 5432, internal)                             |
|           |                                                           |
|  +--------+---------+     webhook     +------------------+            |
|  |                  +---------------->+                  |            |
|  |  UOB-AI Chatbot  |                |  n8n             |            |
|  |  (FastAPI :8005) |<----------------+  (Docker :5678)  |            |
|  |                  |   ticket info   |                  |            |
|  +--------+---------+                +--------+---------+            |
|           ^                                   |                      |
|           | Nginx reverse proxy               | Jira REST API        |
|           |                                   |                      |
|  +--------+---------+                +--------v---------+            |
|  |                  |                |                  |            |
|  |  Nginx (:443)    |                |  Jira Cloud      |            |
|  |  SSL termination |                |  (atlassian.net) |            |
|  |  + static files  |                |                  |            |
|  +--------+---------+                +------------------+            |
|           ^                                                          |
+-----------+----------------------------------------------------------+
            |
            | HTTPS
            |
+-----------+----------------------------------------------------------+
|           v                                                          |
|  +--------+---------+                                                |
|  |  uob.edu.bh      |                                                |
|  |  (WordPress)      |                                                |
|  |                  |                                                |
|  |  +-------------+ |                                                |
|  |  | Chat Widget | |  <-- iframe/JS embed                          |
|  |  | (bottom-    | |  <-- talks to FastAPI via HTTPS                |
|  |  |  right)     | |  <-- captures page URL for context             |
|  |  +-------------+ |                                                |
|  +------------------+                                                |
|         Student's Browser                                            |
+----------------------------------------------------------------------+
```

### Port Map

| Service | Port | Access | Purpose |
|---------|------|--------|---------|
| Nginx | 443 (HTTPS) | Public | SSL termination, serve widget, proxy API |
| UOB-AI (FastAPI) | 8005 | Internal only | Chatbot backend |
| n8n | 5678 | Internal only | Workflow automation |
| Aurora PostgreSQL | 5432 | Internal only (VPC) | Embeddings + ticket data |

### Nginx Configuration (Updated for Widget)

```nginx
server {
    listen 443 ssl;
    server_name your-server.com;

    ssl_certificate /etc/ssl/certs/your-cert.pem;
    ssl_certificate_key /etc/ssl/private/your-key.pem;

    # Serve the chat widget embed script
    location /widget/ {
        alias /var/www/uob-ai/widget/;
        add_header Access-Control-Allow-Origin "https://www.uob.edu.bh";
        add_header Cache-Control "public, max-age=3600";
    }

    # Serve the React chat app (inside iframe)
    location / {
        root /var/www/uob-ai/frontend/dist;
        try_files $uri /index.html;
    }

    # Proxy API calls to FastAPI
    location /api/ {
        proxy_pass http://127.0.0.1:8005;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # SSE streaming support
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
    }

    # Block direct n8n access from outside
    # n8n is only accessible from FastAPI (internal)
}
```

---

## Part 5: File Structure

### Jira-AI Repo (this repo)

```
Jira-AI/
|
|-- PLAN.md                          # This file
|-- ARCHITECTURE.md                  # Mermaid diagrams
|-- README.md                        # Setup & deployment guide
|
|-- n8n/
|   |-- docker-compose.yml           # Run n8n (+ Aurora access)
|   |-- .env.example                 # n8n, Jira, Aurora credentials
|   |-- workflows/
|   |   |-- escalation.json          # Workflow 1: chatbot -> Jira
|   |   |-- status-check.json        # Workflow 2: check ticket status
|   |   |-- jira-notify.json         # Workflow 3: Jira -> notification
|
|-- chatbot-plugin/
|   |-- escalation.py                # Detects when to escalate
|   |-- categories.py                # Category & priority mapping
|   |-- n8n_client.py                # Sends webhooks to n8n
|   |-- models.py                    # Pydantic models for tickets
|   |-- __init__.py
|
|-- db/
|   |-- migration.sql                # Aurora schema (tables + indexes)
|   |-- seed_tickets.sql             # Test data
|
|-- widget/
|   |-- uob-chat.js                  # Embed script (loaded by WordPress)
|   |-- widget.css                   # Floating button styles
|   |-- README.md                    # WordPress embed instructions
|
|-- jira-setup/
|   |-- project-config.md            # How to set up Jira project
|   |-- issue-types.md               # Custom fields & workflows
|
|-- tests/
|   |-- test_escalation.py
|   |-- test_categories.py
|   |-- test_n8n_client.py
```

### Changes to UOB-AI Repo

```
UOB-AI/
|-- db.py            -> db_aurora.py       # Replace Supabase with Aurora (asyncpg)
|-- api.py                                 # Add: escalation route, Jira callback endpoint
|-- core.py                                # Add: should_escalate(), escalation keywords
|-- .env                                   # Replace: SUPABASE_* with AURORA_*
|-- requirements.txt                       # Add: asyncpg. Remove: supabase
|-- frontend/
|   |-- src/App.jsx                        # Add: ticket UI, "Create Tech Support Ticket" button
|   |-- src/widget-mode.css                # New: compact styles for iframe mode
|-- widget/
|   |-- uob-chat.js                        # New: embed script for WordPress
|   |-- widget.css                         # New: floating button styles
```

---

## Part 6: Implementation Phases

### Phase 0: Aurora Migration (Day 1-2)

**Goal:** Move from Supabase to Aurora PostgreSQL

- [ ] Create Aurora cluster in me-south-1 (Bahrain)
- [ ] Enable pgvector extension
- [ ] Run migration.sql (create tables + HNSW indexes)
- [ ] Write `db_aurora.py` (asyncpg client)
- [ ] Migrate embeddings from JSON files to Aurora
- [ ] Update `core.py` to use new db module
- [ ] Test: FAQ, calendar, regulation searches all work on Aurora
- [ ] Remove Supabase dependencies

### Phase 1: Widget Embedding (Day 3-4)

**Goal:** Chatbot appears as a widget on uob.edu.bh

- [ ] Build `uob-chat.js` (embed script)
- [ ] Build `widget.css` (floating button + iframe container)
- [ ] Add `widget-mode.css` to React app (compact layout for iframe)
- [ ] Configure CORS for `uob.edu.bh`
- [ ] Configure Nginx to serve widget files with correct CORS headers
- [ ] Test on uob.edu.bh (or a staging copy)
- [ ] Test mobile responsiveness
- [ ] Test RTL (Arabic) layout in widget mode

### Phase 2: Jira Escalation via n8n (Day 5-7)

**Goal:** Bot can create Jira tickets when it can't help

- [ ] Deploy n8n via Docker on the server
- [ ] Set up Jira Cloud project (UOBSUP)
- [ ] Create `escalation.py` (detection logic)
- [ ] Create `categories.py` (routing rules)
- [ ] Create `n8n_client.py` (webhook sender)
- [ ] Build n8n Workflow 1 (escalation)
- [ ] Add escalation route to `api.py`
- [ ] Add ticket UI to `App.jsx`
- [ ] Create `ticket_sessions` table in Aurora
- [ ] Test end-to-end: chat -> escalation -> Jira ticket

### Phase 3: Status Check + Notifications (Day 8-9)

**Goal:** Students can check ticket status, get notified on updates

- [ ] Build n8n Workflow 2 (status check)
- [ ] Add ticket status intent detection in chatbot
- [ ] Build n8n Workflow 3 (Jira -> email notification)
- [ ] Configure Jira webhooks (on issue update)
- [ ] Test full cycle

### Phase 4: Testing & Polish (Day 10)

- [ ] End-to-end testing (all 3 workflows)
- [ ] Test Arabic & English flows
- [ ] Test error cases (n8n down, Jira down, Aurora down)
- [ ] Load testing (widget performance on WordPress)
- [ ] Security audit (CORS, rate limiting, webhook auth)

---

## Part 7: Example Conversations

### Scenario 1: Normal Q&A (no Jira needed)

```
Student: When does spring semester start?
Bot:     Spring semester 2025/2026 begins on January 25, 2026.
         [Source: FAQ match 95%]
```

**No escalation.** Bot answers from FAQ. This is the primary use case.

### Scenario 2: Student has a technical issue (Arabic) -> Jira ticket

```
Student: البوابة ما تفتح معي، يطلع خطأ 500
Bot:     يبدو إن فيه مشكلة تقنية في البوابة.
         بسوي لك تذكرة دعم تقني عشان فريق الـ IT يتابعون الموضوع.
         شنو كنت تحاول تسوي بالضبط؟

Student: كنت أبي أشوف درجاتي
Bot:     تم فتح تذكرة رقم UOB-456
         النوع: خطأ في بوابة الطالب
         الأولوية: عالية
         فريق الدعم التقني بيتابعون الموضوع.
         تقدر تسألني عن وضع التذكرة في أي وقت.
```

### Scenario 3: Student reports email issue

```
Student: الايميل الجامعي ما يفتح معي
Bot:     يبدو إن فيه مشكلة تقنية في الايميل.
         بسوي لك تذكرة دعم. شنو رسالة الخطأ اللي تطلع لك؟

Student: يقول invalid credentials بس كلمة السر صحيحة
Bot:     تم فتح تذكرة UOB-457
         النوع: مشكلة إيميل / تطبيقات
         الأولوية: متوسطة
         فريق الدعم التقني بيتواصلون معك.
```

### Scenario 4: Student checks ticket status

```
Student: شنو وضع التذكرة UOB-456؟
Bot:     تذكرة UOB-456:
         الحالة: قيد المعالجة
         الفريق: فريق التطبيقات
         آخر تحديث: "تم اكتشاف المشكلة وجاري الإصلاح" (قبل ساعتين)
```

### Scenario 5: Non-technical issue -> Bot redirects (NO Jira ticket)

```
Student: I paid my fees but it still shows unpaid
Bot:     I specialize in academic info and tech support.
         For payment issues, please contact the Finance Office:
         - Visit: Administration Building, Ground Floor
         - Call: +973 1743 8888
         - Email: finance@uob.edu.bh
         
         Is there anything else I can help with?
```

**No Jira ticket.** Payment is not a tech issue — bot redirects to the right department.

### Scenario 6: Bot answers academic question (NO Jira ticket)

```
Student: How do I change my major?
Bot:     [FAQ match: 85% - "changing major process"]
         To change your major, you need to:
         1. Complete at least one semester
         2. Have a GPA of 2.0 or higher
         3. Submit a change request through the registration office
         [Source: Regulation Article 15]
```

**No Jira ticket.** Bot answered the question. This is the primary use case.

---

## Part 8: Security & Compliance

| Area | Implementation |
|------|---------------|
| **Data residency** | Aurora in me-south-1 (Bahrain) — data stays in country |
| **Widget isolation** | iframe sandbox — widget can't access uob.edu.bh DOM |
| **CORS** | Only allow `uob.edu.bh` origin |
| **Webhook auth** | Shared secret token between FastAPI and n8n |
| **n8n access** | Internal only (port 5678 blocked from outside) |
| **Aurora access** | VPC + security group — only app server can connect |
| **Jira API token** | Stored in `.env`, rotated quarterly |
| **Rate limiting** | 30 msg/10 min per IP + max 3 tickets per session per hour |
| **PII in tickets** | Minimal — no student ID or grades in Jira tickets |
| **Ticket scope** | Only tech support issues — academic/admin issues redirected |
| **SSL** | HTTPS everywhere (Nginx SSL termination) |
| **Logging** | Metadata only — raw messages never logged |

---

## Part 9: Cost Estimate

| Service | Cost | Notes |
|---------|------|-------|
| **Aurora PostgreSQL** | ~$60-100/mo | db.r6g.large in me-south-1 |
| **n8n** | Free | Self-hosted (Docker) |
| **Jira Cloud** | Free tier or $7.75/user/mo | Free for up to 10 users |
| **OpenAI API** | ~$20-50/mo | Depends on traffic |
| **EC2 (app server)** | ~$30-80/mo | t3.medium or larger |
| **SSL Certificate** | Free | Let's Encrypt |
| **Total** | ~$110-230/mo | |

---

## Summary: What Goes Where

| Component | Responsibility | Location |
|-----------|---------------|----------|
| **uob.edu.bh** | Just hosts the widget `<script>` tag | WordPress (unchanged) |
| **Widget JS** | Floating button + iframe | Served from your Nginx |
| **React Frontend** | Chat UI (inside iframe) | Served from your Nginx |
| **FastAPI Backend** | Q&A pipeline + escalation logic | Your server :8005 |
| **Aurora PostgreSQL** | Embeddings + ticket sessions | AWS me-south-1 |
| **n8n** | Workflow automation (Jira bridge) | Docker on your server :5678 |
| **Jira Cloud** | Ticket management | Atlassian cloud |
