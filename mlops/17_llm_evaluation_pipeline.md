<!-- Template Version: 1.0 | boto3: 1.35+ | rouge-score: 0.1.2+ | nltk: 3.9+ | bert-score: 0.3.13+ -->

# Template 17 — LLM Evaluation Pipeline

## Purpose
Generate a production-ready LLM evaluation pipeline: automated metrics computation (ROUGE, BLEU, BERTScore) against ground-truth datasets, Bedrock Model Evaluation jobs with built-in and custom metrics, human-in-the-loop evaluation via Step Functions with SNS notifications and API Gateway rating collection, A/B testing using SageMaker inference endpoints with production variants, evaluation report aggregation into a single dashboard-ready JSON output, and EventBridge-based model promotion gating when metrics fall below configurable thresholds.

---

## Role Definition

You are an expert AWS ML engineer specializing in LLM evaluation and benchmarking with expertise in:
- Automated NLP metrics: ROUGE (rouge-score library), BLEU (nltk), BERTScore (bert-score library)
- Amazon Bedrock Model Evaluation: `create_evaluation_job()` with automated and human evaluation configs, task types (Summarization, QuestionAndAnswer, Classification, TextGeneration), built-in and custom metrics
- Human-in-the-loop evaluation: Step Functions state machines for routing samples to reviewers, SNS notifications, API Gateway endpoints for collecting ratings
- A/B testing: SageMaker inference endpoints with production variants, traffic splitting, per-variant metric collection
- Evaluation report aggregation: combining automated metrics, human ratings, and A/B test results into unified JSON reports
- Model promotion gating: EventBridge rules that block SageMaker Model Registry promotion when evaluation thresholds are not met
- Statistical significance testing for model comparison decisions

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

EVAL_TYPE:              [REQUIRED - automated | human | ab_test | bedrock_eval | all]
                        automated: compute ROUGE/BLEU/BERTScore against ground-truth
                        human: Step Functions workflow routing samples to human reviewers
                        ab_test: SageMaker production variants with traffic splitting
                        bedrock_eval: Bedrock Model Evaluation job with built-in metrics
                        all: generate all evaluation pipelines

