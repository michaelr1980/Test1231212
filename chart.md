# BoinksBot Architecture - User Flow

## Mermaid Diagram

```mermaid
flowchart TD
    Start([User: /boinks lock staging2 on]) --> BotLambda[Bot Lambda]
    
    BotLambda --> CheckDupe{User already has<br/>pending request?}
    CheckDupe -->|Yes| RejectDupe[❌ Error: Already queued]
    CheckDupe -->|No| WriteQueue[(Write to<br/>boinks_queue)]
    
    WriteQueue --> QueuePos[Calculate queue position]
    QueuePos --> AckUser[📱 Ack to User:<br/>'Position #2, 1 ahead']
    QueuePos --> TriggerWorker[🔔 Trigger Worker Lambda]
    
    TriggerWorker --> WorkerStart[Worker Lambda]
    
    WorkerStart --> ScanQueue[(Scan boinks_queue<br/>for processed=False)]
    ScanQueue --> HasItems{Items<br/>found?}
    
    HasItems -->|No| Done1[✅ Done]
    HasItems -->|Yes| GetOldest[Get oldest by<br/>first_queued_time]
    
    GetOldest --> MarkProcessed[(Update: processed=True)]
    MarkProcessed --> CheckAction{Action<br/>type?}
    
    CheckAction -->|wake_check| Skip[⏭️ Skip, continue<br/>to next item]
    CheckAction -->|on/off| QueryAudit[(Query boinks_audit<br/>for current state)]
    
    QueryAudit --> IsLock{Action = lock?}
    
    IsLock -->|Yes| CheckContention{Env already<br/>locked?}
    CheckContention -->|Yes| Requeue[(Re-queue with<br/>new timestamp)]
    Requeue --> NotifyContention[📢 Slack: User re-queued]
    NotifyContention --> Skip
    
    CheckContention -->|No| ProcessLock[Write lock to audit]
    ProcessLock --> NotifyLock[📢 Slack: @user<br/>env locked!]
    NotifyLock --> Skip
    
    IsLock -->|No| IsUnlock{Action = unlock?}
    
    IsUnlock -->|Yes| ValidateOwner{User owns<br/>the lock?}
    ValidateOwner -->|No| RejectUnlock[📢 Slack: Unauthorized]
    RejectUnlock --> Skip
    
    ValidateOwner -->|Yes| ProcessUnlock[Write unlock to audit]
    ProcessUnlock --> InsertWake[(Insert wake_check<br/>to trigger next)]
    InsertWake --> NotifyUnlock[📢 Slack: env unlocked]
    NotifyUnlock --> Skip
    
    Skip --> HasItems
    
    style Start fill:#e1f5ff
    style BotLambda fill:#fff4e6
    style WorkerStart fill:#fff4e6
    style WriteQueue fill:#e8f5e9
    style QueryAudit fill:#e8f5e9
    style MarkProcessed fill:#e8f5e9
    style ProcessLock fill:#e8f5e9
    style ProcessUnlock fill:#e8f5e9
    style InsertWake fill:#e8f5e9
    style NotifyLock fill:#f3e5f5
    style NotifyUnlock fill:#f3e5f5
    style NotifyContention fill:#f3e5f5
    style AckUser fill:#f3e5f5
    style RejectDupe fill:#ffebee
    style RejectUnlock fill:#ffebee
```

---

## Simplified View

```mermaid
flowchart LR
    User[👤 User in Slack] -->|/boinks lock| Bot[🤖 Bot Lambda]
    Bot -->|writes| Queue[(🗄️ Queue Table)]
    Bot -->|triggers| Worker[⚙️ Worker Lambda]
    Worker -->|reads| Queue
    Worker -->|validates & processes| Audit[(📋 Audit Table)]
    Worker -->|notifies| Slack[💬 #test_engineering]
    
    style User fill:#e1f5ff
    style Bot fill:#fff4e6
    style Worker fill:#fff4e6
    style Queue fill:#e8f5e9
    style Audit fill:#e8f5e9
    style Slack fill:#f3e5f5
```

---

## Key Differences From "Global Queue" Misconception

### ❌ What Your Colleague Thought:
```mermaid
flowchart LR
    A[User A wants staging2] --> GQ[Global Queue]
    B[User B wants staging3] --> GQ
    C[User C wants staging2] --> GQ
    GQ --> Assign[Assign to any<br/>available staging]
```

### ✅ What You Actually Built:
```mermaid
flowchart TD
    A[User A: staging2] --> Q2[(staging2 queue)]
    B[User B: staging3] --> Q3[(staging3 queue)]
    C[User C: staging2] --> Q2
    
    Q2 --> W2[Process when<br/>staging2 available]
    Q3 --> W3[Process when<br/>staging3 available]
    
    W2 --> A2[User A gets staging2]
    W3 --> B3[User B gets staging3]
    
    style Q2 fill:#ffebee
    style Q3 fill:#e8f5e9
```

---

## Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant S as Slack
    participant B as Bot Lambda
    participant Q as Queue Table
    participant W as Worker Lambda
    participant A as Audit Table
    participant C as Channel
    
    U->>S: /boinks lock staging2 on
    S->>B: Slash command event
    B->>Q: Write: {env:staging2, action:on, processed:False}
    B->>U: ✅ Queued, position #1
    B->>W: Invoke async
    
    W->>Q: Scan for processed=False
    Q-->>W: Returns oldest item
    W->>Q: Update: processed=True
    
    W->>A: Query: staging2 current state
    A-->>W: Available (no lock)
    
    W->>A: Write: {env:staging2, action:on, user:...}
    W->>C: Post: 🔒 staging2 LOCKED by @user
    C->>U: Notification
```

---

## Copy-Paste This To Your Colleague! 😄

This shows:
- ✅ Per-environment queuing (not global)
- ✅ Validation & conditional processing
- ✅ Re-queuing on contention
- ✅ Ownership checks
- ✅ Wake-check mechanism

