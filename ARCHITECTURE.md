# Architecture Diagrams

## 1. Full System Overview

```mermaid
graph TB
    subgraph StudentBrowser["Student's Browser"]
        UOB[uob.edu.bh\nWordPress + Avada]
        Widget[Chat Widget\niframe]
    end
    
    subgraph AWS["AWS (me-south-1 Bahrain)"]
        Nginx[Nginx :443\nSSL + Reverse Proxy]
        
        subgraph AppServer["Application Server"]
            FastAPI[UOB-AI\nFastAPI :8005]
            n8n[n8n\nDocker :5678]
        end
        
        Aurora[(Aurora PostgreSQL\npgvector)]
    end
    
    Jira[Jira Cloud\natlassian.net]
    
    UOB -->|loads script| Nginx
    Widget -->|HTTPS /api/chat| Nginx
    Nginx -->|proxy| FastAPI
    Nginx -->|serves| Widget
    FastAPI -->|asyncpg| Aurora
    FastAPI -->|webhook| n8n
    n8n -->|callback| FastAPI
    n8n -->|REST API| Jira
    Jira -->|webhook| n8n

    style UOB fill:#7B1FA2,color:#fff
    style Widget fill:#4CAF50,color:#fff
    style FastAPI fill:#2196F3,color:#fff
    style n8n fill:#FF6D00,color:#fff
    style Aurora fill:#FF9800,color:#fff
    style Jira fill:#0052CC,color:#fff
    style Nginx fill:#009688,color:#fff
```

## 2. Widget Embedding on uob.edu.bh

```mermaid
sequenceDiagram
    participant S as Student
    participant W as WordPress (uob.edu.bh)
    participant JS as uob-chat.js
    participant IF as Chat iframe
    participant API as FastAPI Backend

    S->>W: Visits uob.edu.bh/any-page
    W->>W: Loads page (Avada theme)
    W->>JS: <script src="your-server/widget/uob-chat.js">
    JS->>JS: Creates floating button (bottom-right)
    
    Note over S,JS: Student sees a chat bubble icon
    
    S->>JS: Clicks chat button
    JS->>IF: Opens hidden iframe (your-server/chat)
    IF->>API: GET / (loads React chat app)
    API-->>IF: React SPA
    
    Note over S,IF: Chat window appears
    
    S->>IF: Types: "When is exam week?"
    IF->>API: POST /api/chat/stream
    API-->>IF: SSE stream (answer tokens)
    IF-->>S: Displays answer in real-time
    
    S->>JS: Clicks X to close
    JS->>JS: Hides iframe (keeps session alive)
```

## 3. Widget UI Layout

```mermaid
graph TB
    subgraph UOBWebsite["uob.edu.bh (any page)"]
        subgraph PageContent["Page Content"]
            Header[Header / Nav]
            Content[Page Content\nAdmissions, Programs, etc.]
            Footer[Footer]
        end
        
        subgraph ChatWidget["Chat Widget (fixed position, bottom-right)"]
            subgraph Closed["Closed State"]
                Bubble[Chat Bubble Button\nwith UOB logo]
            end
            
            subgraph Open["Open State (400x600px)"]
                WidgetHeader[UOB AI Assistant Header]
                Messages[Chat Messages Area\nFAQ + RAG responses]
                TicketBanner[Ticket Banner\nUOB-456 created]
                Input[Message Input\n+ Send Button]
                HumanBtn[Talk to Human Button]
            end
        end
    end

    style Bubble fill:#4CAF50,color:#fff
    style WidgetHeader fill:#2196F3,color:#fff
    style Messages fill:#E3F2FD,color:#000
    style TicketBanner fill:#FF9800,color:#fff
    style Input fill:#fff,color:#000
    style HumanBtn fill:#f44336,color:#fff
```

## 4. Primary Flow: Q&A (No Jira)

```mermaid
flowchart TD
    A[Student asks question\nvia widget on uob.edu.bh] --> B[FastAPI receives message]
    B --> C[Sanitize + Detect Language]
    C --> D[Search FAQ Embeddings\nin Aurora pgvector]
    
    D --> E{FAQ Score >= 0.70?}
    E -->|Yes| F[Return FAQ Answer\nvia SSE stream]
    
    E -->|No| G[Search RAG\nCalendar + Regulation chunks]
    G --> H{Relevant chunks found?}
    H -->|Yes| I[Stream GPT-4.1 Response\nwith context from Aurora]
    
    H -->|No| J{Escalation needed?}
    J -->|No - vague/greeting| K[Ask student to clarify]
    J -->|Yes - out of domain| L[Offer Jira ticket]
    
    F --> M[Student gets answer]
    I --> M
    K --> M

    style A fill:#4CAF50,color:#fff
    style F fill:#2196F3,color:#fff
    style I fill:#2196F3,color:#fff
    style L fill:#FF6D00,color:#fff
    style M fill:#4CAF50,color:#fff
```

