# AI Voice Calling as a Service — Overview

> **Purpose of this doc:** A detailed reference for what it takes to build AI voice calling as a service. It answers the core product, telephony, model, integration, cost, and evaluation questions.  
> **Engineer implementation docs:** [implementation/README.md](implementation/README.md) — actionable playbooks mapped from each section below.  
> **Out of scope:** Model training/fine-tuning internals.  
> **Audience:** Engineers and technical leads implementing voice agents in any domain without training their own models.  
> **Pricing note:** Vendor rates move often. Treat dollar figures as order-of-magnitude guidance (as of mid‑2026), not contracts.

---

## 1. Goal: What does it take to build AI voice calling as a service?

Building AI voice calling as a service means offering a system that can:

1. **Place or receive phone calls** (PSTN / mobile / SIP) at scale.
2. **Understand speech in real time** (speech-to-text / ASR).
3. **Reason and decide** what to say next (LLM + tools / business logic).
4. **Speak back naturally** (text-to-speech / TTS, or speech-to-speech).
5. **Orchestrate the loop** with low latency, barge-in, turn-taking, and call control (transfer, hold, hangup, DTMF).
6. **Integrate** with CRMs, calendars, ticketing, payments, and domain APIs.
7. **Operate** with observability, evals, compliance, concurrency limits, and cost controls.

You are building (or assembling) a **real-time conversational pipeline over telephony**, not just “chat with voice.”

### 1.1 Capability checklist (what “done” looks like)

| Layer | Must-have capabilities |
| --- | --- |
| Telephony | Inbound DIDs, outbound dialing, SIP trunks, number provisioning, call recording, transfer, DTMF, caller ID |
| Media | Bidirectional audio streaming (typically PCM/mulaw 8 kHz or higher), jitter buffers, codec handling |
| Speech | Streaming STT, streaming TTS (or S2S), VAD / endpointing, barge-in |
| Intelligence | LLM (or speech-native model), system prompts, tools/function calling, memory, guardrails |
| Orchestration | Session state, turn-taking, timeouts, fallbacks, human handoff |
| Platform | Multi-tenant config, auth, webhooks, analytics, billing by minute/concurrency |
| Trust | Consent, recording disclosure, PII handling, regional data residency, spam/compliance rules |

### 1.2 Build vs buy spectrum

| Approach | What you own | Typical use |
| --- | --- | --- |
| **Full DIY** | SIP + media server + STT/TTS/LLM wiring | Maximum control, highest eng cost |
| **Orchestration platform (BYOK)** | Prompt, tools, UX; keys for STT/LLM/TTS/telco | Fastest path for most product teams (e.g. Vapi, Retell-style) |
| **Bundled voice agent SaaS** | Mostly config + integrations | Lowest eng effort; less stack control |
| **Speech-native APIs** | Single multimodal realtime API + telephony bridge | Fewer moving parts; less model mix-and-match |

Most “AI calling as a service” products start on orchestration + BYOK, then selectively self-host or swap components for latency, cost, or data residency.

---

## 2. Use cases

AI calling is valuable wherever a phone conversation is still the default channel for action or trust.

### 2.1 Common use-case families

| Family | Direction | Examples | Success metric ideas |
| --- | --- | --- | --- |
| Sales & growth | Outbound / inbound | Lead qualification, demo booking, reactivation | Connect rate, booking rate, talk-time |
| Collections & reminders | Outbound | Payment reminders, appointment confirmations | Promise-to-pay, confirm/cancel rate |
| Support & IVR replacement | Inbound | Tier‑1 support, order status, password resets | Containment %, AHT, CSAT, escalation quality |
| Operations | Both | Dispatch, field-tech scheduling, logistics ETA | First-call resolution, SLA adherence |
| Healthcare / regulated | Both | Appointment scheduling, intake triage (with strict compliance) | Accuracy, consent, audit completeness |
| Surveys & research | Outbound | NPS, market research | Completion rate, data quality |
| Internal tools | Inbound/outbound | On-call paging, voice ops for staff | Time-to-ack, error rate |

### 2.2 Inbound vs outbound (product implications)

