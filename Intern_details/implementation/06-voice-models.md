# 06 — Voice Models Guide (Engineer Selection)

> **Implements:** Overview §6 (except evals §6.9 and pricing §6.10 — see 07/08)  
> **Outcome:** You can choose STT/TTS/LLM/S2S, understand tokens/metering, runtime hardware, quantization, and open vs closed — without training models.

## 1. Model families you will actually integrate

| Family | I/O | Role in calling |
| --- | --- | --- |
| ASR / STT | Audio → text | Hear the caller |
| TTS | Text → audio | Speak to the caller |
| LLM | Text/tools → text/tools | Reason and act |
| Speech-realtime / S2S | Audio ↔ audio (+ text) | Combined conversational path |

Wispr Flow-class products are **dictation into apps**, not telephony agents. Do not use them as your call stack. (Contact-center “whisper” to an agent is also unrelated.)

## 2. STT options (practical)

| System | Open/closed | Fit |
| --- | --- | --- |
| Deepgram Nova family | Closed API | Default for low-latency phone streaming |
| AssemblyAI / Gladia / Google / Azure / AWS | Closed | Enterprise features, language coverage |
| OpenAI Whisper API | Closed API | Strong batch; check streaming needs |
| Whisper + faster-whisper / whisper.cpp | Open weights + runtime | Self-host batch or pseudo-streaming |

### Streaming vs batch (implementer’s rule)

| Need | Use |
| --- | --- |
| Live agent turns | Streaming STT with interim partials |
| Post-call CRM notes / compliance | Batch Whisper-class OK (can be higher accuracy) |
| Self-host near-realtime | Chunked Whisper (pseudo-streaming); validate latency |

**Do not** put naïve full-utterance batch Whisper on the hot path unless you measured RTF and UX.

### Telephony ASR checklist

- [ ] Test on **8 kHz / compressed** call recordings from your region  
- [ ] Measure entity error rate (names, phone numbers, order IDs), not only WER  
- [ ] Configure language / multilingual policy  
- [ ] Hotwords / keyterms for brand and SKU names  
- [ ] Punctuation + endpointing behavior documented  

## 3. TTS options (practical)

| System | Notes |
| --- | --- |
| ElevenLabs | High realism; often top COGS line |
| Cartesia | Strong low-latency streaming |
| Deepgram Aura / OpenAI TTS / cloud neural voices | Cost/simplicity trade-offs |
| Open TTS (XTTS, CosyVoice, Piper, …) | Self-host for cost/residency; bench quality + TTFB |

### TTS implementation requirements

- Streaming time-to-first-byte  
- Hard interrupt / cancel  
- Stable `voice_id` per agent  
- Pronunciation lexicon for brands  
- Cache audio for repeated IVR phrases  

## 4. LLM vs voice models

| | LLM | ASR | TTS |
| --- | --- | --- | --- |
| Job | Reason, plan, tool-call | Recognize speech | Synthesize speech |
| Metering | Text tokens | Usually audio minutes | Chars / credits / audio minutes |
| Eval | Task success, tool correctness | WER, entity errors | Preference, pronunciation, latency |

They are all neural nets; **only the LLM (or S2S brain) should decide side effects**.

## 5. Token concepts (do not mix them)

| Layer | Unit | What you configure |
| --- | --- | --- |
| LLM | Subword tokens | Context window, $/1K tokens, truncation policy |
| ASR | Audio seconds/minutes (API); frames internally | Streaming session billing |
| TTS | Characters, credits, or minutes | Voice tier; speaking rate impacts chars/min |
| Realtime S2S | Vendor “audio tokens” or session minutes | Read vendor docs; not interchangeable with GPT tokens |

**Capacity planning:** convert everything to **$ / answered minute** and, if self-hosting, **GPU-seconds / minute**.

### LLM context hygiene for calls

- Keep system prompt short  
- Store slots in structured state, don’t re-narrate endlessly  
- Truncate transcript to last N turns  
- Tool JSON can explode tokens — keep schemas tight  

## 6. Where models run (GPU vs CPU)

| Workload | Typical | Notes |
| --- | --- | --- |
| Vendor APIs | Their GPUs | Best ops default |
| Self-host large ASR/TTS/LLM | GPU | Needed for concurrency + RTF ≪ 1 |
| Tiny ASR/TTS, VAD, DSP | CPU | Fine |
| Quantized Whisper small/medium | CPU possible | Low concurrency only |

**Realtime factor (RTF)** = process_time / audio_duration. Live calls need RTF ≪ 1 with headroom for N concurrent sessions per machine.

## 7. Quantization (self-host only)

| Precision | Use |
| --- | --- |
| FP16 / BF16 | Default GPU serving |
| INT8 / INT8_FP16 | Common Whisper (faster-whisper) / LLM serving |
| 4-bit / GGUF | Edge LLMs; validate quality hard |

Always re-eval WER / task success on **telephony audio** after quantizing. Lab benchmarks lie.

### Runtime cheat sheet (Whisper-class)

| Hardware | Common choice |
| --- | --- |
| NVIDIA GPU | faster-whisper, FP16 or INT8 |
| Apple Silicon | whisper.cpp (Metal / Core ML) |
| CPU edge | whisper.cpp INT8 or faster-whisper INT8 |

## 8. Open vs closed decision matrix

| Requirement | Lean closed API | Lean open/self-host |
| --- | --- | --- |
| Time to MVP | ✓ | |
| Best streaming DX | ✓ | |
| Data residency / air-gap | | ✓ |
| Very high volume COGS | maybe hybrid | ✓ at scale |
| Small team SRE | ✓ | |
| Custom fine-tune later | API fine-tune or open | ✓ |

**Common hybrid:** closed streaming STT + closed TTS + chosen LLM live; open Whisper batch post-call; revisit TTS self-host when invoices hurt.

## 9. Speech-realtime / S2S

Use when:

- You want fewer moving parts  
- Vendor interrupt + tool calling meets your needs  
- You accept harder text-level control  

Still keep: telephony port, state machine, tool allowlists, transcript persistence, fallback cascaded path if SLA demands.

## 10. Selection bakeoff (run this)

For a fixed 20–50 call sample set:

| Scorecard | Weight |
| --- | --- |
| Task success % | high |
| Critical entity accuracy | high |
| p50 / p95 turn latency | high |
| $/answered minute | medium |
| Language/accent fit | high |
| Ops complexity | medium |

Publish a one-pager decision; store model IDs in call metadata forever.

## 11. Implementation checklist

- [ ] STT/TTS/LLM ports implemented with cancel  
- [ ] Bakeoff recorded for v1 locale  
- [ ] Post-call transcription path decided (can differ from live)  
- [ ] Pronunciation glossary process owned  
- [ ] Fallback provider configured for STT or TTS  
- [ ] If self-hosting: RTF + concurrency load test done  

## Next

→ [07 — Eval playbook](07-evals.md)  
→ [08 — Cost & pricing](08-cost-and-pricing.md)
