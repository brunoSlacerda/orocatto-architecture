# Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/), which GitHub renders natively.

---

## System architecture

```mermaid
flowchart LR
    Customer(["Customer"])
    Cloud["WhatsApp Cloud API"]
    WH["whatsapp-webhook"]
    DB[("conversations / messages")]
    Cron(["pg_cron"])
    PB["process-buffer"]
    Gemini["Gemini agent"]

    Customer -->|message| Cloud
    Cloud -->|webhook| WH
    WH -->|"store + open buffer window"| DB
    Cron -->|"every few seconds"| PB
    PB -->|"claim due rows, SKIP LOCKED"| DB
    PB -->|"batched history"| Gemini
    Gemini -->|"reply or ESCALAR"| PB
    PB -->|send| Cloud
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
    [*] --> Bot
    Bot --> Human: ESCALAR, error, or staff reply
    Human --> Bot: re-enabled
    Bot --> Off: kill switch off
    Off --> Bot: launch
```

---

## Component responsibilities

```mermaid
flowchart LR
    WH["whatsapp-webhook<br/>- resolve tenant by phone_number_id<br/>- dedupe by wamid<br/>- store message<br/>- open buffer window<br/>- log delivery statuses"]
    PB["process-buffer<br/>- claim due conversations<br/>- call Gemini with fallback<br/>- send reply<br/>- escalate on failure"]
    G["Gemini<br/>- per-tenant system prompt<br/>- replay recent history<br/>- reply or ESCALAR"]

    WH --> PB --> G
```