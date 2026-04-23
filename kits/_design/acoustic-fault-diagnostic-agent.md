<!-- Design Doc | Companion to: kits/acoustic-fault-diagnostic-agent.md | Last-updated: 2026-04-22 -->

# Design — Acoustic Fault Diagnostic Agent Kit

> **Purpose of this doc.** Captures the rationale, UX shape, architecture, and three business-flow examples that frame why the kit has the shape it does. The kit itself (`kits/acoustic-fault-diagnostic-agent.md`) is the executable playbook; this doc is the "why we built it this way" reference for partners, pitch decks, and internal training.

---

## 1. Why this kit — business case

**Timing is excellent.** AWS is discontinuing Lookout for Equipment on **October 7, 2026** (new customers already blocked since October 2025). Every existing customer needs a replacement path. AWS Monitron persists but is severely limited (vibration + temperature only, Android-only app, 2–7 day baseline). There is no first-party AWS alternative for audio-based anomaly detection + diagnosis. The kit lands in a real gap.

**Team already has the ML foundation.** The NBS audio team ran 4 approaches (GeCo Transformer, YAMNet/VGGish+LSTM, PANNs+vector search, Mobile-FaceNet on spectrograms) plus an external paper's 65-dim MFCC+PCP+STE → CNN+LSTM hitting 98% on 10 vehicle classes. What's missing is not the ML — it's the **production platform + agentic "engineering-expert" reasoning layer + dashboard**.

**One shape, many verticals.**
- Automotive — manufacturing QA, dealer diagnostic, fleet maintenance
- Industrial — boilers, conveyors, mining rigs, HVAC compressors (the original NBS IIoT / CloudRail play)
- Aerospace — engine test cells, bench testing
- Utility — substation transformer humming, turbine monitoring

Consultant picks `DOMAIN_PACK` at kickoff; the kit swaps classifier model + FMEA knowledge base + report template, not architecture.

**Sells in three wrappers:**
| Wrapper | Buyer | Pricing anchor |
|---|---|---|
| Automotive QA (end-of-line) | Plant engineering | ~$200k/yr per plant |
| Dealer service diagnostic | Dealer network IT | ~$40k/yr per network + per-use |
| Fleet preventive maintenance | Fleet operators | $2–5 per vehicle/month |
| Industrial anomaly detection (Lookout replacement) | Plant ops / IIoT leads | $80–300k/yr per site |

---

## 2. Frontend UX — what the end user sees

**Single dashboard. Engineer uploads audio, sees diagnosis with cited evidence.** No multi-chat — this is an expert system, not a conversational assistant. (Unlike the Deep Research kit where the user drives the plan, here the agent auto-runs once audio lands.)

