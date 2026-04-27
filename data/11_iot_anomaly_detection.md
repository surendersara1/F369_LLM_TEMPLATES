<!-- Template Version: 2.0 | F369 Wave 12 (composite) | Composes: DATA_KINESIS_STREAMS_FIREHOSE + DATA_MANAGED_FLINK + DATA_OPENSEARCH_SERVERLESS + LLMOPS_BEDROCK -->

# Template 11 — IoT / Real-Time Anomaly Detection (Kinesis · Managed Flink · OpenSearch · Bedrock for explainability · alerts)

## Purpose

Stand up a **production-grade IoT / real-time anomaly detection pipeline in 4-5 days** that: ingests millions of telemetry events/sec, runs windowed statistical anomaly detection in Apache Flink, surfaces anomalies in OpenSearch dashboards, sends actionable alerts to operators, and uses Bedrock to explain anomalies in natural language.

Use cases:
- Industrial IoT — sensor temperature/pressure/vibration anomalies
- Connected vehicles — engine telemetry + driving behavior
- Fraud detection — payment transaction anomalies
- Network security — packet flow anomalies
- App performance — latency/error spikes

Generates production-deployable CDK + Flink job + alert workflow.

---

## Role Definition

You are an expert AWS streaming + ML architect with deep expertise in:
- Kinesis Data Streams + IoT Core ingestion paths
- Apache Flink stateful processing — z-score, EWMA, isolation forest, CUSUM
- Flink windowing — tumbling, sliding, session
- OpenSearch Serverless TIMESERIES collections + anomaly visualization
- Bedrock InvokeModel for natural language explanations of anomalies
- EventBridge alert routing + auto-remediation
- IoT device certificate auth + MQTT topic policies

Generate complete, production-deployable code. No TODOs.

---

## Context and Inputs

```
PROJECT_NAME:                [REQUIRED]
AWS_REGION:                  [REQUIRED]
ENV:                         [REQUIRED — dev | stage | prod]

# --- TELEMETRY ---
SENSOR_TYPES:                [REQUIRED — comma-separated; e.g. temperature,pressure,vibration]
DEVICE_COUNT:                [REQUIRED — order of magnitude]
EVENTS_PER_DEVICE_PER_SEC:   [REQUIRED]
INGEST_PATH:                 [REQUIRED — iot_core_mqtt | api_gateway_https | direct_kinesis]

# --- ANOMALY DETECTION ---
DETECTION_ALGORITHMS:        [REQUIRED — z_score, ewma, isolation_forest, cusum (multiple OK)]
WINDOW_SIZE_SECONDS:         [60 default — tumbling window]
ANOMALY_THRESHOLD_STD:       [3 default — z-score threshold]
HISTORICAL_BASELINE_DAYS:    [7 default — for EWMA / isolation forest]

# --- ALERTING ---
ALERT_DESTINATIONS:          [REQUIRED — pagerduty, slack, sms, email (multiple OK)]
PAGERDUTY_INTEGRATION_KEY:   [REQUIRED if pagerduty]
SLACK_WEBHOOK_URL:           [REQUIRED if slack]
SMS_PHONE_NUMBERS:           [comma-separated +1234567890]
ENABLE_BEDROCK_EXPLAINER:    [true default — natural-language anomaly description]
BEDROCK_MODEL_ID:            [anthropic.claude-haiku-4-5-20251001 default for low-cost]

# --- AUTO-REMEDIATION ---
ENABLE_AUTO_REMEDIATION:     [false default — true to invoke Lambda on critical anomaly]
REMEDIATION_LAMBDA_ARN:      [optional]

# --- COMPLIANCE ---
KMS_KEY_ARN:                 [REQUIRED]
TELEMETRY_RETENTION_DAYS:    [90 default in OS]
RAW_RETENTION_YEARS:         [3 default in S3]
```

---

## Partial Library (Claude MUST load)

| Partial | Why |
|---|---|
| `DATA_KINESIS_STREAMS_FIREHOSE` | Telemetry ingestion + S3 raw archive |
| `DATA_MANAGED_FLINK` | Flink job for windowed anomaly detection (z-score, EWMA, etc.) |
| `DATA_OPENSEARCH_SERVERLESS` | TIMESERIES + anomaly index + dashboards |
| `LLMOPS_BEDROCK` | Bedrock InvokeModel for anomaly explanation |
| `LAYER_BACKEND_LAMBDA` | Alert Lambda + auto-remediation Lambda |
| `LAYER_OBSERVABILITY` | CW alarms on pipeline health |
| `EVENT_DRIVEN_PATTERNS` | EventBridge alert routing |

---

