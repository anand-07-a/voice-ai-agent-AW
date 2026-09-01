# 08 — Cost & Pricing

> **Implements:** Overview §6.10  
> **Outcome:** You can estimate COGS per minute, meter usage, and package SaaS pricing. Rates are mid‑2026 order-of-magnitude — verify with vendors.

## 1. Cost stack

Billable AI calling cost is layered:

| Layer | Ballpark | Meter |
| --- | --- | --- |
| Telephony | ~$0.01–$0.03/min domestic US; intl varies | Per leg minute + number rental |
| STT | ~$0.005–$0.02/min | Audio duration |
| LLM | ~$0.005–$0.08/min | Tokens ≈ f(transcript, tools, prompt) |
| TTS | ~$0.03–$0.12/min | Chars/credits or minutes (often largest AI line) |
| Orchestration platform | ~$0.05–$0.12/min if BYOK platform used | Platform minutes |
| **Typical all-in** | **~$0.08–$0.30/min** | Sum of layers |

### BYOK vs bundled

| Model | What invoice looks like |
| --- | --- |
| BYOK orchestrator | Low platform fee + separate STT/LLM/TTS/telco invoices |
| Bundled voice agent | Fewer invoices; less component control |
| Speech-native API | Session/minute (+ telephony bridge) |

Always compute **all-in answered minute**, not headline platform rate.

## 2. Unit economics worksheet

Fill for your stack:

```text
Assumptions
  answered_minutes_per_month = ?
  avg_call_minutes = ?
  concurrency_peak = ?

COGS_per_min =
  telco + stt + llm + tts + orchestration + other

Monthly_COGS =
  answered_minutes * COGS_per_min
  + number_rentals
  + fixed_gpu_or_seats
  + compliance_adders

Waste_factor =
  failed dials, AMD voicemail, long silence (if billed)

Fully_loaded_COGS_per_min =
  Monthly_COGS / answered_minutes
```

Track separately: **dialed minutes** vs **answered minutes** (outbound waste is real).

## 3. Metering in your system

Emit per call:

```text
call_id
tenant_id
answered_seconds
telco_seconds_in / telco_seconds_out
stt_seconds
tts_characters (or tts_seconds)
llm_input_tokens / llm_output_tokens
orchestration_seconds
provider_cost_estimate
prompt_version / model_ids / voice_id
```

Roll up to tenant-day for billing. Keep raw provider usage IDs for dispute reconciliation.

## 4. Cost control levers

| Lever | Effect |
| --- | --- |
| Shorter prompts / smaller triage LLM | LLM $ |
| Premium TTS only on sales; cheaper on IVR nodes | TTS $ |
| Cache repeated TTS phrases | TTS $ + latency |
| End calls cleanly; tune no-input | Telco + all layers |
| Concurrency caps + budget alarms | Runaway spend |
| Post-call Whisper batch ≠ live premium ASR | Quality where it matters |
| Self-host at high volume | CapEx/ops vs API COGS |

## 5. Packaging your SaaS

Common dimensions:

| Dimension | Why |
| --- | --- |
| Included minutes | Aligns to COGS |
| Max concurrency | Protects capacity |
| Premium voices / models | Pass-through margin |
| Numbers / locales | Fixed cost recovery |
| HIPAA / BAA / VPC | Compliance overhead |

### Pricing formula (simple)

```text
List_price_per_min >= Fully_loaded_COGS_per_min * (1 + margin)
  + support_overhead_allocation
```

Add minimum monthly commit so idle tenants don’t lose money on numbers + support.

## 6. Example (illustrative only)

3-minute inbound US call, mid-range stack:

| Component | ~/min | ×3 min |
| --- | --- | --- |
| STT | 0.008 | 0.024 |
| LLM | 0.020 | 0.060 |
| TTS | 0.060 | 0.180 |
| Telephony | 0.009 | 0.027 |
| **Subtotal** | **~0.10** | **~0.29** |

Add orchestration platform fee if used. Your market price must clear this plus margin.

## 7. Alerts & budgets

- [ ] Per-tenant daily $ cap  
- [ ] Anomaly: $/min spike (bad TTS loop, stuck call)  
- [ ] Concurrency hard limit  
- [ ] Outbound dialer spend separate from inbound  
- [ ] Weekly COGS review vs revenue  

## 8. Checklist

- [ ] Written COGS model for v1 locale  
- [ ] Meters on every call  
- [ ] Waste (outbound) measured  
- [ ] Price card covers concurrency + minutes  
- [ ] Provider rate change review quarterly  

## Next

→ [09 — Production ops](09-production-ops.md)