- **Inbound:** User dials your number → AI answers. Needs DIDs/SIP, IVR-like routing, business hours, queue/handoff.
- **Outbound:** You dial the user → AI speaks. Needs dialer logic, retries, timezone/quiet hours, DNC/consent, AMD (answering machine detection).
- **Hybrid:** Missed call → callback AI; or AI qualifies then warm-transfers to a human.

### 2.3 What makes a use case “voice-AI ready”

Good fit when:

- The conversation is **structured enough** (clear intents, finite outcomes).
- Latency tolerance is roughly **sub‑second to ~1.5s** turn delay for acceptable UX.
- There is a **system of record** the agent can read/write (CRM, OMS, calendar).
- Failure modes are tolerable via **escalation to human** or safe fallback.

Poor fit (initially) when:

- High-stakes open-ended counseling with weak guardrails.
- Heavy dialect/noise with no eval data.
- Legal requirement for a licensed human on every interaction (unless AI is clearly assistive).

---

## 3. What is AI calling all about?

At a high level: **telephony media + conversational AI**.

```text
Caller <--PSTN/Mobile/SIP--> Telco/CPaaS <--WebRTC/SIP/WS audio--> Voice orchestration
                                                                      |
                                                    STT --> LLM/Tools --> TTS
                                                                      |
                                                              Business APIs / CRM
```

There are two architectural styles:

1. **Cascaded (most common today):** Audio → STT → LLM → TTS → Audio.
2. **Speech-to-speech / realtime multimodal:** Audio (and partial text) in → model → audio out, with less explicit STT/TTS chaining (e.g. OpenAI Realtime-style, some Gemini/LiveKit paths).

Cascaded stacks are easier to debug and swap; S2S can feel more natural and lower latency but is harder to control/tool/evaluate textually.

### 3.1 Is it VoIP?

**Yes, in practice the AI side almost always runs over VoIP/IP media**, even when the human is on a normal mobile/landline.

Clarify the layers:

| Term | Meaning |
| --- | --- |
| **PSTN** | Public switched telephone network — the global phone system users know as “calling a number.” |
| **VoIP** | Voice over IP — audio packets over the internet (SIP, WebRTC, proprietary streams). |
| **CPaaS** | Communications Platform as a Service (Twilio, Telnyx, Vonage, Plivo, Exotel, etc.) — APIs to buy numbers, place/receive calls, stream audio. |
| **SIP trunk** | A VoIP connection from a carrier/PBX into your media stack. |
| **WebRTC** | Browser/app realtime media; often used for “call from web,” less often for pure PSTN unless bridged. |

**Typical production path:**

1. User’s phone hits PSTN.
2. Carrier interconnects into a CPaaS / SIP provider.
3. Provider streams audio to your server (WebSocket media stream, SIP, or similar).
4. Your AI pipeline processes audio and streams speech back.
5. Provider bridges AI audio back onto the PSTN leg.

So: **users do not need VoIP apps**; **your platform does use VoIP/IP media behind the scenes**.

### 3.2 Do they need to buy SIM cards?

**End users:** No. They just receive/make normal phone calls.

**Your service:** Almost never “buy SIMs and stick them in modems” for a cloud AI calling product. That GSM-modem / SIM-farm pattern exists in niche/legacy setups and is a poor fit for scalable, compliant AI calling.

What you actually buy/provision:

| Asset | Who provides it | Notes |
| --- | --- | --- |
| **Phone numbers (DIDs)** | CPaaS / carriers | Local, toll-free, mobile numbers depending on country |
| **SIP trunks / voice minutes** | Carriers / CPaaS | Billed per minute + number rental |
| **Caller ID / branded calling** | Carrier features / partners | Varies heavily by country |
| **SIM cards** | Rarely needed | Only for special mobile-originated hardware/IoT cases |

**Country reality check (important):** Number types, KYC, spam registration (e.g. US STIR/SHAKEN, India DLT/TRAI-style constraints for commercial SMS/voice), and outbound consent rules differ by jurisdiction. For India specifically, commercial calling often involves registered headers/templates for SMS and careful use of permitted voice routes — work with a local CPaaS (Exotel, Knowlarity, Twilio local inventory, etc.) rather than assuming US Twilio defaults.