## Architecture

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ IoT Devices (sensors, vehicles, etc.)                            │
   │   - Cert-based MQTT auth                                          │
   │   - Publish to topic: telemetry/{device_id}/{sensor_type}        │
   └────────────────────────────────┬─────────────────────────────────┘
                                    │ MQTT (TLS)
                                    ▼
                   ┌─────────────────────────────────┐
                   │ AWS IoT Core                     │
                   │   - Device registry              │
                   │   - Per-device cert + policy     │
                   │   - IoT Rule: SELECT *, time...  │
                   └────────────────┬─────────────────┘
                                    │ Rule action: PUT to KDS
                                    ▼
                   ┌─────────────────────────────────┐
                   │ Kinesis Data Stream (raw)        │
                   │   - On-demand auto-scale          │
                   │   - 24h retention                  │
                   │   - KMS encrypted                  │
                   └─────────────┬─────────┬───────────┘
                                 │         │
                ┌────────────────┘         └────────────────┐
                ▼                                            ▼
   ┌────────────────────────┐               ┌──────────────────────────────┐
   │ Firehose → S3 archive  │               │ Managed Flink Application    │
   │ (Parquet + partition)  │               │ - 4 KPUs (auto-scale)         │
   │ Retention 3y           │               │ - Stateful per-device:        │
   └────────────────────────┘               │     · 60s tumbling window     │
                                            │     · Z-score per sensor      │
                                            │     · EWMA baseline           │
                                            │     · CUSUM change detect     │
                                            │ - Output stream: anomalies    │
                                            │     {device_id, sensor,       │
                                            │      value, expected, score,  │
                                            │      window_end, severity}    │
                                            └──────────────┬───────────────┘
                                                           │
                              ┌────────────────────────────┼─────────────────┐
                              ▼                            ▼                 ▼
                  ┌──────────────────────┐    ┌──────────────────┐   ┌──────────────┐
                  │ Firehose → OS Sink   │    │ EventBridge      │   │ Optional:    │
                  │ (anomalies index)    │    │  rule pattern    │   │ S3 archive   │
                  │ - ISM hot 30d → del  │    │  severity > 5    │   │ for ML       │
                  │ - Dashboard:         │    └────┬─────────────┘   │ retraining   │
                  │   anomalies/min,     │         │                  └──────────────┘
                  │   top devices,       │         ▼
                  │   geo heatmap        │    ┌──────────────────┐
                  └──────────────────────┘    │ Alert Lambda     │
                                              │ 1. Format alert   │
                                              │ 2. Bedrock        │
                                              │   InvokeModel     │
                                              │   → "Temperature  │
                                              │   spike in Tower  │
                                              │   42 likely from  │
                                              │   coolant fail"   │
                                              │ 3. Fan-out:       │
                                              │   - PagerDuty     │
                                              │   - Slack         │
                                              │   - SMS (SNS)     │
                                              │ 4. (opt) trigger  │
                                              │   remediation Lam.│
                                              └──────────────────┘
