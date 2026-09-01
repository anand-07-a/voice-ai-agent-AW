# Voice AI Calling — Engineer Implementation Docs

Actionable docs for software engineers implementing AI voice calling **without training or fine-tuning foundation models**.

Source material: [../voice-ai-calling-overview.md](../voice-ai-calling-overview.md)

## Reading order

| # | Doc | Overview section | When you need it |
| --- | --- | --- | --- |
| 01 | [Goal & build path](01-goal-and-build-path.md) | §1 | Choosing DIY vs BYOK vs bundled; MVP scope |
| 02 | [Use cases & readiness](02-use-cases.md) | §2 | Picking / scoping a domain use case |
| 03 | [Telephony primer](03-telephony-primer.md) | §3 | Numbers, VoIP, routing, SIM myths, handoff |
| 04 | [Agent architecture](04-architecture.md) | §4 | Cascaded pipeline, latency, barge-in, state |
| 05 | [Integrations](05-integrations.md) | §5 | CPaaS, tools, webhooks, CRM contracts |
| 06 | [Voice models guide](06-voice-models.md) | §6 | STT/TTS/S2S selection, tokens, GPU, open/closed |
| 07 | [Eval playbook](07-evals.md) | §6.9 | Golden sets, metrics, CI gates |
| 08 | [Cost & pricing](08-cost-and-pricing.md) | §6.10 | COGS stack, metering, SaaS packaging |
| 09 | [Production ops](09-production-ops.md) | §7 | Compliance, scale, observability, security |
| 10 | [Use-case implementation template](10-use-case-template.md) | §8–9 | End-to-end checklist to ship any domain |

## How to use these docs

1. Skim **01** and pick a build path.
2. Lock a use case with **02** + fill **10**.
3. Implement telephony (**03**) and the realtime loop (**04**).
4. Wire providers and business tools (**05**, **06**).
5. Gate launch with **07**, budget with **08**, operate with **09**.

## Out of scope (all docs)

- Training / fine-tuning ASR, TTS, or LLMs
- Carrier interconnect engineering from scratch
- Legal advice (compliance checklists are engineering-oriented; involve counsel for launch regions)