### 3.3 How do calls get routed to users / agents?

Routing has two meanings: **telephony routing** and **conversation routing**.

#### Telephony routing (how the call finds a destination)

**Outbound**

1. Your app decides to call `+91…` / `+1…`.
2. Orchestrator asks CPaaS to create an outbound call with a webhook/stream URL.
3. CPaaS/carrier routes through interconnects to the destination network.
4. When answered (human/voicemail), media stream opens to your AI.

**Inbound**

1. You purchase/configure a number and point it at a voice app / SIP endpoint / webhook.
2. Caller dials the number.
3. Carrier → CPaaS → your webhook answers and attaches a media stream / TwiML-like instructions / SIP app.
4. AI session starts; optionally later transfers to a human agent SIP endpoint or PSTN number.

**To human agents (handoff)**

- Warm/cold transfer via CPaaS `Dial`/`Refer`.
- Bridge into contact-center platforms (Genesys, Five9, Amazon Connect, native PBX).
- “Whisper” context to agent (text summary) while customer is on hold — *this is contact-center jargon, not Wispr the product*.

#### Conversation routing (how the AI decides path)

- Intent classification / LLM tool choice.
- State machines or graphs for regulated flows.
- Skills-based routing after AI qualification (language, topic, VIP).
- Queueing, callback, and escalation policies.

---

## 4. Reference architecture

### 4.1 Core components

```text
┌─────────────────────────────────────────────────────────────────┐
│                        Client / Channel                          │
│         PSTN phone | SIP desk phone | Web softphone              │
└───────────────────────────────┬─────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────┐
│                 Telephony / Media edge (CPaaS)                   │
│     Numbers, SIP, recording, DTMF, AMD, stream forks             │
└───────────────────────────────┬─────────────────────────────────┘
                                │ realtime audio (WS/SIP/RTP)
┌───────────────────────────────▼─────────────────────────────────┐
│                    Voice orchestration service                   │
│  session · VAD/endpointing · barge-in · turn policy · timers     │
│  prompt/runtime · tool router · guardrails · handoff             │
└─────┬───────────────────┬───────────────────┬───────────────────┘
      │                   │                   │
      ▼                   ▼                   ▼
   STT API             LLM API             TTS API
 (Deepgram/             (GPT/Claude/      (ElevenLabs/
  Whisper/…)            Gemini/…)          Cartesia/…)
      │                   │                   │
      └────────────┬──────┴─────────┬─────────┘
                   ▼                ▼
            Business tools     Data stores
         (CRM, calendar,     (transcripts,
          payments, OMS)      call metrics,
                              eval datasets)
```

### 4.2 Latency budget (cascaded stack)

A usable phone agent usually targets **~600–1200 ms** from end-of-user-speech to first audio back (p50), with p95 higher.

Rough breakdown:

| Stage | Typical budget |
| --- | --- |
| Endpointing / VAD decision | 200–600 ms (policy-dependent) |
| STT final/partial for turn | 100–400 ms |
| LLM first token | 150–600 ms |
| TTS time-to-first-byte | 100–400 ms |
| Network / jitter | 50–200 ms |

**Design implication:** Streaming everywhere (partial STT → speculative LLM → streaming TTS) matters more than picking a slightly smarter non-streaming model.

### 4.3 Critical realtime behaviors

- **VAD / endpointing:** Decide when the user finished speaking.
- **Barge-in:** User interrupts TTS; stop playback, cancel LLM/TTS work, listen again.
- **Interruption handling:** Don’t leave half-spoken sentences that confuse state.
- **Backchannels:** Optional “mm-hmm” / “okay” — use carefully; can feel fake.
- **Silence / no-input timeouts:** Reprompt, then escalate/hang up.
- **DTMF fallback:** Menus or ID entry when ASR fails.
- **Answering machine detection (AMD):** For outbound, branch to voicemail drop vs live conversation.

---

## 5. Integrations

Integrations are what turn a demo voice bot into a service.

### 5.1 Telephony & contact center