## 5. Secondary Flow: Jira Escalation

```mermaid
flowchart TD
    A[Bot can't answer\nOR student asks for human] --> B{Student confirms\nthey want a ticket?}
    
    B -->|No| C[Continue chatting]
    B -->|Yes| D[Detect Category]
    
    D --> E{Category}
    E -->|Payment/Fees| F[Priority: HIGH\nTeam: Finance]
    E -->|Registration| G[Priority: HIGH\nTeam: Registrar]
    E -->|Grades| H[Priority: MEDIUM\nTeam: Academic]
    E -->|IT/System| I[Priority: MEDIUM\nTeam: IT]
    E -->|Complaint| J[Priority: HIGH\nTeam: QA]
    E -->|General| K[Priority: LOW\nTeam: Student Services]
    
    F & G & H & I & J & K --> L[Send webhook to n8n]
    
    L --> M[n8n creates Jira issue]
    M --> N[Save to ticket_sessions\nin Aurora]
    N --> O[Return ticket ID to student\nUOB-456]

    style A fill:#f44336,color:#fff
    style L fill:#FF6D00,color:#fff
    style M fill:#0052CC,color:#fff
    style N fill:#FF9800,color:#fff
    style O fill:#4CAF50,color:#fff
```

## 6. n8n Workflow 1: Escalation

```mermaid
flowchart LR
    A[Webhook\nPOST /webhook/escalate] --> B[Validate\nshared secret]
    B --> C[Set Priority\nbased on category]
    C --> D[Format Jira\nDescription]
    D --> E[Create Issue\nJira REST API]
    E --> F{Success?}
    F -->|201| G[Return ticket_id\nto chatbot]
    F -->|Error| H[Retry 3x\nwith backoff]
    H -->|Still fails| I[Alert admin\nSlack/email]
    
    G --> J[Optional:\nnotify Slack channel]
    G --> K[Optional:\nemail support team]

    style A fill:#FF6D00,color:#fff
    style E fill:#0052CC,color:#fff
    style G fill:#4CAF50,color:#fff
    style I fill:#f44336,color:#fff
```

## 7. n8n Workflow 2: Status Check

```mermaid
flowchart LR
    A[Webhook\nPOST /webhook/status] --> B[Extract ticket_id\nfrom payload]
    B --> C[Lookup session\nin Aurora]
    C --> D{Session owns\nthis ticket?}
    D -->|No| E[Return: not found]
    D -->|Yes| F[GET Jira issue\n/rest/api/3/issue/ID]
    F --> G[Format response\nstatus + assignee + comments]
    G --> H[Return to chatbot]

    style A fill:#FF6D00,color:#fff
    style F fill:#0052CC,color:#fff
    style H fill:#4CAF50,color:#fff
```

## 8. n8n Workflow 3: Jira -> Notification

```mermaid
flowchart LR
    A[Jira Webhook\nissue:updated] --> B[Extract changes\nstatus/comment]
    B --> C[Query Aurora\nticket_sessions table]
    C --> D[Find student email\nfrom session]
    D --> E{Has email?}
    E -->|Yes| F[Send email\nstatus update]
    E -->|No| G[Log: no contact\ninfo available]
    
    F --> H[Optional:\nSlack notification]

    style A fill:#0052CC,color:#fff
    style C fill:#FF9800,color:#fff
    style F fill:#FF6D00,color:#fff
```

## 9. Complete Sequence: Escalation End-to-End

```mermaid
sequenceDiagram
    participant S as Student (Widget)
    participant W as uob.edu.bh
    participant N as Nginx
    participant F as FastAPI
    participant A as Aurora DB
    participant n as n8n
    participant J as Jira Cloud
    participant T as Support Agent

    Note over S,T: Student asks question bot can't answer

    S->>W: Opens chat widget
    W->>N: Loads uob-chat.js
    N-->>S: Widget appears (floating button)
    S->>S: Clicks button, types message
    
    S->>N: POST /api/chat/stream
    N->>F: Proxy to :8005
    F->>A: Search FAQ embeddings (pgvector)
    A-->>F: No match (score < 0.40)
    F->>A: Search RAG chunks
    A-->>F: No relevant chunks
    F->>F: Detect: SUPPORT_NEEDED
    F-->>S: SSE: "تبيني افتح لك تذكرة؟"

    S->>N: POST /api/chat/stream ("ايه")
    N->>F: Proxy
    F->>F: Detect: ESCALATION_CONFIRMED
    F->>F: Classify: payment category
    
    F->>n: POST /webhook/escalate
    n->>J: POST /rest/api/3/issue
    J-->>n: 201 {key: "UOB-456"}
    n-->>F: {ticket_id: "UOB-456"}
    
    F->>A: INSERT ticket_sessions
    F-->>S: SSE: "تم فتح تذكرة UOB-456"

    Note over S,T: Support agent works on ticket

    T->>J: Update status: In Progress
    T->>J: Add comment
    J->>n: Webhook: issue updated
    n->>A: Query ticket_sessions
    A-->>n: Student email
    n->>S: Email notification

    Note over S,T: Student checks status

    S->>N: POST /api/chat/stream ("وضع UOB-456?")
    N->>F: Proxy
    F->>n: POST /webhook/status
    n->>J: GET /rest/api/3/issue/UOB-456
    J-->>n: Issue details
    n-->>F: Status + comments
    F-->>S: SSE: "قيد المعالجة - جاري التحقق"
```

