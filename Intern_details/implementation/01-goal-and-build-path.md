# 01 — Goal & Build Path

> **Implements:** Overview §1  
> **Outcome:** You can define MVP scope and choose DIY vs BYOK vs bundled vs speech-native.

## 1. What you are building

An AI calling service is a **realtime conversational pipeline over telephony**, not chat-with-microphone.

Required capabilities:

1. Place or receive phone calls (PSTN / mobile / SIP) at scale
2. Understand speech in realtime (STT / ASR)
3. Reason and act (LLM + tools / business logic)
4. Speak back (TTS or speech-to-speech)
5. Orchestrate turn-taking, barge-in, transfers, DTMF, hangup
6. Integrate with domain systems (CRM, calendar, OMS, payments)
7. Operate: observability, evals, compliance, concurrency, cost controls

## 2. Definition of done (capability checklist)

Use this as a backlog. Mark each cell: `N/A` / `MVP` / `GA`.


| Layer         | Capability                       | MVP? | Notes for your product |
| ------------- | -------------------------------- | ---- | ---------------------- |
| Telephony     | Inbound DID                      |      |                        |
| Telephony     | Outbound dial                    |      |                        |
| Telephony     | Call transfer / handoff          |      |                        |
| Telephony     | DTMF collect                     |      |                        |
| Telephony     | Recording + disclosure           |      |                        |
| Media         | Bidirectional stream (mulaw/PCM) |      |                        |
| Speech        | Streaming STT                    |      |                        |
| Speech        | Streaming TTS (or S2S)           |      |                        |
| Speech        | VAD / endpointing                |      |                        |
| Speech        | Barge-in                         |      |                        |
| Intelligence  | System prompt + tools            |      |                        |
| Intelligence  | Guardrails / allowlists          |      |                        |
| Orchestration | Session state machine            |      |                        |
| Orchestration | Timeouts / no-input              |      |                        |
| Orchestration | Human handoff                    |      |                        |
| Platform      | Multi-tenant agent config        |      |                        |
| Platform      | Webhooks / call events           |      |                        |
| Platform      | Per-minute / concurrency billing |      |                        |
| Trust         | Consent / DNC / quiet hours      |      |                        |
| Trust         | PII redaction in logs            |      |                        |


## 3. Build-path decision


| Path                            | You own                                       | Buy / rent                                                     | Choose when                                                 |
| ------------------------------- | --------------------------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------- |
| **A. Full DIY**                 | SIP/media server, orchestrator, provider SDKs | STT/TTS/LLM APIs (or self-host)                                | Hard latency/residency constraints; platform is the product |
| **B. Orchestration + BYOK**     | Prompts, tools, product UX, billing           | Orchestrator (Vapi/Retell-class) + your STT/LLM/TTS/CPaaS keys | Default for most product teams                              |
| **C. Bundled voice-agent SaaS** | Domain config + integrations                  | Almost everything else                                         | Fastest POC; accept less stack control                      |
| **D. Speech-native API**        | Tools + telephony bridge + product            | Single realtime multimodal API                                 | Want fewer moving parts; OK with vendor lock-in             |


**Recommended default for a new service:** start on **B**, keep interfaces so you can swap STT/TTS/LLM later, move pieces to **A** only when cost, latency, or data residency forces it.

### Decision questions (answer before writing code)

1. Do you sell **minutes as a platform** to many tenants, or one internal voice agent?
2. Must audio/transcripts stay in a specific region?
3. Target launch concurrency (simultaneous calls)?
4. Inbound, outbound, or both in v1?
5. Human handoff required on day one?

## 4. MVP slice (suggested)

Ship one **narrow task** end-to-end:

- One language / locale  
- One direction (usually inbound *or* outbound, not both)  
- One happy-path outcome (e.g. book appointment, qualify lead)  
- One CRM/tool write  
- Explicit human escalation phrase  
- Recording disclosure + basic transcript store  
- Latency logged per turn

Defer: multi-language, custom voices at scale, self-hosted ASR, fancy backchannels, full contact-center CTI.

## 5. Work breakdown (engineering workstreams)

```text
W1 Telephony     → numbers, inbound answer OR outbound dial, media stream
W2 Orchestrator  → session, VAD, STT→LLM→TTS loop, barge-in
W3 Tools         → 1–3 domain APIs with timeouts + idempotency
W4 Observability → call_id timeline, redacted transcript, cost tags
W5 Evals         → 30–50 golden dialogues + task success metric
W6 Compliance    → disclosure, retention, outbound consent if needed
```

Parallelize W1+W3 early; W2 is the critical path; W5 must exist before prompt thrash.

## 6. Interface boundaries to freeze early

Define these contracts even if the first implementation is a vendor SDK:


| Boundary        | Minimal contract                                                                               |
| --------------- | ---------------------------------------------------------------------------------------------- |
| `TelephonyPort` | `startInbound`, `startOutbound`, `streamAudio`, `sendAudio`, `transfer`, `hangup`, DTMF events |
| `SttPort`       | streaming partials + finals + language hint                                                    |
| `LlmPort`       | streaming tokens + tool calls                                                                  |
| `TtsPort`       | streaming audio frames + interrupt/cancel                                                      |
| `ToolPort`      | typed JSON in/out, timeout, idempotency key                                                    |
| `CallStore`     | call metadata, turns, tool traces, cost                                                        |


Freezing ports lets you change Deepgram→self-hosted Whisper or Vapi→custom orchestrator without rewriting the product.

## 7. Exit criteria for this phase

- Build path A/B/C/D chosen and written down  
- MVP checklist filled  
- One use case selected (see [02-use-cases.md](02-use-cases.md))  
- Port interfaces sketched (even as TypeScript/Go interfaces or OpenAPI stubs)

## Next

→ [02 — Use cases & readiness](02-use-cases.md)