- **CPaaS:** Twilio, Telnyx, Vonage, Plivo, Bandwidth, Exotel, etc.
- **SIP / PBX:** Asterisk, FreeSWITCH, Kamailio, cloud SIP trunks.
- **Contact center:** Amazon Connect, Genesys, Aircall, Talkdesk, custom CCR.
- **Recording & compliance storage:** Dual-channel recording, redaction pipelines.

### 5.2 AI / speech providers

- **STT:** Deepgram, AssemblyAI, Google Speech-to-Text, Azure Speech, AWS Transcribe, OpenAI Whisper API, self-hosted Whisper/faster-whisper, Gladia, etc.
- **TTS:** ElevenLabs, Cartesia, PlayHT, Deepgram Aura, Azure/Google Neural voices, OpenAI TTS, CosyVoice / open TTS stacks.
- **LLM / realtime:** OpenAI, Anthropic, Google Gemini, open weights via vLLM/TGI, speech-realtime APIs.
- **Orchestration platforms:** Vapi, Retell, LiveKit Agents, Pipecat, Daily, custom.

### 5.3 Business systems

- CRM (Salesforce, HubSpot, Zoho)
- Ticketing (Zendesk, Freshdesk, Jira)
- Calendar / scheduling (Google, Outlook, Calendly)
- Payments / billing
- OMS / inventory / logistics
- Auth / customer identity (phone OTP, account lookup)
- Data warehouses for analytics (BigQuery, Snowflake, ClickHouse)

### 5.4 Platform plumbing

- Webhooks for call events (`started`, `answered`, `ended`, `tool_error`)
- Config APIs for agents, prompts, voices, numbers
- Secrets management for provider keys
- Feature flags / prompt versioning
- Idempotent tool calls (double-speak / retry safety)

### 5.5 Integration design rules

1. **Tools must be fast** — aim for tool latency budgets (e.g. <300–500 ms) or speak a filler (“let me check that”) with care.
2. **Phone identity ≠ authenticated identity** — verify before sensitive actions.
3. **Write audit logs** for every tool call and disclosure.
4. **Design explicit handoff contracts** (summary, reason code, customer ID, last intent).

---

## 6. Voice models

“Voice models” is an umbrella term. In production stacks you usually meet three families:

1. **ASR / STT models** — speech → text  
2. **TTS models** — text → speech  
3. **Speech-native / multimodal conversational models** — speech ↔ speech (sometimes with text side channel)

Plus the **LLM** that does language reasoning in cascaded stacks.

### 6.1 What are some voice models / systems?

#### Speech-to-text (ASR)

| Model / system | Type | Notes |
| --- | --- | --- |
| OpenAI Whisper (and large-v3 / turbo) | Open weights + API | Strong batch ASR; native streaming is non-trivial |
| faster-whisper / whisper.cpp | Open inference runtimes | Production local Whisper serving |
| Deepgram Nova family | Closed API | Popular for low-latency streaming phone ASR |
| Google / Azure / AWS speech | Closed cloud | Strong enterprise + language coverage |
| AssemblyAI, Gladia | Closed API | Streaming + features (diarization, etc.) |
| Distil-Whisper / other distilled ASR | Open | Faster/cheaper approximations |

#### Text-to-speech

| Model / system | Type | Notes |
| --- | --- | --- |
| ElevenLabs | Closed | High realism; common in agents; often a top cost line |
| Cartesia Sonic | Closed | Low-latency streaming TTS |
| OpenAI TTS / gpt speech voices | Closed | Simple integration with OpenAI stack |
| Deepgram Aura | Closed | Cost-competitive agent voices |
| Azure / Google WaveNet-style neural voices | Closed | Broad language/locale coverage |
| Open TTS (XTTS, CosyVoice, StyleTTS2, Piper, etc.) | Open / mixed | Self-host for cost/residency; quality/latency vary |

#### Speech-to-speech / realtime

| System | Notes |
| --- | --- |
| OpenAI Realtime API | Multimodal voice sessions; telephony via SIP/WebRTC bridges |
| Gemini Live / speech variants | Google’s realtime conversational speech path |
| Vendor “voice agents” (ElevenLabs ConvAI, Deepgram Voice Agent, etc.) | Bundled STT+LLM+TTS or speech pipelines |

