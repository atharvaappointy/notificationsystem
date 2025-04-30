# Notli Microservices Database ER Diagram

This entity-relationship diagram represents the database structure across Notli's microservices architecture. It shows the relationships between different entities and tables within the system.

```mermaid
erDiagram
    %% Authentication Service (MS1)
    USERS {
        uuid id PK
        string email UK
        string password
        timestamp created_at
        timestamp updated_at
    }

    API_KEYS {
        uuid id PK
        uuid user_id FK
        string name
        string prefix
        string key_hash UK
        timestamp last_used
        timestamp created_at
        timestamp updated_at
    }

    %% Message Queue Service (MS2)
    IDEMPOTENCY_KEYS {
        string id PK
        timestamp created_at
    }

    %% Priority Service (MS3)
    PRIORITY_RULES {
        uuid id PK
        string name
        int priority_level
        string conditions_json
        bool is_active
        timestamp created_at
        timestamp updated_at
    }

    %% Delivery Service (MS4)
    EMAIL_DELIVERIES {
        string message_id PK
        string idempotency_key UK
        string recipient
        string sender
        string subject
        string template_id FK
        string provider
        string status
        string status_details
        int retry_count
        timestamp next_retry_at
        timestamp created_at
        timestamp updated_at
    }

    EMAIL_STATUS_EVENTS {
        int id PK
        string message_id FK
        string status
        string provider
        string details
        timestamp occurred_at
        timestamp recorded_at
    }

    EMAIL_TEMPLATES {
        string template_id PK
        string name
        string description
        string subject_template
        string plain_template
        string html_template
        json variables
        timestamp created_at
        timestamp updated_at
    }

    %% Relationships
    USERS ||--o{ API_KEYS : "has"
    API_KEYS }o--|| USERS : "belongs to"
    
    EMAIL_DELIVERIES ||--o{ EMAIL_STATUS_EVENTS : "has"
    EMAIL_STATUS_EVENTS }o--|| EMAIL_DELIVERIES : "belongs to"
    
    EMAIL_DELIVERIES }o--o| EMAIL_TEMPLATES : "uses"
    EMAIL_TEMPLATES |o--o{ EMAIL_DELIVERIES : "used by"
    
    EMAIL_DELIVERIES }o--o| IDEMPOTENCY_KEYS : "references"
    
    %% Business flow relationships (logical, not FK constraints)
    USERS ||--o{ IDEMPOTENCY_KEYS : "creates messages with"
    API_KEYS ||--o{ EMAIL_DELIVERIES : "authenticates"
    PRIORITY_RULES ||--o{ EMAIL_DELIVERIES : "prioritizes"
```

## Entity Descriptions

### Authentication Service (MS1)
- **USERS**: Stores user accounts with email and password (hashed)
- **API_KEYS**: Authentication tokens for accessing the notification APIs

### Message Queue Service (MS2)
- **IDEMPOTENCY_KEYS**: Ensures message deduplication to prevent duplicate processing

### Priority Service (MS3)
- **PRIORITY_RULES**: Defines rules for message prioritization based on conditions

### Delivery Service (MS4)
- **EMAIL_DELIVERIES**: Main table tracking all email deliveries and their status
- **EMAIL_STATUS_EVENTS**: History of status changes for each delivery (for auditing)
- **EMAIL_TEMPLATES**: Reusable email templates with variable substitution

## Key Relationships

1. Each User can have multiple API Keys
2. Each Email Delivery can have multiple Status Events
3. Email Deliveries can use Email Templates
4. Each Email Delivery references an Idempotency Key
5. API Keys are used to authenticate message submissions
6. Priority Rules determine message handling priority

This diagram provides a comprehensive view of how data flows through the Notli notification system across its microservices architecture.

