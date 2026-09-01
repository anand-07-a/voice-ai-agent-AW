# 07 — Eval Playbook

> **Implements:** Overview §6.9  
> **Outcome:** You can build golden sets, metrics, CI gates, and online monitors for a specific use case.

## 1. Principle

Evaluate the **phone task**, not generic chatbot quality.

Layer evals:

1. Components (ASR, TTS, LLM policy)  
2. Task outcomes (did the business action succeed?)  
3. Conversation quality (human/LLM rubrics)  
4. Latency & reliability SLOs  
5. Online production monitors  

## 2. Golden set (minimum viable)

Start with **50–200** scenarios per use case / locale.

Per scenario store:

| Field | Example |
| --- | --- |
| `id` | `book_happy_001` |
| `locale` | `en-IN` |
| `audio` or `user_script` | wav **or** simulated user turns |
| `expected_slots` | `{ "date": "2026-08-04", "doctor_id": "D12" }` |
| `expected_tools` | `["lookupPatient","listSlots","bookSlot"]` |
| `expected_outcome` | `booked` |
| `must_say` | disclosure phrases |
| `must_not_say` | medical diagnosis claims |
| `tags` | `noisy`, `accent_hi`, `barge_in` |

### Scenario categories to include

- Happy path  
- Missing info / multi-turn slot fill  
- ASR-hard entities (names, IDs)  
- User barge-in  
- Wrong person / wrong number  
- Angry user → escalate  
- Tool failures (`SLOT_TAKEN`)  
- Long silence / no-input  
- Language switch (if in scope)  
- Adversarial prompt injection via speech (“ignore instructions…”)  

## 3. Metrics

### Component

| Metric | How |
| --- | --- |
| WER / CER | Against reference transcript |
| Entity error rate | Exact match on normalized IDs, dates, amounts, phones |
| TTS preference | Side-by-side human or calibrated judge on sample |
| Pronunciation hits | Glossary terms spoken correctly |

### Task (primary)

| Metric | Definition |
| --- | --- |
| Task success | Final outcome == expected_outcome |
| Slot accuracy | Correct extracted/confirmed slots |
| Tool correctness | Right tools, args, no extra side effects |
| Disclosure compliance | Required phrases present |
| Unsafe action rate | Side effect without verification / confirmation |

### Experience / systems

| Metric | Target ideas (set your own) |
| --- | --- |
| Median turn latency | ≤ 1.0–1.2 s |
| p95 turn latency | ≤ 2.0–2.5 s |
| Barge-in success | Agent stops ≤ 300–500 ms |
| Crash / timeout rate | ~0 on golden set |
| Dead-end loops | Max-turn escalate instead of spinning |

## 4. Harness architecture

```text
Test runner
  → loads scenario
  → runs agent in "sim" mode (text user) and/or audio replay
  → captures transcript, tools, outcome, latency
  → scores vs expected
  → emits JUnit/JSON report + failure artifacts
```

### Two modes

| Mode | Pros | Cons |
| --- | --- | --- |
| **Text simulation** | Fast CI, stable | Misses ASR/TTS issues |
| **Audio replay** | Catches ASR | Needs recordings; flakier |
| **Full dial (staging)** | True telephony | Slow; use nightly / pre-release |

**CI default:** text sim on every prompt/tool change.  
**Nightly:** audio subset.  
**Pre-release:** N live staging calls.

## 5. LLM-as-judge (use carefully)

Allowed for: empathy, repetition, unnatural phrasing.  
Not allowed alone for: money/PHI actions, compliance disclosures, slot exactness.

Calibrate judges on a **human-labeled** set of ≥30 calls; track agreement. If agreement is poor, drop the judge metric.

## 6. CI gates (example)

Fail the build if any:

- Task success < threshold on critical suite  
- Any `must_not_say` hit  
- Any side-effect tool called without `verified=true` when required  
- Disclosure missing on scenarios tagged `requires_disclosure`  
- p95 latency regression > X% vs baseline  

## 7. Online evals

| Signal | Use |
| --- | --- |
| Containment / escalation rate | Product health |
| Outcome events from tools | Ground truth when available |
| CSAT / thumbs | Sparse but directional |
| “Sorry?” / repeat rate | Proxy for ASR/UX pain |
| Human QA sample (1–5%) | Calibration |
| Shadow / A/B prompts & voices | Controlled experiments |

Store `prompt_version`, model IDs, voice IDs on every call for attribution.

## 8. Domain rollout process

1. Write state machine + tools ([02](02-use-cases.md), [10](10-use-case-template.md))  
2. Author 50 golden scenarios (text)  
3. Reach task success threshold in sim  
4. Add 20 real audio calls; fix ASR glossary  
5. Staging dial tests  
6. Limited production with heavy QA  
7. Expand concurrency / traffic  

## 9. Checklist

- [ ] Golden set exists per locale  
- [ ] Task success is the north-star metric  
- [ ] Entity normalization rules documented (dates, phones)  
- [ ] CI runs on prompt changes  
- [ ] Production dashboards for latency + outcomes  
- [ ] Owner for weekly failure mining → new golden cases  

## Next

→ [08 — Cost & pricing](08-cost-and-pricing.md)
