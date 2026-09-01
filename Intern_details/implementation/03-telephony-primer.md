# 03 — Telephony Primer for App Engineers

> **Implements:** Overview §3  
> **Outcome:** You can provision numbers, connect media to your orchestrator, route/transfer calls, and avoid SIM-farm mistakes.

## 1. Mental model

```text
User phone (PSTN/mobile)
    → Carrier interconnect
    → CPaaS / SIP trunk
    → Your answer URL or SIP app
    → Bidirectional audio stream to orchestrator
    → STT / LLM / TTS loop
    → Audio back to caller
```

Users stay on normal phones. **Your stack uses VoIP/IP media** behind the scenes.

| Term | Engineer meaning |
| --- | --- |
| PSTN | The public phone network |
| VoIP | Audio over IP (SIP, WebRTC, vendor WS streams) |
| CPaaS | API for numbers, dial, answer, stream (Twilio, Telnyx, Exotel, …) |
| DID | Phone number that rings into your app |
| SIP trunk | VoIP trunk from carrier/PBX into media stack |
| ANI / caller ID | Who is calling (spoofable; not auth) |
| AMD | Answering machine detection (outbound) |

## 2. Do you need SIM cards?

| Party | Need SIMs? |
| --- | --- |
| End users | No |
| Your cloud AI service | **No** — provision DIDs + SIP/CPaaS minutes |
| Niche GSM modem / SIM farm | Avoid for scalable compliant products |

**Provision instead:** phone numbers, voice minutes, optional branded calling, SIP endpoints.

### Regional note (India-oriented teams)

Commercial calling rules, number KYC, and spam controls differ from US defaults. Prefer a CPaaS with local inventory (Exotel, Knowlarity, Twilio local numbers, etc.) and involve compliance early for outbound campaigns. Do not assume US Twilio patterns transfer cleanly.

## 3. Inbound call flow (implement this first if possible)

```text
1. Buy/configure DID on CPaaS
2. Point number → Voice webhook / SIP URI
3. On incoming call event:
   a. Return instructions: answer + start media stream to wss://your-orchestrator/...
   b. Or respond with SIP 200 and RTP to your media server
4. Orchestrator accepts stream, starts STT/TTS session
5. On hangup webhook: finalize transcript, billing, CRM write
```

### Webhook responsibilities

| Event | Your job |
| --- | --- |
| `incoming` / `ringing` | Create `call_id`, choose agent config, answer |
| `stream_connected` | Bind audio sockets, start VAD/STT |
| `dtmf` | Feed digits into state machine |
| `completed` | Persist duration, recording URL, cost, outcome |

**Security:** validate webhook signatures; use short-lived stream tokens; TLS everywhere.

## 4. Outbound call flow

```text
1. Your dialer enqueues job {to, from, agent_id, idempotency_key, consent_ref}
2. Check quiet hours / DNC / frequency caps
3. CPaaS create call with answer/stream URL
4. On answer:
   - If AMD=machine → voicemail drop or hangup
   - If AMD=human → start agent session
5. On completion → retries or suppress per policy
```

### Dialer rules to encode

- Idempotency key per logical outreach  
- Max attempts + backoff  
- Timezone-aware quiet hours  
- Concurrent dial caps (separate from answered concurrency)  
- Caller ID reputation monitoring  

## 5. Media: what your orchestrator must handle

| Topic | Practical guidance |
| --- | --- |
| Codec | Often **mulaw 8 kHz** mono on PSTN legs; some streams offer PCM16 16 kHz |
| Frame size | Commonly 20 ms frames; process incrementally |
| Direction | Dual channel if possible (caller vs agent) for analytics |
| Clock | Don’t assume perfect realtime; buffer + pace TTS playout |
| Interrupt | On barge-in, **stop sending TTS frames** and cancel in-flight synthesis |

### Minimal media session API (conceptual)

```text
onAudio(frame)      → push to STT / VAD
playAudio(frame)    → send to CPaaS stream
clearPlayback()     → flush jitter buffer / stop TTS
onDtmf(digit)
hangup(reason)
transfer(target, {whisperSummary})
```

## 6. Routing

### Telephony routing

| Goal | Mechanism |
| --- | --- |
| Number → AI agent | DID webhook / SIP URI mapping in CPaaS |
| AI → human PSTN | CPaaS transfer / dial leg |
| AI → contact center | SIP REFER / queue DNIS / native CC connector |
| Multi-tenant | Map DID or `To` number → `tenant_id` + `agent_id` |

### Conversation routing (after AI starts)

Implement as explicit policy, not free-form LLM guesswork alone:

- Business hours → AI vs voicemail vs callback  
- Language detect → agent variant  
- VIP / account tier → priority queue on escalate  
- Intent → skill queue on warm transfer  

## 7. Human handoff

| Type | Behavior | Implement |
| --- | --- | --- |
| Cold transfer | Drop AI; human answers blind | `transfer(number)` |
| Warm transfer | AI summary to agent; then bridge | Consult/whisper leg or CC “whisper” |
| Scheduled callback | No live bridge | Create ticket + outbound job |

**Handoff payload (minimum):**

```json
{
  "call_id": "...",
  "customer_phone": "+91...",
  "verified": false,
  "intent": "reschedule",
  "slots": {"date": "2026-08-04"},
  "summary": "Patient wants to move Tuesday slot; not verified yet.",
  "reason_code": "user_requested_human",
  "transcript_url": "..."
}
```

## 8. DTMF

Use DTMF when ASR is weak for:

- PINs, OTPs, account numbers  
- Menu fallbacks (“Press 1 for…”)  
- PCI-sensitive digits (prefer vendor PCI connector; never log digit streams)

Flush STT turn when DTMF mode starts to avoid mixed state.

## 9. Recording & dual channel

- Enable recording at CPaaS if legally allowed and disclosed.  
- Prefer dual-channel for speaker separation.  
- Store recording URL + retention policy ID on `call` row.  
- Redact before sharing externally.

## 10. Implementation checklist

- [ ] CPaaS account + KYC for target countries  
- [ ] At least one DID answered by your webhook in staging  
- [ ] Media stream reaches orchestrator; you can echo audio  
- [ ] Hangup webhook closes session cleanly  
- [ ] Transfer tested to a human phone  
- [ ] DTMF tested  
- [ ] Outbound dial + AMD branch (if outbound in scope)  
- [ ] Webhook signature verification on  
- [ ] Call recording + disclosure path decided  

## 11. Common failure modes

| Symptom | Likely cause |
| --- | --- |
| One-way audio | Wrong codec/stream track; not sending frames |
| Robot talks over user | Missing barge-in / playback clear |
| Double greetings | Webhook retried; non-idempotent answer |
| Ghost calls billed | Stream not closed on hangup |
| Wrong tenant agent | DID → tenant mapping bug |
| Outbound spam blocks | Caller ID reputation / missing registration |

## Next

→ [04 — Agent architecture](04-architecture.md)