### 6.2 How are they similar / different from LLMs?

| Dimension | Typical LLM | ASR model | TTS model | Speech-realtime model |
| --- | --- | --- | --- | --- |
| Input | Tokens/text (sometimes images) | Audio features / spectrograms / waveforms | Text (sometimes phonemes/prosody controls) | Audio (+ optional text) |
| Output | Tokens/text (tools) | Text / word timestamps | Waveform / audio frames | Audio (+ text transcripts) |
| Core job | Reason, plan, tool-call | Recognize speech | Synthesize speech | Converse with low latency |
| Training objective | Next-token (and variants) | Sequence recognition / CTC/RNNT/Attention ASR | Spectrogram/waveform generation | Joint speech understanding + generation |
| Controllability | Prompt + tools | Decoding params, hotwords, language hints | Voice ID, style, stability, speed | Session instructions; often less “tool-native” than text LLMs |
| Eval metrics | Exact match, rubrics, task success | WER/CER, latency | MOS/preference, pronunciation, latency | Task success + latency + naturalness |

**Similarity:** All are large neural nets, often Transformer-based, trained at scale, served via APIs or GPU inference.

**Difference:** ASR/TTS are **signal ↔ text transducers**; LLMs are **language reasoners**. A phone agent needs both transduction *and* reasoning (unless a single speech-native model covers the session).

### 6.3 What’s their “token” concept?

Tokens mean different things in each layer:

| Layer | What gets metered / modeled | Practical meaning |
| --- | --- | --- |
| **LLM** | Subword tokens (BPE/SentencePiece) | Prompt + completion tokens drive $ and context limits |
| **ASR** | Often **audio duration** (seconds/minutes), sometimes characters | Phone ASR APIs usually bill per minute of audio; internal models may chunk audio into frames (e.g. 10–30 ms) — not LLM tokens |
| **TTS** | **Characters**, credits, or audio minutes | ElevenLabs-style character/credit billing, or per-minute conversational billing |
| **Realtime speech models** | Audio tokens / discrete audio units + text tokens (vendor-specific) | Vendors expose “audio tokens” or per-minute session pricing; not interchangeable with GPT text tokens |

**Rule of thumb for engineers:**

- Don’t assume Whisper “tokens” == GPT tokens.
- For capacity planning, convert everything to **$ / minute of call** and **GPU-seconds / minute** if self-hosting.
- Context windows for LLMs still matter: long transcripts + RAG + tool JSON can blow token budgets mid-call.

### 6.4 Transcription: real-time vs non-real-time

| Mode | Latency profile | Use in calling | Examples |
| --- | --- | --- | --- |
| **Batch / offline** | Seconds to minutes after audio completes | Post-call notes, compliance archive, training sets | Whisper batch, async cloud jobs |
| **Streaming partials** | Hundreds of ms; interim hypotheses | Live captions, endpointing aids, speculative LLM start | Deepgram live, cloud streaming STT |
| **Streaming finals** | On utterance end | Canonical turn text for tools/logging | Most agent stacks |
| **Pseudo-streaming Whisper** | Chunked windows (~0.5–3s+) | Self-hosted near-realtime when true streaming ASR unavailable | whisper_streaming, chunked faster-whisper |

For **live calling**, you want **streaming ASR with interim results** plus a clear finalization policy. Pure batch Whisper is excellent for post-call, weak as a naïve realtime engine.

**Post-call transcription** can use a higher-accuracy (slower/costlier) model than the live path — common pattern: fast streaming model live, Whisper-class model offline for CRM notes.

### 6.5 Where do these models run? GPU vs CPU?

| Workload | Typical production hardware | Why |
| --- | --- | --- |
| Cloud STT/TTS/LLM APIs | Vendor GPUs (you don’t manage) | Easiest ops |
| Self-hosted Whisper large / ASR | **GPU** preferred for concurrency | Realtime factor & batching |
| Small ASR / quantized Whisper | **CPU** possible for low concurrency | Cost/edge; watch RTF > 1 |
| Neural TTS (high quality) | **GPU** common | Autoregressive/flow/diffusion-style synthesis is heavy |
| Small TTS (Piper, etc.) | CPU often fine | Lower quality/expressiveness |
| LLM (7B–70B+) | GPU (or specialized inference hardware) | Latency-sensitive agents need fast TTFT |
| VAD / audio DSP | CPU | Cheap classical or tiny models |

