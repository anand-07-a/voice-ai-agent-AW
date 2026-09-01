# 09 — Production Ops, Compliance & Security

> **Implements:** Overview §7  
> **Outcome:** You can launch and operate voice agents with compliance, reliability, observability, and security baselines.

## 1. Pre-production gate

Do not open public traffic until:

- [ ] Disclosure + recording policy implemented for launch locales  
- [ ] Outbound consent / DNC / quiet hours enforced (if outbound)  
- [ ] PII redaction in logs/transcripts path defined  
- [ ] Human escalation tested  
- [ ] Eval CI green on golden set ([07](07-evals.md))  
- [ ] Latency dashboards + error alerts live  
- [ ] Budget / concurrency caps on  
- [ ] Incident runbook written (STT/TTS/LLM/telco outage)  

## 2. Compliance (engineering checklist — not legal advice)

| Topic | Implement |
| --- | --- |
| Recording disclosure | Speak/play required notice; set `disclosed_recording=true` before continuing |
| AI identity | State agent is automated where required/desired |
| Outbound consent | Store consent reference on dial jobs; block if missing |
| Quiet hours | Timezone-aware dial windows |
| DNC / frequency caps | Suppress list + max attempts |
| PCI | DTMF redirection / hosted fields; never log PAN |
| PHI / sensitive | Minimize transcript retention; encrypt; access ACLs |
| Retention | TTL jobs for recordings/transcripts; tenant-configurable |
| Number reputation | Monitor blocks; register where jurisdiction requires |

US STIR/SHAKEN, India commercial calling rules, EU GDPR, etc. differ — maintain a **per-country launch checklist** with counsel.

## 3. Reliability & scale

| Concern | Pattern |
| --- | --- |
| Sticky call session | Pin stream to one orchestrator instance (or shared Redis state carefully) |
| Horizontal scale | Stateless control plane + sticky realtime workers |
| Provider failover | Secondary STT/TTS/LLM with feature flag |
| Back-pressure | Reject/queue new calls when concurrency maxed |
| Dialer safety | Idempotent jobs; global kill switch |
| Poison calls | Max duration; max tool failures; forced hangup |

### Capacity planning inputs

- Target concurrent answered calls  
- p95 CPU/GPU or API RPS per call  
- Dialer fanout vs answer rate  
- Region placement next to CPaaS + AI APIs  

## 4. Observability

### Per-call timeline (required)

```text
call_created → media_up → first_agent_audio
→ turn{n}: endpoint, stt_final, llm_ttft, tts_ttfb, audio_out
→ tool{name}: start, end, status
→ transfer | hangup → persisted
```

### Dashboards

- Call volume, answer rate, avg duration  
- Outcome distribution  
- Turn latency p50/p95  
- Tool error rate  
- Escalation rate  
- $/answered minute by tenant  
- Provider error budgets  

### Logs

- Structured JSON with `call_id`, `tenant_id`, `turn_id`  
- Redact OTP, PAN, government IDs  
- Separate debug audio dumps (opt-in, short TTL)  

### Alerts

- Spike in STT/TTS/LLM 5xx  
- Turn latency p95 breach  
- Zero answer rate on a DID  
- Cost anomaly  
- Dialer runaway  

## 5. Security

| Control | Detail |
| --- | --- |
| Webhook auth | Signature verification + timestamp skew checks |
| Stream URLs | Short-lived tokens; no permanent public WS |
| Secrets | KMS/secret manager; rotate provider keys |
| Tenant isolation | Strict `tenant_id` on every query; no cross-tenant tools |
| Least privilege | Tool credentials scoped per tenant/integration |
| Admin auth | SSO + audit log for prompt/number changes |
| Abuse | Rate limits; outbound anomaly detection |

## 6. Human-in-the-loop ops

| Mode | When |
| --- | --- |
| Warm transfer | User asks / policy escalate |
| Agent assist | Human stays primary; AI suggests (different product) |
| QA sampling | 1–5% calls reviewed against rubric |
| Takeover | Supervisor barges into live call (if CC supports) |

Train humans on the handoff payload fields so context isn’t lost.

## 7. Testing in production-like envs

- Load: N concurrent synthetic calls  
- Chaos: kill TTS mid-sentence; stall tool 5s; inject packet loss  
- Devices: mobile speakerphone, noisy cafe recording  
- Locales: date/currency/phone pronunciation  
- Failover drill: disable primary STT  

## 8. Continuous improvement loop

```text
Failed/low-CSAT calls
  → tag root cause (ASR / policy / tool / latency)
  → add golden scenario
  → glossary or prompt/tool fix
  → CI gate
  → staged rollout (flag %)
```

Own a weekly “voice quality” review like an SRE error-budget meeting.

## 9. Multi-tenant SaaS ops extras

- Per-tenant dialer and concurrency quotas  
- Prompt/voice change audit trail  
- Usage export for invoices  
- Data residency region pin  
- Soft-delete tenant data job  

## 10. Incident runbook (skeleton)

1. Declare severity; freeze outbound if needed  
2. Identify failing layer via provider status + error rates  
3. Flip failover flag or disable affected agent  
4. Communicate to tenants  
5. Preserve call_ids for refunds/recon  
6. Postmortem with golden-set additions  

## Next

→ [10 — Use-case implementation template](10-use-case-template.md)