EVAL_DATASET_S3:        [REQUIRED - s3://bucket/path/to/eval_dataset.jsonl]
                        JSONL format: {"prompt": "...", "reference": "...", "category": "..."}

METRICS:                [OPTIONAL: ROUGE,BLEU,BERTScore]
                        Comma-separated list of automated metrics to compute.
                        Options: ROUGE, BLEU, BERTScore

MODEL_VARIANTS:         [OPTIONAL - JSON list of models to evaluate]
                        Example: [
                            {"name": "model_a", "model_id": "anthropic.claude-3-5-sonnet-20241022-v2:0"},
                            {"name": "model_b", "model_id": "anthropic.claude-3-5-haiku-20241022-v1:0"}
                        ]

THRESHOLD_CONFIG:       [OPTIONAL - JSON thresholds for promotion gating]
                        Example: {
                            "rouge_l_f1": 0.45,
                            "bleu": 0.30,
                            "bertscore_f1": 0.85,
                            "human_avg_rating": 3.5
                        }

BEDROCK_EVAL_TASK_TYPE: [OPTIONAL: QuestionAndAnswer]
                        Options: Summarization | QuestionAndAnswer | Classification | TextGeneration

BEDROCK_EVAL_METRICS:   [OPTIONAL: Builtin.Accuracy,Builtin.Robustness]
                        Comma-separated Bedrock built-in metric names

HUMAN_REVIEWER_EMAILS:  [OPTIONAL - comma-separated email addresses for SNS notifications]
HUMAN_SAMPLE_SIZE:      [OPTIONAL: 50 - number of samples to route to human reviewers]
HUMAN_RATING_SCALE:     [OPTIONAL: 1-5 - rating scale for human evaluation]

AB_TRAFFIC_SPLIT:       [OPTIONAL: 50/50 - traffic percentage split between variants]
AB_DURATION_HOURS:      [OPTIONAL: 24 - duration of A/B test in hours]
AB_ENDPOINT_INSTANCE:   [OPTIONAL: ml.g5.xlarge]

OUTPUT_S3_PATH:         [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/evaluations/]
```

---

## Task

Generate complete LLM evaluation pipeline:

```
{PROJECT_NAME}-llm-evaluation/
├── config.py                              # Central configuration
├── automated/
│   ├── compute_metrics.py                 # ROUGE, BLEU, BERTScore computation
│   ├── run_inference.py                   # Run model inference on eval dataset
│   └── metrics_utils.py                   # Metric helper functions and aggregation
├── bedrock_eval/
│   ├── create_eval_job.py                 # Bedrock create_evaluation_job()
│   ├── monitor_eval_job.py                # Poll job status + retrieve results
│   └── parse_eval_results.py              # Parse Bedrock evaluation output
├── human_eval/
│   ├── state_machine.py                   # Step Functions state machine definition
│   ├── send_notifications.py              # SNS notification for reviewers
│   ├── api_handler.py                     # API Gateway Lambda for rating collection
│   ├── aggregate_ratings.py               # Aggregate human ratings
│   └── create_human_eval_resources.py     # Create SNS topic, API GW, DynamoDB table
├── ab_testing/
│   ├── create_ab_endpoint.py              # SageMaker endpoint with production variants
│   ├── run_ab_test.py                     # Send traffic and collect per-variant metrics
│   ├── analyze_ab_results.py              # Statistical analysis of A/B results
│   └── cleanup_ab_endpoint.py             # Delete A/B test endpoint
├── reporting/
│   ├── aggregate_report.py                # Combine all eval results into single JSON
│   └── promotion_gate.py                  # EventBridge rule for threshold-based gating
├── run_evaluation.py                      # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Parse METRICS into a list. Parse THRESHOLD_CONFIG JSON. Validate EVAL_TYPE is one of the supported types.

**compute_metrics.py**: Compute automated NLP metrics against ground-truth:
- Load evaluation dataset from EVAL_DATASET_S3 (JSONL with `prompt`, `reference`, `category` fields)
- Load model predictions (generated by `run_inference.py` or provided externally)
- Compute ROUGE scores (rouge-1, rouge-2, rouge-L) using `rouge_score.rouge_scorer.RougeScorer`
- Compute BLEU score using `nltk.translate.bleu_score.corpus_bleu` with smoothing
- Compute BERTScore (precision, recall, F1) using `bert_score.score()`
- Aggregate per-sample metrics into summary statistics (mean, median, p5, p95)
- Output results as JSON to S3 with per-sample and aggregate sections

**run_inference.py**: Run model inference on the evaluation dataset:
- Load prompts from EVAL_DATASET_S3
- For each model in MODEL_VARIANTS, invoke `bedrock_runtime.converse()` with the prompt
- Collect responses with latency measurements
- Save predictions to S3 as JSONL: `{"prompt": "...", "prediction": "...", "model": "...", "latency_ms": ...}`
- Support batch processing with configurable concurrency

**metrics_utils.py**: Helper functions:
- `compute_rouge(predictions, references)`: Return dict of rouge-1/2/L F1 scores
- `compute_bleu(predictions, references)`: Return corpus BLEU score
- `compute_bertscore(predictions, references)`: Return mean precision/recall/F1
- `aggregate_metrics(per_sample_metrics)`: Compute mean, median, p5, p95 across samples

**create_eval_job.py**: Create Bedrock Model Evaluation job:
- Call `bedrock.create_evaluation_job()` with `jobName`, `roleArn`, `evaluationConfig` (automated with `datasetMetricConfigs`), `inferenceConfig` (model identifier with inference params), and `outputDataConfig` (S3 URI)
- Configure task type from BEDROCK_EVAL_TASK_TYPE
- Configure metric names from BEDROCK_EVAL_METRICS
- Support custom prompt datasets from EVAL_DATASET_S3
- Tag with Project and Environment

**monitor_eval_job.py**: Poll Bedrock evaluation job status:
- Call `bedrock.get_evaluation_job(jobIdentifier=job_arn)` in a loop
- Log status transitions and progress
- On completion, download results from output S3 path
- On failure, log failure reasons and raise error

**parse_eval_results.py**: Parse Bedrock evaluation output:
- Read evaluation results JSON from S3
- Extract per-metric scores and per-sample results
- Convert to standardized format matching the automated metrics output schema
- Return parsed results for report aggregation

**state_machine.py**: Step Functions state machine for human evaluation:
- State 1: Sample Selection — randomly select HUMAN_SAMPLE_SIZE samples from eval dataset
- State 2: Fan Out — for each sample, create a DynamoDB record with status `PENDING`
- State 3: Notify Reviewers — publish SNS notification with review link (API Gateway URL)
- State 4: Wait for Ratings — poll DynamoDB until all samples have ratings or timeout (48 hours)
- State 5: Aggregate — compute average rating, inter-rater agreement, per-category scores
- State 6: Store Results — write aggregated human eval results to S3

**send_notifications.py**: Publish SNS notification to HUMAN_REVIEWER_EMAILS with:
- Evaluation job ID and description
- Link to API Gateway rating endpoint
- Number of samples to review
- Deadline for completion

**api_handler.py**: Lambda handler for API Gateway POST `/rate`:
- Accept `{"sample_id": "...", "rating": 4, "reviewer": "...", "feedback": "..."}`
- Validate rating is within HUMAN_RATING_SCALE range
- Store rating in DynamoDB table `{PROJECT_NAME}-human-eval-{ENV}`
- Return confirmation response
- Also handle GET `/samples` to return pending samples for a reviewer

**aggregate_ratings.py**: Aggregate human evaluation ratings:
- Query all ratings from DynamoDB for the evaluation run
- Compute per-sample average rating (when multiple reviewers)
- Compute overall average, median, standard deviation
- Compute inter-rater agreement (Cohen's kappa for 2 raters, Fleiss' kappa for 3+)
- Compute per-category breakdown
- Output aggregated results as JSON

**create_human_eval_resources.py**: Create infrastructure for human evaluation:
- SNS topic `{PROJECT_NAME}-eval-notifications-{ENV}` with email subscriptions
- DynamoDB table `{PROJECT_NAME}-human-eval-{ENV}` with `sample_id` partition key
- API Gateway REST API `{PROJECT_NAME}-eval-api-{ENV}` with `/rate` POST and `/samples` GET endpoints
- Lambda function for API handler

**create_ab_endpoint.py**: Create SageMaker endpoint with production variants:
- For each model in MODEL_VARIANTS, create a SageMaker Model
- Create EndpointConfig with production variants, each with configurable `InitialVariantWeight` based on AB_TRAFFIC_SPLIT
- Create Endpoint with the multi-variant config
- Wait for endpoint to be InService

**run_ab_test.py**: Execute A/B test:
- Send evaluation prompts to the endpoint (SageMaker routes to variants based on weights)
- Collect per-variant responses with `InvokedProductionVariant` from response metadata
- Measure per-variant latency, error rate, and response quality
- Run for AB_DURATION_HOURS or until all eval samples are processed
- Store per-variant results to S3

**analyze_ab_results.py**: Statistical analysis of A/B test results:
- Compute per-variant metrics (latency p50/p95/p99, error rate, quality scores)
- Run statistical significance test (two-sample t-test or Mann-Whitney U) on quality metrics
- Compute confidence intervals for metric differences
- Determine winner variant with statistical significance
- Output analysis as JSON

**cleanup_ab_endpoint.py**: Delete A/B test endpoint, endpoint config, and models to prevent ongoing charges.

**aggregate_report.py**: Combine all evaluation results into a single report:
- Load automated metrics (ROUGE/BLEU/BERTScore) from S3
- Load Bedrock evaluation results from S3
- Load human evaluation aggregated ratings from S3
- Load A/B test analysis from S3
- Merge into unified JSON report with sections: `automated_metrics`, `bedrock_eval`, `human_eval`, `ab_test`, `summary`, `promotion_decision`
- Compute overall promotion decision based on THRESHOLD_CONFIG
- Upload report to OUTPUT_S3_PATH

**promotion_gate.py**: EventBridge rule for model promotion gating:
- Create EventBridge rule that listens for `SageMaker Model Package State Change` events with `ApprovalStatus: PendingManualApproval`
- Lambda target that:
  1. Retrieves the latest evaluation report from S3
  2. Checks all metrics against THRESHOLD_CONFIG thresholds
  3. If all thresholds met: approve the model package via `sagemaker.update_model_package()`
  4. If any threshold not met: reject the model package and send SNS notification with failing metrics
- Create EventBridge rule and Lambda function

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Evaluation Dataset:** JSONL format with `prompt` and `reference` fields required. Optional `category` field for per-category breakdown. Dataset must be in the same region as the evaluation resources. For Bedrock evaluation jobs, dataset must conform to Bedrock prompt dataset schema.

**Automated Metrics:** ROUGE and BLEU require tokenized text — use whitespace tokenization for English. BERTScore requires a GPU instance or will fall back to CPU (slower). For large datasets (10K+ samples), use SageMaker Processing jobs for BERTScore computation. Cache BERTScore model weights in S3 to avoid repeated downloads.

**Bedrock Evaluation:** Evaluation jobs support task types: Summarization, QuestionAndAnswer, Classification, TextGeneration. Built-in metrics include Builtin.Accuracy, Builtin.Robustness, Builtin.Toxicity. Custom prompt datasets must have CORS permissions on the S3 bucket. Jobs are asynchronous — poll with `get_evaluation_job()`.

**Human Evaluation:** Minimum 2 reviewers per sample for inter-rater agreement. Set a 48-hour deadline for review completion. Store all ratings in DynamoDB with TTL for cleanup. API Gateway endpoint must be authenticated (IAM or Cognito) in production.

**A/B Testing:** SageMaker production variants split traffic at the endpoint level. Use `TargetVariant` header to force traffic to a specific variant for controlled testing. Delete A/B endpoints after testing to avoid ongoing charges. Minimum 100 invocations per variant for statistical significance.

**Promotion Gating:** EventBridge rule evaluates metrics against THRESHOLD_CONFIG. All thresholds must be met for automatic approval. Any threshold failure triggers rejection with detailed notification. In dev, thresholds can be relaxed. In prod, always require human approval in addition to metric thresholds.

**Cost:** Bedrock evaluation jobs are billed per model invocation. BERTScore computation on GPU instances adds cost — use spot instances for batch processing. A/B test endpoints incur SageMaker hosting charges for the test duration. Delete all test resources after evaluation completes.

**Security:** Evaluation IAM role needs `bedrock:CreateEvaluationJob`, `bedrock:GetEvaluationJob`, `s3:GetObject`/`PutObject` on eval data and output paths, `sagemaker:*` for A/B endpoints, `states:*` for Step Functions, `sns:Publish` for notifications. Use VPC endpoints for all API calls in private subnets.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Bedrock eval job: `{PROJECT_NAME}-eval-{BEDROCK_EVAL_TASK_TYPE}-{ENV}`
- A/B endpoint: `{PROJECT_NAME}-ab-test-{ENV}`
- Step Functions: `{PROJECT_NAME}-human-eval-{ENV}`
- DynamoDB table: `{PROJECT_NAME}-human-eval-{ENV}`
- SNS topic: `{PROJECT_NAME}-eval-notifications-{ENV}`
- API Gateway: `{PROJECT_NAME}-eval-api-{ENV}`

---

## Code Scaffolding Hints


**Compute ROUGE scores:**
```python
from rouge_score import rouge_scorer

def compute_rouge(predictions: list[str], references: list[str]) -> dict:
    """Compute ROUGE-1, ROUGE-2, ROUGE-L scores."""
    scorer = rouge_scorer.RougeScorer(["rouge1", "rouge2", "rougeL"], use_stemmer=True)

    scores = {"rouge1": [], "rouge2": [], "rougeL": []}
    for pred, ref in zip(predictions, references):
        result = scorer.score(ref, pred)
        for key in scores:
            scores[key].append({
                "precision": result[key].precision,
                "recall": result[key].recall,
                "fmeasure": result[key].fmeasure,
            })

    # Aggregate
    aggregated = {}
    for key in scores:
        f1_scores = [s["fmeasure"] for s in scores[key]]
        aggregated[key] = {
            "mean_f1": sum(f1_scores) / len(f1_scores),
            "median_f1": sorted(f1_scores)[len(f1_scores) // 2],
            "min_f1": min(f1_scores),
            "max_f1": max(f1_scores),
        }

    return {"per_sample": scores, "aggregate": aggregated}
```

**Compute BLEU score:**
```python
from nltk.translate.bleu_score import corpus_bleu, SmoothingFunction

def compute_bleu(predictions: list[str], references: list[str]) -> dict:
    """Compute corpus-level BLEU score with smoothing."""
    # Tokenize by whitespace
    ref_tokens = [[ref.split()] for ref in references]  # list of list of reference tokens
    pred_tokens = [pred.split() for pred in predictions]

    smoothie = SmoothingFunction().method1

    bleu_4 = corpus_bleu(ref_tokens, pred_tokens, smoothing_function=smoothie)
    bleu_1 = corpus_bleu(ref_tokens, pred_tokens, weights=(1, 0, 0, 0), smoothing_function=smoothie)
    bleu_2 = corpus_bleu(ref_tokens, pred_tokens, weights=(0.5, 0.5, 0, 0), smoothing_function=smoothie)

    return {
        "bleu_1": bleu_1,
        "bleu_2": bleu_2,
        "bleu_4": bleu_4,
    }
```

**Compute BERTScore:**
```python
from bert_score import score as bert_score_fn

def compute_bertscore(predictions: list[str], references: list[str], model_type: str = "microsoft/deberta-xlarge-mnli") -> dict:
    """Compute BERTScore precision, recall, F1."""
    P, R, F1 = bert_score_fn(
        predictions,
        references,
        model_type=model_type,
        lang="en",
        verbose=True,
    )

    return {
        "precision": {"mean": P.mean().item(), "std": P.std().item()},
        "recall": {"mean": R.mean().item(), "std": R.std().item()},
        "f1": {"mean": F1.mean().item(), "std": F1.std().item()},
        "per_sample_f1": F1.tolist(),
    }
```

**Create Bedrock Model Evaluation job:**
```python
import boto3

bedrock = boto3.client("bedrock", region_name=AWS_REGION)

response = bedrock.create_evaluation_job(
    jobName=f"{PROJECT_NAME}-eval-{BEDROCK_EVAL_TASK_TYPE.lower()}-{ENV}",
    jobDescription=f"LLM evaluation for {PROJECT_NAME} in {ENV}",
    roleArn=evaluation_role_arn,
    evaluationConfig={
        "automated": {
            "datasetMetricConfigs": [
                {
                    "taskType": BEDROCK_EVAL_TASK_TYPE,  # "QuestionAndAnswer", "Summarization", etc.
                    "dataset": {
                        "name": f"{PROJECT_NAME}-eval-dataset",
                        "datasetLocation": {
                            "s3Uri": EVAL_DATASET_S3,
                        },
                    },
                    "metricNames": BEDROCK_EVAL_METRICS,  # ["Builtin.Accuracy", "Builtin.Robustness"]
                }
            ]
        }
    },
    inferenceConfig={
        "models": [
            {
                "bedrockModel": {
                    "modelIdentifier": model_id,  # e.g. "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0"
                    "inferenceParams": '{"inferenceConfig":{"maxTokens":512,"temperature":0.7,"topP":0.9}}',
                }
            }
        ]
    },
    outputDataConfig={
        "s3Uri": f"{OUTPUT_S3_PATH}bedrock-eval/",
    },
    jobTags=[
        {"key": "Project", "value": PROJECT_NAME},
        {"key": "Environment", "value": ENV},
    ],
)

eval_job_arn = response["jobArn"]
```

**Monitor Bedrock evaluation job:**
```python
import time

def monitor_eval_job(job_arn: str, poll_interval: int = 60) -> dict:
    """Poll Bedrock evaluation job until completion."""
    while True:
        response = bedrock.get_evaluation_job(jobIdentifier=job_arn)
        status = response["status"]
        print(f"Evaluation job status: {status}")

        if status == "Completed":
            return {
                "status": "Completed",
                "output_s3": response["outputDataConfig"]["s3Uri"],
                "creation_time": str(response.get("creationTime", "")),
            }
        elif status in ("Failed", "Stopped"):
            failure_messages = response.get("failureMessages", [])
            raise RuntimeError(f"Evaluation job {status}: {failure_messages}")

        time.sleep(poll_interval)
```

**Step Functions state machine for human evaluation:**
```python
import json

state_machine_definition = {
    "Comment": f"Human evaluation workflow for {PROJECT_NAME}",
    "StartAt": "SelectSamples",
    "States": {
        "SelectSamples": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-select-samples-{ENV}",
            "Parameters": {
                "dataset_s3": EVAL_DATASET_S3,
                "sample_size": HUMAN_SAMPLE_SIZE,
            },
            "ResultPath": "$.samples",
            "Next": "CreateReviewTasks",
        },
        "CreateReviewTasks": {
            "Type": "Map",
            "ItemsPath": "$.samples.selected",
            "MaxConcurrency": 0,
            "Iterator": {
                "StartAt": "StoreSample",
                "States": {
                    "StoreSample": {
                        "Type": "Task",
                        "Resource": "arn:aws:states:::dynamodb:putItem",
                        "Parameters": {
                            "TableName": f"{PROJECT_NAME}-human-eval-{ENV}",
                            "Item": {
                                "sample_id": {"S.$": "$.sample_id"},
                                "prompt": {"S.$": "$.prompt"},
                                "prediction": {"S.$": "$.prediction"},
                                "reference": {"S.$": "$.reference"},
                                "status": {"S": "PENDING"},
                            },
                        },
                        "End": True,
                    }
                },
            },
            "ResultPath": "$.tasks",
            "Next": "NotifyReviewers",
        },
        "NotifyReviewers": {
            "Type": "Task",
            "Resource": "arn:aws:states:::sns:publish",
            "Parameters": {
                "TopicArn": f"arn:aws:sns:{AWS_REGION}:{AWS_ACCOUNT_ID}:{PROJECT_NAME}-eval-notifications-{ENV}",
                "Subject": f"LLM Evaluation Review Required - {PROJECT_NAME}",
                "Message.$": "States.Format('Please review {} samples at: https://{}.execute-api.{}.amazonaws.com/{}/samples', $.samples.count, $.api_id, $.region, $.stage)",
            },
            "ResultPath": "$.notification",
            "Next": "WaitForRatings",
        },
        "WaitForRatings": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-check-ratings-{ENV}",
            "Parameters": {
                "table_name": f"{PROJECT_NAME}-human-eval-{ENV}",
                "expected_count": HUMAN_SAMPLE_SIZE,
            },
            "ResultPath": "$.rating_status",
            "Next": "CheckComplete",
        },
        "CheckComplete": {
            "Type": "Choice",
            "Choices": [
                {
                    "Variable": "$.rating_status.all_rated",
                    "BooleanEquals": True,
                    "Next": "AggregateRatings",
                }
            ],
            "Default": "WaitTimer",
        },
        "WaitTimer": {
            "Type": "Wait",
            "Seconds": 3600,
            "Next": "WaitForRatings",
        },
        "AggregateRatings": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-aggregate-ratings-{ENV}",
            "Parameters": {
                "table_name": f"{PROJECT_NAME}-human-eval-{ENV}",
                "output_s3": f"{OUTPUT_S3_PATH}human-eval/",
            },
            "End": True,
        },
    },
}
```

**SageMaker A/B testing with production variants:**
```python
import boto3

sagemaker = boto3.client("sagemaker", region_name=AWS_REGION)

# Create endpoint config with production variants
variant_configs = []
weights = [int(w) for w in AB_TRAFFIC_SPLIT.split("/")]

for i, model_variant in enumerate(MODEL_VARIANTS):
    # Create SageMaker Model for each variant
    sagemaker.create_model(
        ModelName=f"{PROJECT_NAME}-ab-{model_variant['name']}-{ENV}",
        PrimaryContainer={
            "Image": inference_image_uri,
            "ModelDataUrl": model_variant.get("model_data_url", ""),
            "Environment": {
                "MODEL_ID": model_variant["model_id"],
            },
        },
        ExecutionRoleArn=sagemaker_role_arn,
    )

    variant_configs.append({
        "VariantName": model_variant["name"],
        "ModelName": f"{PROJECT_NAME}-ab-{model_variant['name']}-{ENV}",
        "InitialInstanceCount": 1,
        "InstanceType": AB_ENDPOINT_INSTANCE,
        "InitialVariantWeight": weights[i] / 100.0,
    })

sagemaker.create_endpoint_config(
    EndpointConfigName=f"{PROJECT_NAME}-ab-test-config-{ENV}",
    ProductionVariants=variant_configs,
)

sagemaker.create_endpoint(
    EndpointName=f"{PROJECT_NAME}-ab-test-{ENV}",
    EndpointConfigName=f"{PROJECT_NAME}-ab-test-config-{ENV}",
)
```

**Invoke A/B endpoint and collect per-variant metrics:**
```python
import time

sagemaker_runtime = boto3.client("sagemaker-runtime", region_name=AWS_REGION)

def invoke_ab_endpoint(endpoint_name: str, prompt: str, target_variant: str = None) -> dict:
    """Invoke A/B test endpoint and record which variant served the request."""
    invoke_params = {
        "EndpointName": endpoint_name,
        "Body": json.dumps({"prompt": prompt}),
        "ContentType": "application/json",
    }
    if target_variant:
        invoke_params["TargetVariant"] = target_variant

    start = time.time()
    response = sagemaker_runtime.invoke_endpoint(**invoke_params)
    latency_ms = (time.time() - start) * 1000

    return {
        "prediction": json.loads(response["Body"].read()),
        "variant": response["InvokedProductionVariant"],
        "latency_ms": latency_ms,
    }
```

**EventBridge promotion gate — Lambda handler:**
```python
import boto3
import json

s3 = boto3.client("s3")
sagemaker = boto3.client("sagemaker")
sns = boto3.client("sns")

def lambda_handler(event, context):
    """EventBridge handler: gate model promotion based on evaluation thresholds."""
    detail = event.get("detail", {})
    model_package_arn = detail.get("ModelPackageArn", "")
    approval_status = detail.get("ModelApprovalStatus", "")

    if approval_status != "PendingManualApproval":
        return {"action": "skip", "reason": "Not a pending approval event"}

    # Load latest evaluation report
    report = load_latest_eval_report(OUTPUT_S3_PATH)
    if not report:
        return reject_model(model_package_arn, "No evaluation report found")

    # Check thresholds
    failures = []
    thresholds = THRESHOLD_CONFIG

    automated = report.get("automated_metrics", {})
    if "rouge_l_f1" in thresholds:
        rouge_l = automated.get("rougeL", {}).get("mean_f1", 0)
        if rouge_l < thresholds["rouge_l_f1"]:
            failures.append(f"ROUGE-L F1 {rouge_l:.3f} < {thresholds['rouge_l_f1']}")

    if "bleu" in thresholds:
        bleu = automated.get("bleu_4", 0)
        if bleu < thresholds["bleu"]:
            failures.append(f"BLEU {bleu:.3f} < {thresholds['bleu']}")

    if "bertscore_f1" in thresholds:
        bert_f1 = automated.get("bertscore", {}).get("f1", {}).get("mean", 0)
        if bert_f1 < thresholds["bertscore_f1"]:
            failures.append(f"BERTScore F1 {bert_f1:.3f} < {thresholds['bertscore_f1']}")

    if "human_avg_rating" in thresholds:
        human_avg = report.get("human_eval", {}).get("average_rating", 0)
        if human_avg < thresholds["human_avg_rating"]:
            failures.append(f"Human avg rating {human_avg:.2f} < {thresholds['human_avg_rating']}")

    if failures:
        return reject_model(model_package_arn, "; ".join(failures))
    else:
        return approve_model(model_package_arn)


def approve_model(model_package_arn: str) -> dict:
    """Approve model package in SageMaker Model Registry."""
    sagemaker.update_model_package(
        ModelPackageArn=model_package_arn,
        ModelApprovalStatus="Approved",
        ApprovalDescription="Automated approval: all evaluation thresholds met",
    )
    return {"action": "approved", "model_package_arn": model_package_arn}


def reject_model(model_package_arn: str, reason: str) -> dict:
    """Reject model package and notify via SNS."""
    sagemaker.update_model_package(
        ModelPackageArn=model_package_arn,
        ModelApprovalStatus="Rejected",
        ApprovalDescription=f"Automated rejection: {reason}",
    )
    sns.publish(
        TopicArn=f"arn:aws:sns:{AWS_REGION}:{AWS_ACCOUNT_ID}:{PROJECT_NAME}-eval-notifications-{ENV}",
        Subject=f"Model Promotion Blocked - {PROJECT_NAME}",
        Message=f"Model package {model_package_arn} was rejected.\nReason: {reason}",
    )
    return {"action": "rejected", "model_package_arn": model_package_arn, "reason": reason}
```

**Create EventBridge rule for promotion gating:**
```python
events = boto3.client("events", region_name=AWS_REGION)

events.put_rule(
    Name=f"{PROJECT_NAME}-eval-promotion-gate-{ENV}",
    Description="Gate model promotion based on evaluation metric thresholds",
    EventPattern=json.dumps({
        "source": ["aws.sagemaker"],
        "detail-type": ["SageMaker Model Package State Change"],
        "detail": {
            "ModelApprovalStatus": ["PendingManualApproval"],
        },
    }),
    State="ENABLED",
    Tags=[
        {"Key": "Project", "Value": PROJECT_NAME},
        {"Key": "Environment", "Value": ENV},
    ],
)

events.put_targets(
    Rule=f"{PROJECT_NAME}-eval-promotion-gate-{ENV}",
    Targets=[
        {
            "Id": "promotion-gate-lambda",
            "Arn": f"arn:aws:lambda:{AWS_REGION}:{AWS_ACCOUNT_ID}:function:{PROJECT_NAME}-promotion-gate-{ENV}",
        }
    ],
)
```

**Evaluation report aggregator:**
```python
import json
from datetime import datetime

def aggregate_report(
    automated_results: dict = None,
    bedrock_eval_results: dict = None,
    human_eval_results: dict = None,
    ab_test_results: dict = None,
    threshold_config: dict = None,
) -> dict:
    """Combine all evaluation results into a single dashboard-ready report."""
    report = {
        "report_id": f"{PROJECT_NAME}-eval-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}",
        "project": PROJECT_NAME,
        "environment": ENV,
        "timestamp": datetime.utcnow().isoformat(),
        "automated_metrics": automated_results or {},
        "bedrock_eval": bedrock_eval_results or {},
        "human_eval": human_eval_results or {},
        "ab_test": ab_test_results or {},
    }

    # Compute promotion decision
    promotion = {"meets_all_thresholds": True, "failures": [], "thresholds": threshold_config or {}}

    if threshold_config and automated_results:
        checks = [
            ("rouge_l_f1", automated_results.get("rougeL", {}).get("mean_f1", 0)),
            ("bleu", automated_results.get("bleu_4", 0)),
            ("bertscore_f1", automated_results.get("bertscore", {}).get("f1", {}).get("mean", 0)),
        ]
        for metric_key, actual_value in checks:
            if metric_key in threshold_config and actual_value < threshold_config[metric_key]:
                promotion["meets_all_thresholds"] = False
                promotion["failures"].append({
                    "metric": metric_key,
                    "actual": actual_value,
                    "threshold": threshold_config[metric_key],
                })

    report["promotion_decision"] = promotion
    return report
```

---

## Integration Points

- **Upstream**: `mlops/09` → Bedrock fine-tuned custom model to evaluate against base model
- **Upstream**: `mlops/10` → Model Registry provides model package ARNs for promotion gating; evaluation metrics stored as model package metadata
- **Upstream**: `devops/04` → IAM roles for Bedrock evaluation, SageMaker endpoints, Step Functions, Lambda, and API Gateway
- **Downstream**: `enterprise/05` → Centralized Model Registry consumes evaluation metrics for cross-account model governance
- **Downstream**: `mlops/13` → Continuous training pipeline uses evaluation results to decide champion-challenger promotion
- **Downstream**: `devops/11` → Custom CloudWatch metrics for evaluation scores published as model quality indicators
