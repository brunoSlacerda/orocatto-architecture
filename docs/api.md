# API

Orocatto has no public REST API for third parties. Its surface is a set of **Supabase Edge
Functions** (Deno) and the **WhatsApp Cloud API** contract. All functions use Graph API
`v21.0` and authenticate to Postgres with the service role.

## Edge Functions

| Function | Auth | Role |
|---|---|---|
| `whatsapp-webhook` | public (Meta verify token) | Inbound: receive messages and delivery statuses. |
| `process-buffer` | `x-cron-secret` header | Flush: turn buffered conversations into AI replies. |
| `send-message` | service | Outbound send (used by the operator flow). |
| `meta-coexistence-callback` | public | Meta Coexistence events (human-on-the-number). |
| `criar-membro` | JWT | Provision a team member. |

---

## 1. `whatsapp-webhook` — inbound

**Verification (GET).** Meta calls once to verify the endpoint; the function compares
`hub.verify_token` against `WHATSAPP_VERIFY_TOKEN` and echoes `hub.challenge`.

```
GET /functions/v1/whatsapp-webhook
  ?hub.mode=subscribe&hub.verify_token=<token>&hub.challenge=<challenge>
```

**Messages (POST).** Returns `200 OK` **immediately** and processes asynchronously
(`EdgeRuntime.waitUntil`) so Meta never times out. Per message it:

1. Resolves the tenant: `restaurants.phone_number_id == value.metadata.phone_number_id`.
2. Handles **delivery statuses** (`sent | delivered | read | failed`) by updating the
   matching `messages.status` (looked up by `wamid`).
3. For text messages: **dedupes** by `wamid`, **upserts** the conversation
   (`onConflict: restaurant_id,customer_phone`), and inserts a `customer` message.
4. If `restaurant.bot_global_enabled` and `conversation.bot_enabled` are both true, sets
   `buffer_until = now() + 60s` and stores `last_customer_wamid`.
5. Sends a **typing indicator** and marks the message **read**.

Inbound payload (standard Cloud API shape):

```jsonc
{
  "entry": [{
    "changes": [{
      "field": "messages",
      "value": {
        "metadata": { "phone_number_id": "<tenant key>" },
        "contacts": [{ "profile": { "name": "..." }, "wa_id": "5551..." }],
        "messages": [{ "from": "5551...", "id": "wamid...", "type": "text",
                       "text": { "body": "uma pizza calabresa grande" } }]
      }
    }]
  }]
}
```

---

## 2. `process-buffer` — flush (cron-triggered)

Invoked by `pg_cron`; rejects any request without the correct `x-cron-secret` header.

1. `claim_buffered_conversations()` — atomically claims conversations whose
   `buffer_until` has elapsed (using `FOR UPDATE … SKIP LOCKED`), clearing the window in
   the same statement so no run double-processes.
2. For each claimed conversation: reload kill switches, load the last ~10 messages, and
   **only proceed if the latest non-system message is from the customer** (guards against
   a human having already replied during the window).
3. Call **Gemini** with the tenant's `system_prompt` and history (model fallback chain).
4. Handle the result: a normal reply is sent and stored as a `bot` message; `[ESCALAR]`,
   an error, or an empty response **escalates** (pauses the conversation) — except in
   `is_demo` conversations, which fall back to a canned reply.
5. Send via the Cloud API, then `clear_stale_buffers()` tidies windows for conversations
   that were disabled mid-flight.

---

## 3. Outbound — sending replies

```
POST https://graph.facebook.com/v21.0/<phone_number_id>/messages
Authorization: Bearer <WHATSAPP_TOKEN>

{ "messaging_product": "whatsapp", "to": "5551...", "type": "text",
  "text": { "body": "<reply>" } }
```

Sends try **Brazilian phone-number variants** (with/without the 9th digit) and retry on
Meta error codes `131030` / `131026` before failing.

---

## Configuration (environment)

`WHATSAPP_VERIFY_TOKEN`, `WHATSAPP_TOKEN`, `GEMINI_API_KEY`, `CRON_SECRET`,
`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` — all read from the environment; no secrets
are committed.
