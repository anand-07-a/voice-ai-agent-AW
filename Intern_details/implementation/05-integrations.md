# 05 — Integrations

> **Implements:** Overview §5  
> **Outcome:** You can wire telephony, speech/LLM providers, and business tools with safe contracts.

## 1. Integration map

```text
                    ┌─ CPaaS / SIP
Telephony edge ─────┤─ Contact center (optional)
                    └─ Recording storage

                    ┌─ STT
AI providers ───────┤─ LLM / realtime
                    └─ TTS

                    ┌─ CRM / ticketing
Business tools ─────┤─ Calendar / OMS / payments
                    └─ Identity / OTP

                    ┌─ Webhooks out (customer)
Platform ───────────┤─ Config API (agents, numbers)
                    └─ Secrets, flags, prompt versions
```

## 2. Telephony integrations

| Provider class | Examples | Integrate via |
| --- | --- | --- |
| CPaaS | Twilio, Telnyx, Vonage, Plivo, Exotel | REST + webhooks + media streams |
| SIP / PBX | Asterisk, FreeSWITCH, cloud SIP | SIP + RTP into media layer |
| Contact center | Amazon Connect, Genesys, Aircall | Native connectors / SIP transfer |

**Implement behind `TelephonyPort`** (see [01](01-goal-and-build-path.md)). Never sprinkle vendor SDK calls through prompt code.

### Required events to normalize

```text
CallIncoming | CallAnswered | CallEnded
AudioFrameIn | DtmfReceived
TransferStarted | TransferFailed
RecordingReady
```

## 3. AI provider integrations

| Port | Must support | Nice to have |
| --- | --- | --- |
| `SttPort` | Streaming partial + final, language hint, disconnect recovery | Endpointing, keywords/hotwords, confidence |
| `LlmPort` | Streaming text, tool calls, cancel | JSON schema tools, prompt caching |
| `TtsPort` | Streaming audio, interrupt, voice_id | Pronunciation lexicon, SSML-ish controls |

### Provider selection workflow

1. Freeze locale + telephony codec assumptions.  
2. Bench 2 STT + 2 TTS candidates on **your** sample calls (not demo WAV).  
3. Pick LLM for tool reliability first, eloquence second.  
4. Record p50/p95 turn latency and $/min before locking.

Detailed model trade-offs: [06-voice-models.md](06-voice-models.md).

## 4. Business tool design

### Tool contract template

```json
{
  "name": "book_slot",
  "description": "Books an appointment slot after user confirmation",
  "input_schema": {
    "type": "object",
    "required": ["patient_id", "slot_id", "idempotency_key"],
    "properties": {
      "patient_id": {"type": "string"},
      "slot_id": {"type": "string"},
      "idempotency_key": {"type": "string"}
    }
  },
  "timeout_ms": 4000,
  "side_effect": true,
  "allowed_states": ["CONFIRM"]
}
```

### Rules that prevent production pain

1. **Fast tools:** aim <300–500 ms p95; else cache or pre-fetch.  
2. **Idempotency keys** on every side effect (TTS retries will double-call).  
3. **Phone ≠ authenticated user** — verify before payments, PHI, PII changes.  
4. **Allowlist tools by state** — booking tool unavailable in `GREETING`.  
5. **Typed errors** to the LLM (`NOT_FOUND`, `SLOT_TAKEN`) — never raw stack traces.  
6. **Audit log** every invocation with `call_id`, args (redacted), result code, latency.

### Filler / wait policy

| Tool latency | Behavior |
| --- | --- |
| <300 ms | Silent |
| 300–1500 ms | Optional one short phrase (“Let me check that.”) |
| >1500 ms | Optimize tool, async callback pattern, or escalate |

## 5. Webhooks (outbound to your customers / tenants)

Emit stable product events (not raw CPaaS payloads):

| Event | When |
| --- | --- |
| `call.started` | Session created |
| `call.answered` | Media up |
| `call.tool_called` | Optional; high volume — usually omit |
| `call.outcome` | Terminal business result known |
| `call.ended` | Hangup + duration + cost summary |
| `call.failed` | System failure |

Include `idempotency_key` / event IDs; customers will retry.

## 6. Config & secrets

| Item | Storage |
| --- | --- |
| Provider API keys | Secret manager; per-tenant BYOK optional |
| Agent prompts / voices / tools | Versioned config DB |
| Number → agent map | Config DB with audit trail |
| Feature flags | Flag service (prompt X, TTS Y) |

**Prompt versioning:** every call stores `prompt_version`, `stt_model`, `tts_voice`, `llm_model` for evals.

## 7. CRM / ticketing patterns

| Pattern | Use |
| --- | --- |
| Lookup-by-phone | Soft identity; still verify |
| Create/update ticket on escalate | Always include summary + reason_code |
| Write disposition on hangup | Even if user abandons mid-flow |
| Attach transcript/recording links | Respect retention + ACL |

### Handoff fields (minimum)

See [03-telephony-primer.md](03-telephony-primer.md) §7 — keep the same payload into CRM notes.

## 8. Identity & sensitive actions

```text
ANI/caller_id → candidate account(s)
  → challenge (OTP, last4, DOB policy, etc.)
  → session.verified = true
  → allow sensitive tools
```

Never enable `refund`, `cancel_insurance`, `share_medical` tools until verified.

## 9. Integration test plan

- [ ] Sandbox dial-in / dial-out against staging numbers  
- [ ] STT/TTS/LLM key rotation without downtime  
- [ ] Tool timeout + idempotent retry simulation  
- [ ] Warm transfer creates CRM ticket with summary  
- [ ] Webhook signature + replay protection  
- [ ] Kill switch: disable outbound dialer globally  

## 10. Anti-patterns

- Calling HubSpot/Salesforce directly from the LLM prompt thread without a tool layer  
- Logging full card numbers / OTPs  
- Synchronous PDF generation inside a live turn  
- One shared API key across all tenants with no attribution  
- Treating webhook delivery as guaranteed without a durable outbox  

## Next

→ [06 — Voice models guide](06-voice-models.md)
