# 10 — Use-Case Implementation Template

> **Implements:** Overview §8–9 (end-to-end path)  
> **Outcome:** A reusable checklist to implement any domain voice agent from scratch **without model training**.

Copy this file per use case: `use-cases/<name>.md`.

---

## A. Identity

| Field | Value |
| --- | --- |
| Use case name | |
| Owner (eng / product) | |
| Direction | inbound / outbound / hybrid |
| Locales | |
| Build path | A DIY / B BYOK / C bundled / D speech-native ([01](01-goal-and-build-path.md)) |
| Target launch date | |

## B. Problem & outcomes

**User problem (1–2 sentences):**

**Terminal outcomes:**

| Code | Meaning | Who closes it |
| --- | --- | --- |
| `success` | | agent |
| `escalate` | | transfer |
| `user_declined` | | agent |
| `no_answer` / `voicemail` | | dialer |
| `system_fail` | | platform |

**Primary metric:**  
**Guardrail metrics:**  

## C. Readiness

Complete scorecard in [02-use-cases.md](02-use-cases.md). Total: __ / 16  
Hard blockers: none / list:

## D. State machine

```text
(paste states and transitions)
```

| State | Allowed tools | Exit conditions | Max turns |
| --- | --- | --- | --- |
| GREETING | | | |
| … | | | |

## E. Slots

| Slot | Required | Validation | Confirm aloud? | Source of truth |
| --- | --- | --- | --- | --- |
| | | | | |

## F. Tools

| Name | Side effect? | Timeout | Idempotent? | Allowed states | Owner API |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

Auth / verification required before: _______________

## G. Telephony

| Item | Value |
| --- | --- |
| CPaaS / SIP | |
| Numbers (staging/prod) | |
| Recording | yes/no + disclosure text |
| Transfer targets | |
| DTMF uses | |
| Outbound consent store | n/a / link |
| AMD policy | n/a / voicemail drop / hangup |

Follow [03-telephony-primer.md](03-telephony-primer.md).

## H. Model stack

| Layer | Provider / model | Voice / lang | Fallback |
| --- | --- | --- | --- |
| STT | | | |
| LLM | | | |
| TTS | | | |
| Post-call ASR | | | |
| Orchestration | | | |

Bakeoff notes link: ________  
See [06-voice-models.md](06-voice-models.md).

## I. Prompt pack

- System persona (speech-optimized): link/version  
- Disclosure lines:  
- Escalation phrases:  
- Forbidden behaviors:  
- Sample happy-path dialogue (10–20 turns):  

## J. Architecture wiring

Confirm from [04-architecture.md](04-architecture.md):

- [ ] Session object fields defined  
- [ ] Barge-in + cancel works  
- [ ] Sentence-flush TTS on  
- [ ] Latency timestamps emitted  
- [ ] Max call duration set  

## K. Integrations

From [05-integrations.md](05-integrations.md):

- [ ] CRM/ticket write on escalate + hangup  
- [ ] Webhooks to consumer systems (if any)  
- [ ] Secrets in secret manager  
- [ ] Prompt/model versions stored on call  

## L. Evals

From [07-evals.md](07-evals.md):

| Item | Target | Status |
| --- | --- | --- |
| Golden set size | ≥ 50 | |
| Task success (sim) | ≥ __% | |
| Critical entity accuracy | ≥ __% | |
| Disclosure compliance | 100% on tagged | |
| CI gate | linked | |

## M. Cost

From [08-cost-and-pricing.md](08-cost-and-pricing.md):

| Item | Value |
| --- | --- |
| Expected answered min/month | |
| Estimated COGS $/min | |
| Price / packaging | |
| Concurrency cap | |
| Budget alert threshold | |

## N. Compliance & ops

From [09-production-ops.md](09-production-ops.md):

- [ ] Locale compliance checklist signed off  
- [ ] Retention TTL configured  
- [ ] Dashboards + alerts  
- [ ] Incident runbook link  
- [ ] QA sampling plan  

## O. Implementation sequence (suggested sprints)

| Sprint | Deliverable | Done when |
| --- | --- | --- |
| 0 | RFC + readiness + state machine | This template A–F filled |
| 1 | Telephony echo + hangup store | Staging DID answers; recording optional |
| 2 | Cascaded loop greeting | Agent greets + listens + replies once |
| 3 | Tools + slot fill | Happy path books/updates SoR |
| 4 | Barge-in + escalate | Interrupt + human transfer work |
| 5 | Evals CI + dashboards | Gates green; timeline visible |
| 6 | Limited prod | Cap concurrency; daily QA |
| 7 | Scale + cost tune | Caps, caching, model tiering |

## P. Launch checklist

- [ ] Staging sign-off (eng + product + ops)  
- [ ] Prod numbers + webhook URLs  
- [ ] Kill switches verified  
- [ ] Support playbook for escalations  
- [ ] Post-launch review date set (T+7)  

## Q. Post-launch learning

| Week | Top failures | Fixes | Golden cases added |
| --- | --- | --- | --- |
| 1 | | | |
| 2 | | | |

---

## Quick link-back

| Need | Doc |
| --- | --- |
| Build path | [01](01-goal-and-build-path.md) |
| Readiness | [02](02-use-cases.md) |
| Telephony | [03](03-telephony-primer.md) |
| Loop / barge-in | [04](04-architecture.md) |
| Tools / CRM | [05](05-integrations.md) |
| Models | [06](06-voice-models.md) |
| Evals | [07](07-evals.md) |
| Cost | [08](08-cost-and-pricing.md) |
| Prod ops | [09](09-production-ops.md) |
| Concepts overview | [../voice-ai-calling-overview.md](../voice-ai-calling-overview.md) |