**Realtime factor (RTF):** processing_time / audio_duration. For live calls you need RTF ≪ 1 with headroom for concurrency.

### 6.6 Quantization

Quantization reduces numeric precision of weights/activations (FP32 → FP16/BF16 → INT8 → INT4/GGUF-style) to cut memory and often increase throughput.

| Precision | Typical use | Trade-off |
| --- | --- | --- |
| FP32 | Reference | Too heavy for serving |
| FP16 / BF16 | Default GPU serving | Near-reference quality |
| INT8 / INT8_FP16 | Common Whisper/LLM serving (faster-whisper CTranslate2, TensorRT-LLM, etc.) | Small WER/quality hit; big VRAM win |
| 4‑bit / GGUF Q4–Q5 | Edge LLM / some local speech stacks | Larger quality risk; evaluate carefully |

For Whisper-class ASR, community runtimes (faster-whisper, whisper.cpp) make INT8/FP16 standard. Studies report sizable model-size reductions with limited WER impact when done well — **still validate on your accents and telephony audio (8 kHz, compression, noise)**.

**Telephony caveat:** Phone audio is often narrowband/compressed. A model that wins on LibriSpeech may degrade on real PSTN audio — quantization error + domain shift compound. Eval on *your* call recordings.

### 6.7 Open models vs closed models

| | Open weights | Closed API |
| --- | --- | --- |
| **Examples** | Whisper, many TTS/LLM weights | Deepgram, ElevenLabs, OpenAI Realtime |
| **Pros** | Data residency, cost at scale, customization, no per-vendor lock-in on rates | Best latency/quality per eng-hour, managed scaling, features (diarization, turn detection) |
| **Cons** | You own GPUs, SRE, model upgrades, abuse, streaming glue | Data leaves your perimeter; pricing/policy changes; less control |
| **Typical choice** | Regulated/high-volume after product-market fit | MVP → production for most startups |

Hybrid is common: closed streaming STT + closed TTS + LLM of choice; self-host Whisper for post-call; later self-host TTS if invoices hurt.

### 6.8 Wispr (how it fits — and how it doesn’t)

