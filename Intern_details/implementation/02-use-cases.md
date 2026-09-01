# 02 — Use Cases & Readiness

> **Implements:** Overview §2  
> **Outcome:** You can pick a voice-AI-ready use case, define success metrics, and know inbound vs outbound implications.

## 1. Use-case families (pick one primary)

| Family | Direction | Example outcomes | Primary metrics |
| --- | --- | --- | --- |
| Sales & growth | Out / in | Qualified lead, booked demo | Connect rate, booking rate |
| Collections & reminders | Out | Promise-to-pay, confirm/cancel | PTP rate, confirm rate |
| Support / IVR replace | In | Status resolved without human | Containment %, AHT, CSAT |
| Operations | Both | Dispatch assigned, ETA given | FCR, SLA hit rate |
| Healthcare / regulated | Both | Appointment booked (compliant) | Slot accuracy, audit completeness |
| Surveys | Out | Survey completed | Completion %, data quality |
| Internal tools | Both | On-call ack | Time-to-ack, error rate |

**Rule:** v1 should optimize for **one family + one outcome**. Multi-domain agents fail evals and prompts first.

## 2. Inbound vs outbound engineering matrix

| Concern | Inbound | Outbound |
| --- | --- | --- |
| Number setup | DID → webhook/SIP app | Caller ID / branded calling reputation |
| Answer path | Always-on answer URL | Dial API + AMD / voicemail branch |
| Consent | Recording disclosure on answer | Prior consent, DNC, quiet hours, frequency caps |
| Traffic shape | Spiky by business hours | Controlled by dialer concurrency |
| Failure UX | Escalate / callback offer | Retry policy, leave message, suppress |
| Identity | ANI (caller ID) as hint only | You chose the number; still verify for sensitive acts |

### Hybrid patterns

- Missed human call → AI callback  
- AI qualifies → warm transfer to sales/support  
- Inbound AI → schedule outbound reminder  

Implement hybrid only after one direction is stable.

## 3. Voice-AI readiness scorecard

Score each 0–2 (0 = no, 1 = partial, 2 = yes). Proceed if total ≥ 10/16 and no hard blockers.

| Criterion | Score | Evidence |
| --- | --- | --- |
| Finite outcomes (≤ ~5 terminal states) | | |
| Structured slots to collect (IDs, dates, yes/no) | | |
| System of record API exists (read/write) | | |
| Human escalation path exists | | |
| Latency budget OK (user can wait ~1s turns) | | |
| Sample real call audio/transcripts available | | |
| Language/locale clear for v1 | | |
| Compliance owner identified for region | | |
| **Total** | /16 | |

### Hard blockers (do not start autonomous voice)

- Licensed human required on every call and AI cannot be clearly assistive  
- No API / no human process to complete the task after the call  
- Zero eval data and high dialect/noise with safety-critical outcomes  

## 4. Specify the use case (fill this)

Copy into your ticket / RFC:

```markdown
## Use case name
...

## Direction
[ ] inbound  [ ] outbound  [ ] hybrid (describe)

## Locale / language
e.g. en-IN, hi-IN, en-US

## Actor
Who is calling / being called? What do they want?

## Terminal outcomes
1. success: ...
2. escalate: ...
3. fail_soft: ... (voicemail, no-answer, user decline)

## Slots to collect / confirm
| Slot | Required | Validation | Source of truth |
| --- | --- | --- | --- |

## Tools (APIs)
| Tool | Latency SLO | Side effects | Idempotency |
| --- | --- | --- | --- |

## Disclosures
Recording? AI identity? Marketing consent?

## Out of scope for v1
...

## Success metrics
- Primary:
- Guardrail (must not regress):
```

## 5. Conversation skeleton (before prompts)

Write a **state machine**, not a novel:

```text
GREETING → AUTH_OR_LOOKUP → INTENT → SLOT_FILL → CONFIRM → ACTION → CLOSE
                \-> ESCALATE ------------------------------------/
                \-> RETRY_LIMIT --------------------------------/
```

For each state define:

- What the agent may say  
- What tools it may call  
- Exit conditions  
- Max turns before escalate  

## 6. Example: clinic appointment booking (template fill)

| Field | Example |
| --- | --- |
| Direction | Inbound |
| Locale | en-IN |
| Success | Appointment created in EHR/calendar |
| Slots | patient phone, preferred date window, doctor/clinic, reason (optional) |
| Tools | `lookupPatient`, `listSlots`, `bookSlot` |
| Escalate | clinical symptoms / angry / 2 failed lookups |
| Metrics | booking success ≥ X%, date entity accuracy ≥ 99%, median turn latency ≤ 1.2s |

## 7. Exit criteria

- [ ] Primary family + outcome chosen  
- [ ] Readiness scorecard completed  
- [ ] Use-case RFC filled  
- [ ] State machine drafted  
- [ ] Metrics agreed with product/ops  

## Next

→ [03 — Telephony primer](03-telephony-primer.md)  
→ Parallel: start filling [10 — Use-case implementation template](10-use-case-template.md)
