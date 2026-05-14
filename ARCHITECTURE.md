# Architecture Diagrams

## 1. System Overview

```mermaid
graph TB
    Student[Student Browser] -->|chat message| Nginx[Nginx :80]
    Nginx -->|/api/chat| Chatbot[UOB-AI Chatbot :8005]
    Nginx -->|static files| Frontend[React Frontend]
    
    Chatbot -->|FAQ/RAG answers| Student
    Chatbot -->|webhook: escalate| n8n[n8n Workflow Engine :5678]
    Chatbot -->|webhook: status check| n8n
    
    n8n -->|create ticket| Jira[Jira Cloud]
    n8n -->|fetch status| Jira
    n8n -->|ticket info callback| Chatbot
    
    Jira -->|issue updated webhook| n8n
    n8n -->|notification| Email[Email / SMS]
    n8n -->|notification| Slack[Slack Optional]
    
    Chatbot -->|embeddings & data| Supabase[(Supabase PostgreSQL)]

    style Student fill:#4CAF50,color:#fff
    style Chatbot fill:#2196F3,color:#fff
    style n8n fill:#FF6D00,color:#fff
    style Jira fill:#0052CC,color:#fff
    style Supabase fill:#3ECF8E,color:#fff
```

## 2. Escalation Decision Flow

```mermaid
flowchart TD
    A[Student sends message] --> B[Sanitize & Detect Language]
    B --> C{FAQ Match >= 0.70?}
    
    C -->|Yes| D[Return FAQ Answer]
    C -->|No| E{RAG has relevant context?}
    
    E -->|Yes| F[Stream LLM Response]
    E -->|No| G{Contains escalation keywords?}
    
    G -->|Yes - 'ابي موظف' / 'human agent'| H[Immediate Escalation]
    G -->|No| I[Bot says: I cant help with this.\nWould you like me to create a ticket?]
    
    I --> J{Student says yes?}
    J -->|Yes| K[Ask for category]
    J -->|No| L[End conversation]
    
    K --> M{Category selected}
    M --> H
    
    H --> N[Send webhook to n8n]
    N --> O[n8n creates Jira ticket]
    O --> P[Ticket ID returned to chatbot]
    P --> Q[Bot tells student: Ticket UOB-123 created]

    style A fill:#4CAF50,color:#fff
    style H fill:#FF6D00,color:#fff
    style O fill:#0052CC,color:#fff
    style Q fill:#4CAF50,color:#fff
```

## 3. n8n Workflow 1: Escalation

```mermaid
flowchart LR
    A[Webhook Trigger\nPOST /webhook/escalate] --> B[Set Priority\nbased on category]
    B --> C[Format Description\ninclude conversation]
    C --> D[Create Jira Issue\nPOST /rest/api/3/issue]
    D --> E{Success?}
    E -->|Yes| F[HTTP Response\nreturn ticket ID]
    E -->|No| G[Retry 3x]
    G -->|Failed| H[Log Error\nalert admin]
    
    style A fill:#FF6D00,color:#fff
    style D fill:#0052CC,color:#fff
    style F fill:#4CAF50,color:#fff
    style H fill:#f44336,color:#fff
```

## 4. n8n Workflow 2: Status Check

```mermaid
flowchart LR
    A[Webhook Trigger\nPOST /webhook/status] --> B[Extract Ticket ID\nfrom message]
    B --> C{Valid ticket ID?}
    C -->|Yes| D[GET Jira Issue\n/rest/api/3/issue/UOB-123]
    C -->|No| E[Return error:\nticket not found]
    D --> F[Format Response\nstatus + assignee + comments]
    F --> G[HTTP Response\nreturn to chatbot]
    
    style A fill:#FF6D00,color:#fff
    style D fill:#0052CC,color:#fff
    style G fill:#4CAF50,color:#fff
```

## 5. n8n Workflow 3: Jira Notification

```mermaid
flowchart LR
    A[Jira Webhook\nissue updated] --> B[Extract Changes\nstatus/comment/assignee]
    B --> C[Find Student\nsession/email lookup]
    C --> D{Notification method}
    D -->|Email| E[Send Email\nstatus update]
    D -->|In-Chat| F[Push to Chatbot\nWebSocket/SSE]
    D -->|Slack| G[Post to Channel]
    
    style A fill:#0052CC,color:#fff
    style E fill:#FF6D00,color:#fff
    style F fill:#FF6D00,color:#fff
    style G fill:#FF6D00,color:#fff
```