**Wispr Flow** ([wisprflow.ai](https://wisprflow.ai)) is an **AI dictation / voice-to-text product** for typing into apps (Mac/Windows/iOS/Android). It focuses on polished dictation: filler removal, self-correction handling, personal vocabulary, multi-app paste.

It is **not** a telephony AI calling platform:

| | Wispr Flow | AI voice calling stack |
| --- | --- | --- |
| Primary job | Dictate text into software | Hold a two-way phone conversation |
| Channel | Desktop/mobile OS input | PSTN/SIP + realtime media |
| Turn-taking | Push-to-talk / hotkey UX | Full-duplex call, barge-in, telephony events |
| Typical output | Clean text in a text field | Spoken audio to a caller + tool actions |

**Why mention it in this doc:** People often conflate “voice AI” categories. Wispr-like products are **productivity ASR + rewriting**. Calling-as-a-service needs **telephony + conversational loop + TTS + tools**. You might use similar underlying ASR ideas, but the product architecture diverges sharply.

*(Contact-center “whisper” — playing a private prompt only the human agent hears — is an unrelated term.)*

### 6.9 How would you run evals for a specific use case?

Evals must mirror the **phone task**, not just generic WER or chatbot BLEU.

#### Layered eval strategy

1. **Component evals**
   - **ASR:** WER/CER on domain audio (accents, product names, phone noise). Track entity error rate (names, order IDs, amounts).
   - **TTS:** Preference tests, pronunciation glossary accuracy, latency TTFB.
   - **LLM policy:** Intent accuracy, tool-call correctness, hallucination rate, safety refusals.

2. **Task / outcome evals (most important)**
   - Did the agent book the appointment / resolve the ticket / collect the right fields?
   - Exact-match on structured slots extracted during the call.
   - Compliance script adherence (disclosures said?).

3. **Conversation quality**
   - Human rubrics: helpfulness, interruption handling, repetition, unnatural delays.
   - LLM-as-judge with caution + human calibration set.

4. **Latency / reliability SLOs**
   - Time-to-first-speech, time-between-turns, barge-in success, crash/timeout rate.

5. **Online evals**
   - Shadow mode on live traffic, A/B voices/prompts, post-call CSAT, containment vs escalation.

#### Practical eval harness

- Build a **golden set** of 50–200 real/synthetic calls per use case.
- Store: audio, transcript, expected slots, expected tools, expected outcome.
- Automate regression on every prompt/model/voice change.
- Separate **language packs** (en-IN, hi-IN, en-US…) — never assume one WER number transfers.
- Include adversarial cases: crosstalk, long silence, angry user, wrong number, partial account numbers.

#### Minimal metric dashboard for a booking agent (example)

| Metric | Target idea |
| --- | --- |
| Booking success (simulated) | ≥ X% |
| Critical entity accuracy | ≥ 99% on phone numbers / dates |
| Undisclosed recording (if required) | 0% |
| Median turn latency | ≤ 1.0–1.2 s |
| Unwanted hangups | ≤ Y% |
| Escalation appropriateness | High precision/recall vs rubric |

### 6.10 Pricing per minute?

Think in **stacked cost layers**, not one sticker price.

#### Cost stack (order of magnitude, mid‑2026)

| Layer | Ballpark | Notes |
| --- | --- | --- |
| Telephony | ~$0.01–$0.03 / min (US); varies widely internationally | Inbound ≠ outbound; number rental extra |
| STT streaming | ~$0.005–$0.02 / min | Premium ASR higher |
| LLM | ~$0.005–$0.08 / min | Depends on model + transcript length + tools |
| TTS | ~$0.03–$0.12 / min | Often the largest AI line item |
| Orchestration platform | ~$0.05–$0.12 / min (if used) | BYOK means this is *on top of* STT/LLM/TTS |
| **All-in typical** | **~$0.08–$0.30 / min** | Bundled vs BYOK changes the split |

Examples of market shapes (approximate, verify before budgeting):

- **BYOK orchestrators** (e.g. Vapi-style): low platform fee + you pay Deepgram/OpenAI/ElevenLabs/Twilio separately.
- **Bundled platforms** (Retell/Bland/etc.): higher single rate; fewer invoices; less component control.
- **Speech-native APIs:** sometimes ~$0.20–$0.30+/min depending on model tier.

#### Cost control levers

- Shorter prompts / smaller models for triage turns.
- Cheaper TTS for IVR-like nodes; premium voice only for sales.
- End calls cleanly; avoid billing silence if your vendor allows (policy varies).
- Cache TTS for repeated phrases (“Please enter your PIN”).
- Self-host ASR/TTS only when volume justifies GPU ops.
- Concurrency caps to prevent runaway spend.

#### Pricing your own “calling as a service”

Your price must cover:

1. COGS per minute (stack above)  
2. Number / KYC / compliance overhead  
3. Engineering + on-call  
4. Failed-call & AMD waste  
5. Margin  
6. Minimums / included concurrency tiers (common SaaS packaging)

---

## 7. Additional points necessary to achieve the goal

These are easy to miss in demos and fatal in production.

### 7.1 Product & conversation design

- Script the **happy path** and the **top 20 failure paths**.
- Define persona, speaking pace, and interruption policy.
- Decide languages + code-switching (critical in India and multilingual markets).
- Explicit confirmation for high-risk actions (payments, cancellations).

### 7.2 Compliance, consent, and spam

- Call recording disclosure where required.
- Outbound consent / DNC / quiet hours / frequency caps.
- Region-specific telecom rules and number reputation (STIR/SHAKEN, local spam registries).
- PCI/PHI: avoid storing raw card audio; use DTMF redaction / hosted fields.
- Data retention and deletion SLAs for transcripts/recordings.

### 7.3 Reliability & scale

- Horizontal scaling of orchestrators; sticky session affinity for a call.
- Provider failover (STT/TTS/LLM backups).
- Back-pressure when concurrency exceeds GPU/API limits.
- Idempotent dialers; prevent double-calling.

### 7.4 Observability

- Per-call timeline: media connected → first agent audio → each turn → tools → hangup.
- Redacted transcript store + searchable metadata.
- Alerts on WER proxies (e.g. repeated “sorry?”), latency spikes, tool error rates.
- Cost attribution per tenant / campaign / model.

### 7.5 Security

- Secure webhooks (signatures), short-lived stream URLs.
- Least-privilege tool credentials.
- PII minimization in prompts and logs.
- Tenant isolation for multi-tenant SaaS.

### 7.6 Human-in-the-loop

- Warm transfer with summary.
- Agent assist (AI whispers suggestions to humans) vs AI-autonomous.
- QA review queues for sampled calls.

### 7.7 Testing beyond model evals

- Load tests for concurrent calls.
- Chaos: kill TTS mid-sentence, stall tool API, packet loss.
- Device matrix: mobile speakerphone, feature phones, call-center headsets.
- Locale tests: number formats, date/time, currency pronunciation.

### 7.8 Analytics & learning loop

- Outcome events in your warehouse.
- Prompt/version experiment framework.
- Glossary updates from ASR misses (brand names, SKUs).
- Continuous hard-negative mining from failed calls.

### 7.9 Packaging a SaaS

- Tenant → agents → phone numbers → tools → analytics.
- Plans by minutes + max concurrency + premium voices.
- Usage meters with burst protection.
- Admin console for prompts/voices without redeploying code.

---

## 8. End-to-end mental model (one page)

**AI voice calling as a service =**

1. **Numbers & carriers** to reach real phones (not SIMs in your servers).  
2. **Realtime media** bridge into your orchestrator.  
3. **Speech models** to hear and speak (API or self-hosted).  
4. **LLM + tools** to do work in a domain system.  
5. **Policies** for latency, barge-in, handoff, and compliance.  
6. **Evals + observability + cost controls** so it improves instead of drifting.

If you can explain those six for a specific domain (e.g. “clinic appointment booking in en-IN/hi-IN”), you can implement the use case without training new foundation models.

---

## 9. Engineer implementation docs

Section-by-section playbooks live in [`implementation/`](implementation/README.md):

| Overview | Implementation doc |
| --- | --- |
| §1 Goal | [01-goal-and-build-path.md](implementation/01-goal-and-build-path.md) |
| §2 Use cases | [02-use-cases.md](implementation/02-use-cases.md) |
| §3 Telephony / VoIP / routing | [03-telephony-primer.md](implementation/03-telephony-primer.md) |
| §4 Architecture | [04-architecture.md](implementation/04-architecture.md) |
| §5 Integrations | [05-integrations.md](implementation/05-integrations.md) |
| §6 Voice models | [06-voice-models.md](implementation/06-voice-models.md) |
| §6.9 Evals | [07-evals.md](implementation/07-evals.md) |
| §6.10 Pricing | [08-cost-and-pricing.md](implementation/08-cost-and-pricing.md) |
| §7 Production extras | [09-production-ops.md](implementation/09-production-ops.md) |
| §8 End-to-end | [10-use-case-template.md](implementation/10-use-case-template.md) |

---

## 10. Glossary

| Term | Definition |
| --- | --- |
| **ASR / STT** | Automatic Speech Recognition / Speech-to-Text |
| **TTS** | Text-to-Speech |
| **S2S** | Speech-to-Speech |
| **VAD** | Voice Activity Detection |
| **WER** | Word Error Rate |
| **DID** | Direct Inward Dial number |
| **SIP** | Session Initiation Protocol |
| **PSTN** | Public Switched Telephone Network |
| **CPaaS** | Communications Platform as a Service |
| **AMD** | Answering Machine Detection |
| **BYOK** | Bring Your Own Key |
| **RTF** | Realtime Factor |
| **Barge-in** | User interrupts the agent’s speech |
| **Warm transfer** | Hand off call to human with context |

---

## Document history

| Version | Date | Notes |
| --- | --- | --- |
| 0.1 | 2026-07-31 | Initial detailed overview for voice AI calling-as-a-service |
