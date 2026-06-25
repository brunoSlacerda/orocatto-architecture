# Architecture

## Context

In Brazil, a large share of restaurant orders and questions still happen over WhatsApp —
handled by a person typing replies one message at a time. It works until volume grows:
customers wait, staff are tied to the phone, and nothing is recorded.

**Orocatto** puts an AI agent on the restaurant's WhatsApp line. Customers write the way
they always do; the agent answers using a persona and business information configured per
restaurant, and a human can step in at any moment. It is **multi-tenant** — one deployment
serves many restaurants, each with its own WhatsApp number and personality.

It currently runs in production for **Oro's Restaurante** (Igrejinha/RS, Brazil).

## Design principles

- **Multi-tenant from the ground up.** A restaurant is identified by its WhatsApp
  `phone_number_id`; everything routes from there. One codebase, many tenants.
- **Serverless and event-driven.** No always-on process; the workload is bursty
  (lunch/dinner peaks), so idle cost should be near zero.
- **Postgres coordinates.** The debounce window, the claim queue, and the message history
  all live in the database — no external queue or worker.
- **The AI owns the routine path; humans own the exceptions.** Every conversation can be
  paused, and the bot escalates itself when unsure.

See [diagrams.md](diagrams.md) for the system, sequence, and hand-off views.

## The parts worth reading

### 1. Multi-tenant routing

Each restaurant row holds its WhatsApp identifiers (`phone_number_id`, `waba_id`), a
`system_prompt`, and a `business_info` JSON blob. When a webhook arrives, the restaurant
is resolved by `phone_number_id`, and the agent is driven by *that* tenant's prompt —
so two restaurants on the same deployment have completely different personalities and
knowledge. Row-Level Security is enabled on every table.

### 2. Debounce via `buffer_until` + `pg_cron`

On WhatsApp people write in bursts:

```
oi
queria fazer um pedido
uma pizza
calabresa, grande
```

Four webhook events, seconds apart. Answering each one means four LLM calls and a reply
that talks over itself.

Instead of processing on arrival, the webhook **stores each message and opens (or renews)
a debounce window** — a `buffer_until` timestamp on the conversation, set to roughly a
minute ahead. A `pg_cron` job runs `process-buffer` on a short interval; when a
conversation's window has elapsed, the recent message history is sent to the agent as a
**single coherent turn**. One reply out, a fraction of the cost.

There is no separate buffer table — the window is just a column, and the messages
accumulate as normal rows. This pattern is published as a standalone module (see the
README).

### 3. Concurrency-safe claiming

Because `pg_cron` can fire while a previous run is still working, the flush must never
process the same conversation twice. The `claim_buffered_conversations` function does an
atomic claim:

```sql
UPDATE conversations SET buffer_until = NULL
WHERE id IN (
  SELECT id FROM conversations c JOIN restaurants r ON r.id = c.restaurant_id
  WHERE c.buffer_until <= now() AND c.bot_enabled AND r.bot_global_enabled
  FOR UPDATE OF c SKIP LOCKED      -- the important line
)
RETURNING ...;
```

`FOR UPDATE … SKIP LOCKED` is the classic safe-queue trick: each runner grabs only the
rows nobody else holds, claims them in the same statement, and moves on. No double
replies, no locks piling up.

### 4. The agent: Gemini with a fallback chain

The agent isn't a single API call — it tries a **chain of Gemini models**
(`gemini-2.5-flash-lite` → `2.0-flash-lite` → `2.5-flash` → `flash-latest`), falling
through on availability/rate errors and stopping on real client errors (400/401/403).
The tenant's `system_prompt` is templated with its `business_info`, and the last ~10
messages are replayed as context. Low temperature, short max tokens — it's a focused
assistant, not a chatbot.

### 5. Human hand-off and self-escalation

Three layers of control:
- **`bot_global_enabled`** on the restaurant — a kill switch (it ships *off*, so a tenant
  goes live deliberately).
- **`bot_enabled`** on the conversation — turns the bot off for one customer.
- **Self-escalation** — if the model returns the token `[ESCALAR]`, errors out, or comes
  back empty, the conversation is paused (`bot_enabled = false`, with `paused_at` and a
  `paused_reason`) and a human takes it.

Humans answer through **Meta Coexistence** — the same number works in the WhatsApp app and
the API at once — so staff reply naturally and the bot steps back.

### 6. Delivery resilience: the Brazilian 9th digit

Brazilian mobile numbers are a known pain: the same line may or may not carry the extra
`9` after the area code, and WhatsApp isn't always forgiving. Outbound sends try
**phone-number variants** (with and without the 9) and retry on specific Meta error codes
(`131030`, `131026`) before giving up. This is the difference between "works in the demo"
and "works for real customers."

### 7. Small touches that matter

- The webhook returns `200` **immediately** and processes in the background
  (`EdgeRuntime.waitUntil`), so Meta never times out or retries.
- While the customer waits out the debounce window, the bot sends a **typing indicator**
  and marks the message **read** (blue ticks) — it feels alive.
- **Demo conversations** (`is_demo`) never escalate; they fall back to a friendly canned
  reply, so a live demo can't break.
- Outbound **delivery statuses** (sent / delivered / read / failed) flow back through the
  same webhook and update each message.