```
┌─ Dashboard (React + CloudFront + WebSocket streaming) ─────────────────┐
│                                                                        │
│   [Upload audio] [Record live]       Machine: Toyota Corolla 2024 ▼    │
│                                                                        │
│   File: engine_idle_3.wav (4.2 MB · 15.3 s)                            │
│                                                                        │
│   ┌─ Waveform + spectrogram preview ────────────────────────────────┐  │
│   │  ████▓▒▒░░  (interactive scrubber)                               │  │
│   │  mel-spec color-mapped, anomaly regions highlighted red          │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│   ┌─ Diagnosis ──────────────────────────────────────────────────────┐  │
│   │  🔴 Anomaly detected — confidence 0.87                           │  │
│   │  Top probable cause: Valve clearance drift                        │  │
│   │  Severity 8 · Occurrence 6 · Detection 4 → RPN 192 (escalate)    │  │
│   │                                                                   │  │
│   │  Supporting evidence:                                             │  │
│   │   • Transient tick at 0.3–0.4s aligns with valve-tap signature   │  │
│   │     [DSP analysis: wavelet CWT showed energy at 2–4 kHz band]    │  │
│   │   • Similar pattern seen in 7 past cases — 6 confirmed as valve  │  │
│   │     clearance [see similar cases]                                │  │
│   │   • TSB T-SB-0234-22 applies to this VIN range [view TSB]        │  │
│   │                                                                   │  │
│   │  Recommended next steps:                                          │  │
│   │   1. Check valve clearances per TSB T-SB-0234-22                  │  │
│   │   2. If confirmed, adjust to spec 0.25mm intake / 0.28mm exhaust  │  │
│   │   3. Re-record engine idle after adjustment                       │  │
│   │                                                                   │  │
│   │  [Accept diagnosis]  [Reject]  [Escalate to senior tech]          │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│   ┌─ Similar past cases (click to play audio) ──────────────────────┐  │
│   │  1. 2024-03-15 Corolla LE — valve clearance confirmed           │  │
│   │  2. 2024-02-08 Corolla XSE — valve clearance confirmed          │  │
│   │  3. 2024-01-22 Camry — valve clearance confirmed                │  │
│   │  ... 4 more                                                      │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│   ┌─ Trend (for fleet-mode only) ────────────────────────────────────┐  │
│   │  This VIN's anomaly score over last 8 weeks: trending up ↗      │  │
│   │  Predicted time to failure: 18–32 days                           │  │
│   └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

UX design principles baked into the defaults:
- **Auto-run, not chat.** Engineer uploads or records, agent runs immediately. Saves a click + avoids "what should I ask?"
- **Spectrogram + waveform with anomaly overlay.** Visual feedback — engineers trust what they can see/hear, not just a label.
- **Citations everywhere.** Every claim links to TSB / similar case / DSP analysis that produced it. Matches Deep Research kit's grounding discipline.
- **Accept / Reject / Escalate buttons.** Feeds retraining loop — every diagnosis becomes labeled data.
- **RPN displayed.** Engineers trained in FMEA know the Severity × Occurrence × Detection math; speaks their language.
- **Per-machine view + per-VIN history.** Fleet mode unlocks trend + predicted-time-to-failure.

---

## 3. Architecture under the dashboard

```
Engineer uploads audio OR CloudRail edge box pushes (IoT Core → S3)
         │
         ▼
   S3 raw-audio bucket (KMS-encrypted, per-tenant prefix)
         │
         │ S3 → EventBridge → IngestionLambda
         ▼
   Audio preprocessing (MLOPS_AUDIO_PIPELINE — NEW):
       ├── librosa: resample 44.1 kHz, trim silence, segment 5s / 50% overlap
       ├── Extract features per DOMAIN_PACK:
       │     • mel-spectrogram (AST input)
       │     • MFCC + PCP + STE (hybrid CNN+LSTM path)
       │     • CWT wavelet (transient faults)
       ├── Augmentation (SpecAugment, noise superimpose, time-stretch, pitch-shift)
       └── Store tensors to S3 curated zone + DDB audio_metadata row
         │
         ▼
   Classifier inference (SageMaker endpoint per CLASSIFIER_MODEL):
       ├── AST (Audio Spectrogram Transformer) — fine-tuned on domain
       ├── Wav2Vec2 — alternative for speech-adjacent signals
       ├── Hybrid CNN+LSTM — cheap option, paper's 65-dim feature
       └── GeCo Transformer — unsupervised anomaly scoring
         │
         ▼
   Audio embedding written to S3 Vectors (PATTERN_AUDIO_SIMILARITY_SEARCH — NEW):
       • Strategy A: one vector per 5s window
       • Strategy B: aggregated sample-level vector
       • Strategy C: hybrid both
       • Non-filterable metadata: source_audio_s3_path
       • Filterable: machine_id, timestamp, fault_label, confidence, outcome
         │
         ▼
   Agent (Strands supervisor on AgentCore Runtime) orchestrates:
       ├── ClassifierTool      → SageMaker inference {class_probs, anomaly_score}
       ├── SimilarCaseTool     → S3 Vectors audio similarity (past diagnosed cases)
       ├── ManualKnowledgeTool → S3 Vectors RAG over TSBs / FMEA / repair manuals
       ├── DSPAnalystTool      → AgentCore Code Interpreter runs librosa/scipy/pywt
       │                          live analysis ("what's at 0.3s? CWT shows 2-4kHz")
       ├── FMEAScoreTool       → compute Severity × Occurrence × Detection = RPN
       ├── Synthesizer         → writes diagnosis card
       └── Reviewer            → grounding validator: every TSB citation actually matches
         │
         ▼
   Dashboard (React + CloudFront + OAC + API GW WebSocket):
       ├── Upload form + live waveform + spectrogram render
       ├── Live progress events (🔍 ClassifierTool running AST on 3 segments)
       ├── Diagnosis card with RPN + escalation button
       ├── Similar-cases list with audio playback
       ├── TSB citations with source links
       ├── Per-machine history + trend (fleet mode)
       └── Accept/Reject/Escalate buttons → feedback loop

   Governance:
       ├── FMEA RPN ≥ threshold → human escalation (Step Functions waitForTaskToken)
       ├── Guardrail (AgentCore Agent Control) blocks unsafe recommendations
       ├── Clarify bias monitor on classifier (e.g. fairness across vehicle model years)
       ├── CloudTrail → Object-Lock audit bucket (every diagnosis immutable)
       └── Lookout for Equipment → replaced (gone Oct 7, 2026)