```

---

## Day-by-day execution (4-5 day POC, 1 dev)

### Day 1 — IoT Core + Kinesis ingest
- KMS CMK
- IoT Core thing type + 5 sample devices + per-device cert + policy
- IoT Rule: `SELECT *, timestamp() AS receipt_time FROM 'telemetry/+/+' → PUT to KDS`
- Kinesis Data Stream (on-demand, KMS) + S3 raw archive via Firehose #1 (Parquet)
- Sample device simulator script (Python with `awsiotsdk`) generating telemetry
- **Deliverable:** End of Day 1: simulator publishes → IoT Console shows messages → KDS receives → S3 Parquet within 90s.

### Day 2 — Flink anomaly detection job
- Managed Flink Application + IAM role + S3 checkpoint bucket
- Flink SQL job (or Java DataStream) implementing chosen `DETECTION_ALGORITHMS`:
  - Z-score: `(value - rolling_mean) / rolling_std > THRESHOLD`
  - EWMA: exponentially weighted moving average → deviation from baseline
  - CUSUM: cumulative sum of deviations to detect regime change
- Source: KDS raw telemetry; Sink: KDS anomalies stream
- 60s tumbling windows per (device_id, sensor_type)
- Watermark for event-time processing
- Checkpoints every 60s, snapshots every 24h
- **Deliverable:** Inject artificial anomaly (high temperature) → anomaly detected within 60-90s, written to anomalies stream.

### Day 3 — OpenSearch + Bedrock explainer + alert routing
- OpenSearch Serverless TIMESERIES collection for anomalies
- Firehose #2: anomalies stream → OS sink (`anomalies-{daily}` indices, ISM 30d)
- OS Dashboards: 5 panels — anomalies/min, top devices, severity distribution, geo heatmap, time-series with annotations
- EventBridge rule: anomaly events → Alert Lambda
- Alert Lambda implementation:
  - Format anomaly context (device, sensor, recent values, expected range)
  - Call Bedrock InvokeModel with prompt: "Explain this anomaly in 2 sentences for an operations engineer..."
  - Fan-out to `ALERT_DESTINATIONS` (PagerDuty Events API, Slack webhook, SNS for SMS)
- **Deliverable:** Critical anomaly → Slack message arrives within 30s with NL explanation from Claude Haiku.

### Day 4 — Auto-remediation + alarms + tests
- (If `ENABLE_AUTO_REMEDIATION`) Auto-remediation Lambda invoked from EventBridge for critical anomalies
- Example remediation: send command back to device via IoT Job (e.g., "reduce throttle"), or quarantine device by revoking cert
- 6 CloudWatch alarms:
  - KDS `IncomingRecords` drops 50%
  - Flink `numRestarts` > 0 in last 1h
  - Flink `lastCheckpointDuration` > 30s
  - OS storage > 80% OCU
  - Alert Lambda errors > 1%
  - PagerDuty webhook 4xx/5xx
- Pytest suite: smoke, anomaly injection, Bedrock explainer, alert delivery
- **Deliverable:** Pass 6 failure-mode tests; runbook for IoT operator.

### Day 5 — Tuning + load test + handoff
- Load test: simulator pushes `DEVICE_COUNT × EVENTS_PER_DEVICE_PER_SEC` for 30 min
- Tune Flink parallelism, OS OCU, alert thresholds
- Document operations: how to add new device, change threshold, retrain baseline
- **Deliverable:** Customer can self-serve adding new sensor types; load test sustains target throughput.

---

## Validation criteria

- [ ] **Device simulator publishes** telemetry; IoT Console shows messages in monitor
- [ ] **KDS receives at target throughput** (`IncomingBytes` matches simulator load)
- [ ] **Flink app RUNNING**, checkpoints succeeding every 60s
- [ ] **Injected anomaly detected within 90s** of producer event time
- [ ] **OS Dashboards show anomalies** (test by injection)
- [ ] **EventBridge rule fires** for critical anomalies (verified via CW metric `Invocations`)
- [ ] **Alert Lambda completes < 5s** including Bedrock InvokeModel
- [ ] **Bedrock explanation makes sense** — manual review of 5 sample explanations
- [ ] **PagerDuty/Slack/SMS receive alert** within 30s
- [ ] **Auto-remediation Lambda executes** (if enabled) — verify via CW metric
- [ ] **All 6 alarms in OK state** baseline; firing on injected failures
- [ ] **Load test sustains target** — no KDS throttling, Flink no restarts, OS no rejections

---

## Common gotchas (claude must address proactively)

- **IoT Rule SQL has limitations** — no joins, no Lambda invoke (use Lambda action separately). Use `SELECT *, timestamp() AS receipt_time FROM ...` to avoid losing device-level metadata.
- **Per-device cert auth scales to ~1M devices** — beyond that, use IoT Core for LoRaWAN OR custom Lambda authorizer.
- **Flink stateful per-device requires careful state TTL** — without TTL, state grows unbounded. Add `state.ttl: 7 days` in checkpoint config.
- **Z-score requires sufficient data points** — first 60s window has no baseline. Bootstrap from historical S3 data OR add warmup period.
- **EWMA decay factor** controls sensitivity — α=0.1 = slow, α=0.9 = fast. Start α=0.3 for most sensors.
- **Bedrock InvokeModel from Lambda** can have 1-3s latency. Use `claude-haiku` (faster + cheaper) for alert pipelines, not `claude-opus`.
- **PagerDuty rate limit** = 120 events/minute per integration. For high-anomaly volume, batch via deduplication key.
- **SNS SMS in many regions has spend limit** ($1/mo default) — set explicit `SetSMSAttributes` for `MonthlySpendLimit`.
- **Auto-remediation must be reversible / idempotent.** Test thoroughly in stage; many "smart" remediations make incidents worse.
- **OS index per day works for moderate volume** — for > 1B docs/day, partition further (per-day-per-device-prefix).
- **IoT Core fleet provisioning** for 1000+ devices — use bulk provisioning with provisioning template + claim cert.

---

## Output artifacts

1. **CDK stacks** — `IotIngestStack`, `KinesisStack`, `FlinkStack`, `OssStack`, `AlertStack`
2. **Flink job** — Java DataStream OR SQL (`anomaly_detector.sql`)
3. **Device simulator** — Python script using `awsiotsdk` for load tests
4. **Alert Lambda** — formatting + Bedrock + fan-out (PagerDuty + Slack + SNS)
5. **Auto-remediation Lambda** template (parameterized for device action)
6. **OS Dashboards JSON** — 5 anomaly panels
7. **CloudWatch dashboard** — pipeline health
8. **6 alarms YAML** — wired to SNS
9. **Bedrock prompt template** — anomaly explanation system prompt + few-shot examples
10. **Pytest suite** — covers all validation criteria
11. **Operator runbook** — adding device, changing thresholds, investigating anomaly
12. **Load test script** — produces `DEVICE_COUNT × EVENTS_PER_DEVICE_PER_SEC` for 30 min

---

## Changelog

| Version | Date | Change |
|---|---|---|
| 2.0 | 2026-04-26 | Initial. IoT Core + KDS + Flink anomaly detection + OS Dashboards + Bedrock NL explainer + alert fan-out. Wave 12. |
