# Database

The model is deliberately small — three tables: **restaurants** (the tenants),
**conversations** (one WhatsApp thread per customer), and **messages** (the full history).
The debounce that powers the batching is not a table at all; it's a single timestamp
column on `conversations`. Row-Level Security is enabled on every table.

## Entity-relationship diagram

```mermaid
erDiagram
    RESTAURANTS ||--o{ CONVERSATIONS : has
    CONVERSATIONS ||--o{ MESSAGES : contains

    RESTAURANTS {
        uuid id PK
        text name
        text phone_number_id UK "WhatsApp number id (tenant key)"
        text waba_id
        text display_phone
        text system_prompt "per-tenant AI persona"
        jsonb business_info
        jsonb human_contact
        boolean bot_default_on
        boolean bot_global_enabled "kill switch (ships off)"
        timestamptz created_at
        timestamptz updated_at
    }
    CONVERSATIONS {
        uuid id PK
        uuid restaurant_id FK
        text customer_phone
        text customer_name
        boolean bot_enabled "per-conversation switch"
        timestamptz buffer_until "debounce window"
        text last_customer_wamid
        timestamptz paused_at
        text paused_reason
        timestamptz last_message_at
        timestamptz last_human_reply_at
        boolean is_demo
        timestamptz created_at
    }
    MESSAGES {
        uuid id PK
        uuid conversation_id FK
        text role "customer | bot | human | system"
        text content
        text wamid UK "WhatsApp message id (idempotency)"
        text status "sent | delivered | read | failed"
        jsonb meta
        timestamptz created_at
    }
```

> Unique constraint: `conversations (restaurant_id, customer_phone)` — one thread per
> customer per restaurant, which is what the webhook upserts onto.

## Tables

| Table | Purpose |
|---|---|
| `restaurants` | The tenant. Holds WhatsApp identifiers, the AI `system_prompt`, business info, and the kill switches. |
| `conversations` | One thread per customer; tracks bot/human state, the debounce window, and pause reason. |
| `messages` | Every message, in and out; `role` distinguishes customer / bot / human / system, `status` tracks delivery. |

## Notes

- **The debounce is a column, not a table.** `buffer_until` is set ~60s ahead on each
  inbound message and cleared atomically when `pg_cron` claims the conversation. Fragments
  simply accumulate as `messages` rows and are replayed to the agent together.
- **`wamid` is unique** on `messages` — it's the WhatsApp message id, used both for
  idempotent ingestion (ignore duplicates Meta re-delivers) and to attach delivery-status
  updates to the right row.
- **`role = 'system'`** rows are used to record agent/send errors inline in the thread,
  which makes debugging a conversation a matter of reading it top to bottom.
- **Hand-off state lives here**: `bot_enabled`, `paused_at`, `paused_reason`,
  `last_human_reply_at` — see the hand-off diagram in [diagrams.md](diagrams.md).