## 6. Complete Message Flow (Sequence Diagram)

```mermaid
sequenceDiagram
    participant S as Student
    participant F as Frontend (React)
    participant B as Backend (FastAPI)
    participant N as n8n
    participant J as Jira Cloud
    participant A as Support Agent

    Note over S,A: Scenario: Student needs human help
    
    S->>F: "دفعت الرسوم بس لين الحين يقول ما دفعت"
    F->>B: POST /api/chat/stream
    B->>B: FAQ search (no match)
    B->>B: RAG search (no relevant context)
    B->>B: Detect: SUPPORT_NEEDED
    B-->>F: SSE: "هالموضوع يحتاج متابعة. تبيني افتح تذكرة؟"
    F-->>S: Display message
    
    S->>F: "ايه افتح لي"
    F->>B: POST /api/chat/stream
    B->>B: Detect: ESCALATION_CONFIRMED
    B->>B: Classify: payment category
    B->>N: POST /webhook/escalate (session, message, history, category)
    N->>N: Set priority: HIGH (payment)
    N->>J: POST /rest/api/3/issue (create ticket)
    J-->>N: 201 Created (UOB-456)
    N-->>B: Response: {ticket_id: "UOB-456"}
    B-->>F: SSE: "تم فتح تذكرة UOB-456. فريق المالية بيتواصل معك."
    F-->>S: Display ticket confirmation

    Note over S,A: Later: Agent works on ticket
    
    A->>J: Update status: "In Progress"
    A->>J: Add comment: "Verifying payment with bank"
    J->>N: Webhook: issue updated
    N->>N: Format notification
    N->>S: Email: "Your ticket UOB-456 is being worked on"

    Note over S,A: Student checks status
    
    S->>F: "شنو وضع التذكرة UOB-456?"
    F->>B: POST /api/chat/stream
    B->>B: Detect: STATUS_CHECK
    B->>N: POST /webhook/status (ticket_id: UOB-456)
    N->>J: GET /rest/api/3/issue/UOB-456
    J-->>N: Issue details
    N-->>B: {status: "In Progress", comment: "Verifying..."}
    B-->>F: SSE: "تذكرة UOB-456: قيد المعالجة - جاري التحقق من الدفع"
    F-->>S: Display status card
```

## 7. Deployment Architecture

```mermaid
graph TB
    subgraph Internet
        Student[Student Browser]
        JiraCloud[Jira Cloud\natlassian.net]
    end
    
    subgraph Your Server
        Nginx[Nginx :80]
        
        subgraph Docker
            n8n[n8n :5678]
        end
        
        subgraph Application
            FastAPI[UOB-AI FastAPI :8005]
            React[React Static Files]
        end
        
        subgraph Database
            Supabase[(Supabase\nPostgreSQL + pgvector)]
        end
    end
    
    Student -->|HTTPS| Nginx
    Nginx -->|proxy /api| FastAPI
    Nginx -->|static| React
    FastAPI -->|internal| n8n
    FastAPI -->|queries| Supabase
    n8n -->|API calls| JiraCloud
    JiraCloud -->|webhooks| n8n

    style Student fill:#4CAF50,color:#fff
    style n8n fill:#FF6D00,color:#fff
    style FastAPI fill:#2196F3,color:#fff
    style JiraCloud fill:#0052CC,color:#fff
    style Supabase fill:#3ECF8E,color:#fff
```

## 8. Jira Project Structure

```mermaid
graph TD
    subgraph Jira Project: UOBSUP
        Board[Kanban Board]
        
        Board --> Open[Open]
        Board --> InProgress[In Progress]  
        Board --> Waiting[Waiting for Student]
        Board --> Resolved[Resolved]
        Board --> Closed[Closed]
        
        subgraph Issue Types
            SR[Service Request]
            Bug[Bug Report]
            Complaint[Complaint]
        end
        
        subgraph Teams
            Finance[Finance Team]
            Registrar[Registrar Office]
            Academic[Academic Affairs]
            IT[IT Support]
            QA[Quality Assurance]
        end
    end

    style Board fill:#0052CC,color:#fff
    style Open fill:#4CAF50,color:#fff
    style InProgress fill:#FF9800,color:#fff
    style Waiting fill:#9C27B0,color:#fff
    style Resolved fill:#2196F3,color:#fff
    style Closed fill:#607D8B,color:#fff
```