```

---

## 4. How the tech stack gets exercised

| AgentCore / Strands / ML feature | Role in this kit |
|---|---|
| **AgentCore Runtime** | Supervisor + specialist sub-agents as microVM containers |
| **AgentCore Gateway** | MCP endpoint for tools (classifier, similarity search, manual RAG, DSP) |
| **AgentCore Identity** | OAuth2 outbound to dealer TSB portals / manufacturer secure databases |
| **AgentCore Memory** | User preferences, machine history, recurring patterns, technician expertise |
| **AgentCore Observability** | Per-diagnosis cost, classifier drift, label agreement with technicians |
| **AgentCore Code Interpreter** | **Central tool** — DSP analyst agent runs librosa / scipy / pywavelets on demand. Engineer can ask "show me the CWT at 0.3s" and get a live analysis chart back |
| **AgentCore Browser Tool** | Optional — fetch public TSBs / recall bulletins / manufacturer updates |
| **AgentCore Agent Control** (Cedar + Guardrail) | Block unsafe recommendations ("just ignore it"); enforce "never diagnose without citation"; redact customer PII |
| **AgentCore A2A** | Call external specialist agents (e.g. a specialized transmission-only expert agent) |
| **Strands multi-agent** | Supervisor + 5 specialist sub-agents fanned out per diagnosis |
| **Strands `@tool`** | Every tool is a @tool-decorated function with clear Args/Returns |
| **Strands session manager** | Per-machine history persists across diagnoses |
| **Strands eval** (grounding validator) | Every TSB citation is checked against the source before emission |
| **Strands hooks** | RBAC middleware (per-tenant data isolation), circuit breaker on classifier endpoint |
| **SageMaker Training** | Fine-tune AST / Wav2Vec2 / hybrid CNN+LSTM on domain audio |
| **SageMaker Serving** | Async Inference for non-urgent; sync for dealer-counter UX |
| **SageMaker Batch Transform** | Nightly fleet sweep — score thousands of vehicles |
| **SageMaker Clarify** | Bias monitor — classifier performance across vehicle-model-years / regions |
| **S3 Vectors** | Two indexes: audio-similarity (768-dim) + repair-manual RAG (1024-dim Titan v2) |
| **Audio preprocessing** (NEW MLOPS_AUDIO_PIPELINE) | librosa + pywavelets + scipy — mel-spec / MFCC / PCP / STE / CWT / augmentation |
| **Audio similarity search** (NEW PATTERN_AUDIO_SIMILARITY_SEARCH) | Per-window vs sample-level vector storage + temporal aggregation |

---

## 5. Three business-flow examples — same kit, three verticals

### 5.1 Toyota manufacturing QA (primary use case)

**Client persona:** Toyota plant engineering — end-of-line engine testing for every vehicle before shipment.

**Flow:**
1. Technician runs engine idle + rev-sweep protocol on finished vehicle.
2. Mic records audio → uploaded to S3 via plant network.
3. Preprocess → classifier → agent runs automatically (< 15 seconds).
4. Dashboard displays: "✅ Normal" OR "🔴 Anomaly: valve clearance drift, RPN 192, escalate".
5. On escalation, vehicle routed to diagnostic bay; technician confirms/disconfirms; outcome fed back to memory.

**Value:** catches engine defects before shipment that visual QA + current vibration tests miss. Expected miss-rate reduction: 30–50% based on literature.

**Sell:** ~$200k/yr per plant + SageMaker inference cost pass-through.

---

### 5.2 Dealer service diagnostic assistant

**Client persona:** Toyota dealer network — 800+ dealers, tier-1 service tech diagnosing customer complaints.

**Flow:**
1. Customer brings car in with "weird sound from engine".
2. Tech records engine at idle, 2000 RPM, and under load via phone app.
3. Audio → agent → dashboard shows top-3 probable causes ranked + recommended diagnostic tests ordered by priority.
4. Tech runs priority-1 test; if confirmed, part is ordered; if not, moves to priority-2.
5. Outcome (confirmed/disconfirmed) logged → retraining signal.

**Value:** junior techs (1–2 yrs experience) reach senior-tech diagnostic accuracy (5+ yrs). Reduces comeback rate (customer returns for same issue) by ~40%.

**Sell:** ~$40k/yr per dealer network + per-diagnosis fee.

---

### 5.3 Fleet preventive maintenance

**Client persona:** Enterprise fleet (taxi / delivery / rental) — 500–50,000 vehicles.

**Flow:**
1. Each driver records 30s engine idle sample at start-of-shift (app prompt).
2. Batch Transform nightly scores all fleet samples → anomaly score per VIN.
3. Agent computes trend (this VIN over last 8 weeks) → predicts time-to-failure.
4. Dashboard for fleet manager: top-20 at-risk vehicles this week, with auto-scheduled service.
5. Drivers get in-app notification: "Your vehicle scheduled for service Tuesday 9 AM".

**Value:** shifts fleet maintenance from reactive (breakdown → tow → fix) to predictive (service before failure). Industry data: 15–30% maintenance cost reduction + 10% uptime gain.

**Sell:** $2–5 per vehicle/month recurring.

---

### 5.4 Industrial variant — NBS IIoT legacy

Same architecture, different `DOMAIN_PACK`. For boilers, conveyors, mining rigs — the NBS PDF's original vision. Replaces Lookout for Equipment (retiring Oct 2026). CloudRail edge box pushes to S3; rest of pipeline identical.

**Sell:** $80–300k/yr per site.

---

## 6. Critical decisions baked into the kit

Each becomes a parameter the consultant sets at engagement kickoff:

- `DOMAIN_PACK` — `automotive_engine` | `industrial_machine` | `hvac_boiler` | `custom`
- `CLASSIFIER_MODEL` — `ast` (default) | `wav2vec2` | `hybrid_cnn_lstm` | `geco`
- `CLASSIFIER_TRAINING_MODE` — `fine_tune_from_pretrained` (default) | `train_from_scratch` | `lora_adapter`
- `INTAKE_MODE` — `web_upload` | `edge_box` (CloudRail / Greengrass) | `mobile_app`
- `PROCESSING_LATENCY` — `async` (default) | `sync` | `batch`
- `FEATURES` — subset of `[mel_spectrogram, mfcc, pcp, wavelet_cwt, short_term_energy]`
- `FMEA_RPN_ESCALATE` — default 150 (Severity×Occurrence×Detection)
- `AUGMENTATION_ENABLED` — true/false (SpecAugment + noise superimpose)
- `VECTOR_STRATEGY` — `per_window` | `aggregated_sample` | `hybrid_both`
- `AGENT_ARCHITECTURE` — same 3 options as prior kits (plain_bedrock_lambda / strands_lambda / strands_agentcore)

---

## 7. Partial gaps this kit surfaces

The existing 61 partials at v2.0 cover ~80% of this kit's surface. Two genuine new partials are needed:

| File | Severity | Why partials don't cover it today |
|---|---|---|
| **`MLOPS_AUDIO_PIPELINE.md`** | HIGH — central | Audio has unique DSP steps (resample, windowing, CWT wavelet, librosa-specific idioms, SpecAugment) that text / image / tabular pipelines don't touch. `PATTERN_DOC_INGESTION_RAG` was the text analog; this is the audio analog. |
| **`PATTERN_AUDIO_SIMILARITY_SEARCH.md`** | MED | Audio-specific storage strategy (per-window vs aggregated vs hybrid) + temporal-context idioms not covered by `DATA_S3_VECTORS` (which is modality-agnostic). Canonical query patterns (find-similar-sound / machine-history / training-data-mining) codified. |

**Plan:** both shipped alongside this kit in the same session (same pattern as HR kit's 3 partials, RAG kit's 2, Deep Research kit's 2).

Other existing partials carry most of the weight:
- `MLOPS_SAGEMAKER_TRAINING` / `MLOPS_SAGEMAKER_SERVING` / `MLOPS_BATCH_TRANSFORM` (classifier path)
- `MLOPS_CLARIFY_EXPLAINABILITY` (bias monitor on classifier)
- `DATA_S3_VECTORS` + `PATTERN_DOC_INGESTION_RAG` (manual RAG)
- All `AGENTCORE_*` (9 features) + all `STRANDS_*`
- `LAYER_*` (foundation) + `COMPLIANCE_HIPAA_PCIDSS` (optional)
- `AGENTCORE_CODE_INTERPRETER` — **key** — the DSP analyst agent lives here

---

## 8. What a consultant delivers at end of Week 2

- **CDK app** with ~12 stacks: Network, Security, DataLake (raw + curated buckets), AudioPipeline (NEW partial), Classifier (SageMaker training + serving), VectorStore (2 indexes), ManualIngestion, AgentRuntime (supervisor + 5 sub-agents), Api, Frontend, Observability, Compliance
- **React dashboard** with upload + live spectrogram + diagnosis card + similar cases + trend (fleet mode)
- **5–7 Strands agents** on AgentCore Runtime: supervisor + classifier + similarity + manual-knowledge + DSP analyst + synthesizer + reviewer
- **Trained classifier** (AST or GeCo) fine-tuned on client's labeled samples
- **Populated vector indexes** — past cases + TSB/manual RAG
- **FMEA database** — Severity × Occurrence × Detection × Current Controls × Action
- **Bedrock Guardrail** + Cedar policy per domain pack
- **Observability dashboards** — classifier drift, grounding score, RPN distribution, tech-override rate
- **Compliance stack** — CloudTrail + Object-Lock audit bucket + Config rules
- **Runbook** — 3 sample scenarios per domain pack, expected outputs, rollback steps
- **Pilot test** — 20 labeled audio files per machine class, confusion matrix, before/after accuracy

---

## 9. Open questions for the consulting lead to validate with client

1. **Primary DOMAIN_PACK**: automotive engine OR industrial machine? Drives sample data + classifier fine-tune corpus + FMEA template + TSB corpus.
2. **Client's existing labeled data**: how many labeled audio samples per fault class? If < 50/class, start with public MIMII / DCASE + targeted augmentation + active learning loop.
3. **Inference latency tolerance**: sync for dealer-counter UX (< 5s), async for manufacturing QA (< 30s), batch for fleet nightly — kit supports all three via `PROCESSING_LATENCY` param.
4. **Edge vs cloud-only**: does client need on-machine offline-capable inference (CloudRail edge box, Literal-Labs-style tiny model)? Adds complexity; v1 is cloud-only.
5. **FMEA ownership**: does client provide the FMEA spreadsheet, or is agent helping generate it from past failures? v1: client-provided; v2: agent-generated.
6. **TSB / manual corpus**: does client have digital TSBs/service bulletins/repair manuals? If paper-only, need scanning + OCR (Textract) as ingestion upstream.
7. **Human-in-loop threshold**: what RPN triggers mandatory escalation? (default 150)
8. **Fairness constraints**: does classifier need to perform equally across vehicle model years / driver demographics / geographic regions? Drives bias-monitor config.

---

## 10. References

- NBS IIoT white paper (internal): `E:\aws_services\aws_llm_Toyota_Audio_Analysis\0_Toyota_Car_Sounds\SAMPLE_DATA\NBS - IIoT - Accoustic.pdf`
- Toyota team experiments: `E:\aws_services\aws_llm_Toyota_Audio_Analysis\0_Toyota_Car_Sounds\EXPERIMEMNTS\` (4 team folders)
- Hybrid neural network paper: https://www.nature.com/articles/s41598-021-87399-1
- Acoustic Anomaly Detection via Transfer Learning: https://arxiv.org/pdf/2006.03429
- DCASE 2024 baseline (Mobile-FaceNet): https://arxiv.org/pdf/2403.00379
- Generative-Contrastive (GeCo): https://arxiv.org/pdf/2305.12111
- AST: https://huggingface.co/docs/transformers/en/model_doc/audio-spectrogram-transformer
- Wav2Vec2 on SageMaker: https://aws.amazon.com/blogs/machine-learning/fine-tune-and-deploy-a-wav2vec2-model-for-speech-recognition-with-hugging-face-and-amazon-sagemaker/
- AWS Lookout for Equipment (EOL 2026-10-07): https://aws.amazon.com/lookout-for-equipment/
- MIMII dataset: https://github.com/MIMII-hitachi/mimii_baseline/
- DCASE 2020 Task 2 dataset: https://github.com/AlexandrineRibeiro/DCASE-2020-Task-2

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-04-22 | Initial design doc. Grounded in NBS IIoT PDF + Toyota team experiments (GeCo, YAMNet, PANNs, FaceNet) + 2026 SOTA research (AST / Wav2Vec2 / LoRA fine-tuning / Literal Labs edge AI). Lookout for Equipment EOL (2026-10-07) positions this kit as a replacement. Three business flows: manufacturing QA / dealer service / fleet maintenance. Companion to `kits/acoustic-fault-diagnostic-agent.md`. Flags 2 new partials (`MLOPS_AUDIO_PIPELINE`, `PATTERN_AUDIO_SIMILARITY_SEARCH`). |
