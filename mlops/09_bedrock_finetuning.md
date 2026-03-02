<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template 09 — Bedrock Custom Model Fine-tuning

## Purpose
Generate a production-ready Amazon Bedrock custom model fine-tuning pipeline: dataset preparation (JSONL), fine-tuning job creation via Bedrock API, custom model deployment with provisioned throughput, and evaluation against base model.

---

## Role Definition

You are an expert AWS AI engineer specializing in Amazon Bedrock with expertise in:
- Bedrock Custom Models: fine-tuning and continued pre-training
- Bedrock Provisioned Throughput for custom model serving
- Training data preparation (JSONL format, validation, augmentation)
- Model evaluation: comparing fine-tuned vs base model quality
- Cost optimization for Bedrock fine-tuning and inference
- IAM policies and S3 configuration for Bedrock access

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must be a Bedrock-supported region]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

BASE_MODEL_ID:          [REQUIRED - Bedrock model identifier]
                        Options:
                        - amazon.titan-text-express-v1 (Titan Text)
                        - amazon.titan-text-lite-v1 (Titan Text Lite)
                        - amazon.titan-text-premier-v1:0 (Titan Premier)
                        - cohere.command-light-text-v14 (Cohere Command)
                        - meta.llama3-1-8b-instruct-v1:0 (Llama 3.1 8B)
                        - meta.llama3-1-70b-instruct-v1:0 (Llama 3.1 70B)

CUSTOMIZATION_TYPE:     [OPTIONAL: FINE_TUNING | CONTINUED_PRE_TRAINING]
TRAINING_DATA_S3:       [REQUIRED - s3://bucket/path/to/training.jsonl]
VALIDATION_DATA_S3:     [OPTIONAL - s3://bucket/path/to/validation.jsonl]

CUSTOM_MODEL_NAME:      [OPTIONAL: {PROJECT_NAME}-{ENV}-custom-model]
CUSTOM_MODEL_JOB_NAME:  [OPTIONAL: {PROJECT_NAME}-{ENV}-ft-job-{timestamp}]

HYPERPARAMETERS:        [OPTIONAL]
    EPOCHS:             [OPTIONAL: 2]
    BATCH_SIZE:         [OPTIONAL: 8]
    LEARNING_RATE:      [OPTIONAL: 0.00001]
    WARMUP_RATIO:       [OPTIONAL: 0.1]

PROVISIONED_THROUGHPUT: [OPTIONAL: true for prod, false for dev]
MODEL_UNITS:            [OPTIONAL: 1 - number of provisioned model units]

OUTPUT_S3_PATH:         [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/bedrock-models/]
EVALUATION_PROMPTS_S3:  [OPTIONAL - s3://bucket/path/to/eval_prompts.jsonl for quality comparison]
```

---

## Task

Generate complete Bedrock fine-tuning pipeline:

```
bedrock_finetuning/
├── data_prep/
│   ├── prepare_training_data.py   # Convert raw data to Bedrock JSONL format
│   ├── validate_dataset.py        # Validate against Bedrock schema requirements
│   └── split_dataset.py           # Train/validation split
├── finetuning/
│   ├── create_finetuning_job.py   # Bedrock CreateModelCustomizationJob
│   ├── monitor_job.py             # Poll job status + CloudWatch metrics
│   └── config.py                  # All config in one place
├── deployment/
│   ├── create_provisioned_throughput.py  # Provisioned throughput for custom model
│   ├── invoke_custom_model.py     # Test invocations against custom model
│   └── delete_resources.py        # Cleanup provisioned throughput + custom model
├── evaluation/
│   ├── compare_models.py          # Side-by-side base vs fine-tuned comparison
│   └── generate_eval_report.py    # Quality metrics report
├── run_finetuning.py              # CLI orchestrator
└── requirements.txt
```

**prepare_training_data.py**: Convert various formats to Bedrock JSONL:
- Titan format: `{"prompt": "...", "completion": "..."}`
- Llama/Meta format: `{"prompt": "<s>[INST] ... [/INST]", "completion": "..."}`
- Validate max token counts per model limit
- Sample and log examples for verification

**validate_dataset.py**: Check Bedrock constraints:
- Min 1000 records (Titan), min 32 records (Llama/Meta)
- Max record size varies by model
- No duplicate records
- Validate JSON schema per record
- Report stats: total records, avg tokens, max tokens

**create_finetuning_job.py**: `bedrock.create_model_customization_job()`:
```python
response = bedrock.create_model_customization_job(
    jobName=CUSTOM_MODEL_JOB_NAME,
    customModelName=CUSTOM_MODEL_NAME,
    roleArn=bedrock_customization_role_arn,
    baseModelIdentifier=BASE_MODEL_ID,
    customizationType=CUSTOMIZATION_TYPE,
    trainingDataConfig={"s3Uri": TRAINING_DATA_S3},
    validationDataConfig={"validators": [{"s3Uri": VALIDATION_DATA_S3}]},
    outputDataConfig={"s3Uri": OUTPUT_S3_PATH},
    hyperParameters={"epochCount": str(EPOCHS), "batchSize": str(BATCH_SIZE),
                     "learningRate": str(LEARNING_RATE), "warmupRatio": str(WARMUP_RATIO)}
)
```

**create_provisioned_throughput.py**: For prod deployment:
```python
response = bedrock.create_provisioned_model_throughput(
    modelUnits=MODEL_UNITS,
    provisionedModelName=f"{PROJECT_NAME}-{ENV}-provisioned",
    modelId=custom_model_arn  # ARN from fine-tuning job output
)
```

**compare_models.py**: Run same prompts against base model and fine-tuned model, compute ROUGE/BLEU/perplexity, generate comparison table.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Data:** Bedrock has strict JSONL schema per model family. Validate before submitting. Min/max record counts vary. Training data must be in same region as Bedrock.

**Cost:** Fine-tuning: billed per token processed. Titan Express ~$0.0004/1K tokens. Provisioned throughput: $x/hour per model unit (varies by model). Delete provisioned throughput in dev when not in use.

**Security:** Bedrock customization role needs `bedrock:*`, `s3:GetObject` on training data, `s3:PutObject` on output path. Use VPC endpoint for Bedrock API in private subnet.

**Timing:** Fine-tuning jobs can take hours to days depending on dataset size and model. Use EventBridge rule on `bedrock.customization.job.completed` for async notification.

---

## Code Scaffolding Hints

**Monitor job status:**
```python
while True:
    response = bedrock.get_model_customization_job(jobIdentifier=job_name)
    status = response["status"]
    if status in ["Completed", "Failed", "Stopped"]:
        break
    time.sleep(60)
```

**Invoke custom model:**
```python
response = bedrock_runtime.invoke_model(
    modelId=custom_model_arn,  # or provisioned throughput ARN
    body=json.dumps({"prompt": "Your prompt here", "maxTokens": 512, "temperature": 0.7})
)
```

---

## Integration Points

- **Upstream**: `devops/04` → Bedrock customization IAM role
- **Upstream**: `mlops/02` → alternative: SageMaker fine-tuning for OSS models
- **Downstream**: `mlops/10` → register custom model metadata in model registry
- **Downstream**: `mlops/04` → use fine-tuned model as RAG generation model
- **Downstream**: `cicd/03` → CodePipeline triggers fine-tuning + deployment