## 10. Aurora PostgreSQL Schema

```mermaid
erDiagram
    FAQ_EMBEDDINGS {
        int id PK
        text faq_id
        text question
        text answer_en
        text answer_ar
        vector3072 embedding
    }
    
    CALENDAR_CHUNKS {
        int id PK
        text content
        text source
        vector3072 embedding
    }
    
    REGULATION_CHUNKS {
        int id PK
        text content_en
        text content_ar
        text article_number
        vector3072 embedding_en
        vector3072 embedding_ar
    }
    
    TICKET_SESSIONS {
        int id PK
        text session_id
        text ticket_id
        text category
        text language
        text student_email
        timestamp created_at
        timestamp updated_at
    }

    FAQ_EMBEDDINGS ||--o{ CALENDAR_CHUNKS : "same Aurora DB"
    CALENDAR_CHUNKS ||--o{ REGULATION_CHUNKS : "same Aurora DB"
    REGULATION_CHUNKS ||--o{ TICKET_SESSIONS : "same Aurora DB"
```

## 11. Deployment Architecture (AWS)

```mermaid
graph TB
    subgraph Internet
        Student[Student Browser\nuob.edu.bh widget]
        JiraCloud[Jira Cloud\natlassian.net]
    end

    subgraph AWS["AWS Region: me-south-1 (Bahrain)"]
        subgraph VPC["VPC"]
            subgraph PublicSubnet["Public Subnet"]
                EC2[EC2 Instance\nt3.medium]
            end
            
            subgraph PrivateSubnet["Private Subnet"]
                AuroraWriter[(Aurora Writer\ndb.r6g.large)]
                AuroraReader[(Aurora Reader\nReplica)]
            end
        end
        
        subgraph EC2Detail["EC2 Instance"]
            Nginx2[Nginx :443]
            FastAPI2[FastAPI :8005]
            Docker[Docker]
            n8n2[n8n :5678]
        end
        
        SG1[Security Group\nEC2: 443 from 0.0.0.0]
        SG2[Security Group\nAurora: 5432 from EC2 only]
    end

    Student -->|HTTPS :443| Nginx2
    Nginx2 --> FastAPI2
    FastAPI2 --> n8n2
    FastAPI2 -->|asyncpg :5432| AuroraWriter
    FastAPI2 -->|reads| AuroraReader
    n8n2 -->|HTTPS| JiraCloud
    JiraCloud -->|webhook| n8n2

    style Student fill:#4CAF50,color:#fff
    style AuroraWriter fill:#FF9800,color:#fff
    style AuroraReader fill:#FFB74D,color:#000
    style FastAPI2 fill:#2196F3,color:#fff
    style n8n2 fill:#FF6D00,color:#fff
    style JiraCloud fill:#0052CC,color:#fff
    style Nginx2 fill:#009688,color:#fff
```

## 12. Request Flow Summary

```mermaid
graph LR
    subgraph "90% of requests (Q&A)"
        A1[Student] -->|question| B1[FastAPI]
        B1 -->|vector search| C1[Aurora pgvector]
        C1 -->|results| B1
        B1 -->|answer| A1
    end
    
    subgraph "10% of requests (Escalation)"
        A2[Student] -->|can't answer| B2[FastAPI]
        B2 -->|webhook| C2[n8n]
        C2 -->|create ticket| D2[Jira]
        D2 -->|ticket ID| C2
        C2 -->|callback| B2
        B2 -->|ticket info| A2
    end

    style A1 fill:#4CAF50,color:#fff
    style B1 fill:#2196F3,color:#fff
    style C1 fill:#FF9800,color:#fff
    style A2 fill:#4CAF50,color:#fff
    style B2 fill:#2196F3,color:#fff
    style C2 fill:#FF6D00,color:#fff
    style D2 fill:#0052CC,color:#fff
```
