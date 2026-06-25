# Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/), which GitHub renders natively.

---

## System architecture

```mermaid
flowchart TB
    Customer["Customer"]

    subgraph Meta
        Cloud["WhatsApp Cloud API"]
    end

    subgraph Supabase
        WH["whatsapp-webhook<br/>(Edge Function)"]
        PB["process-buffer<br/>(Edge Function)"]
        Cron["pg_cron"]
        Claim["claim_buffered_conversations()<br/>FOR UPDATE SKIP LOCKED"]
        DB[("conversations / messages")]
    end

    subgraph Google
        Gemini["Gemini (model fallback)"]
    end

    Customer -->|messages| Cloud
    Cloud -->|webhook| WH
    WH -->|store msg + set buffer_until| DB
    WH -->|typing + read receipt| Cloud
    Cron -->|interval| PB
    PB --> Claim
    Claim -->|due, unlocked rows| DB
    PB -->|last ~10 messages| Gemini
    Gemini -->|reply or [ESCALAR]| PB
    PB -->|send reply| Cloud
    Cloud -->|deliver| Customer
```

---

## Sequence: a batched reply

The debounce window is the key beat — the agent acts only once the customer pauses.

```mermaid
sequenceDiagram
    participant C as Customer
    participant W as Cloud API
    participant WH as whatsapp-webhook
    participant D as Database
    participant K as pg_cron
    participant PB as process-buffer
    participant G as Gemini

    C->>W: "oi"
    W->>WH: webhook
    WH->>D: insert message, set buffer_until = now()+60s
    WH->>W: typing + mark read
    C->>W: "uma pizza calabresa grande"
    W->>WH: webhook
    WH->>D: insert message, renew buffer_until
    Note over K,D: window elapses
    K->>PB: invoke (x-cron-secret)
    PB->>D: claim due conversations (SKIP LOCKED)
    PB->>G: system_prompt + last messages
    G-->>PB: reply
    PB->>W: send reply
    W->>C: one coherent answer
    PB->>D: insert bot message
```

---

## Hand-off state

How a conversation moves between bot and human.

```mermaid
stateDiagram-v2
    [*] --> Bot: bot_global_enabled && bot_enabled
    Bot --> Human: model returns [ESCALAR]
    Bot --> Human: LLM error / empty
    Bot --> Human: staff reply (Coexistence)
    Human --> Bot: re-enable conversation
    Bot --> Off: global kill switch off
    Off --> Bot: launch / re-enable
```

---

## Component responsibilities

```mermaid
flowchart LR
    subgraph Ingest
        WH["whatsapp-webhook<br/>• resolve tenant by phone_number_id<br/>• dedupe by wamid<br/>• store message<br/>• open buffer window<br/>• log delivery statuses"]
    end
    subgraph Flush
        PB["process-buffer<br/>• claim due conversations<br/>• call Gemini (fallback)<br/>• send reply<br/>• escalate on failure"]
    end
    subgraph Agent
        G["Gemini<br/>• per-tenant system prompt<br/>• replay recent history<br/>• reply or [ESCALAR]"]
    end

    WH --> PB --> G
```
