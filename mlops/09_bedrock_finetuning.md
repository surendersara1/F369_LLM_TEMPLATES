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
                        Options (verify current availability with:
                        aws bedrock list-foundation-models --by-customization-type FINE_TUNING):
                        - amazon.titan-text-express-v1 (Titan Text Express)
                        - amazon.titan-text-lite-v1 (Titan Text Lite)
                        - amazon.titan-text-premier-v1:0 (Titan Text Premier)
                        - meta.llama3-1-8b-instruct-v1:0 (Llama 3.1 8B)
                        - meta.llama3-1-70b-instruct-v1:0 (Llama 3.1 70B)
                        - meta.llama3-2-1b-instruct-v1:0 (Llama 3.2 1B)
                        - meta.llama3-2-3b-instruct-v1:0 (Llama 3.2 3B)
                        - meta.llama3-3-70b-instruct-v1:0 (Llama 3.3 70B)
                        - anthropic.claude-3-haiku-20240307-v1:0 (Claude 3 Haiku)

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
- Llama 3.x/Meta format (Llama 3.1 8B/70B — non-conversational):
  `{"prompt": "<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\nYour question here<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n", "completion": "Expected answer"}`
- Llama 3.2/3.3 models use Converse API format (see AWS docs for `bedrock-conversation-2024` schema)
- Note: Llama 2 `<s>[INST] ... [/INST]` format is NOT compatible with Llama 3.x models
- Validate max token counts per model limit
- Sample and log examples for verification

**validate_dataset.py**: Check Bedrock constraints:
- Min records vary by model (e.g., Titan: check current quotas; Llama 3.1: min 100 records)
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

**Invoke custom model (Titan):**
```python
# Amazon Titan Text models use inputText / textGenerationConfig
response = bedrock_runtime.invoke_model(
    modelId=custom_model_arn,  # or provisioned throughput ARN
    body=json.dumps({
        "inputText": "Your prompt here",
        "textGenerationConfig": {
            "maxTokenCount": 512,
            "temperature": 0.7,
            "topP": 0.9
        }
    })
)
result = json.loads(response["body"].read())
output_text = result["results"][0]["outputText"]
```

**Invoke custom model (Meta Llama 3.x):**
```python
# Meta Llama 3.x models use prompt / max_gen_len
response = bedrock_runtime.invoke_model(
    modelId=custom_model_arn,  # or provisioned throughput ARN
    body=json.dumps({
        "prompt": "<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\nYour prompt here<|eot_id|><|start_header_id|>assistant<|end_header_id|>\n\n",
        "max_gen_len": 512,
        "temperature": 0.7,
        "top_p": 0.9
    })
)
result = json.loads(response["body"].read())
output_text = result["generation"]
```

---

## Integration Points

- **Upstream**: `devops/04` → Bedrock customization IAM role
- **Upstream**: `mlops/02` → alternative: SageMaker fine-tuning for OSS models
- **Downstream**: `mlops/10` → register custom model metadata in model registry
- **Downstream**: `mlops/04` → use fine-tuned model as RAG generation model
- **Downstream**: `cicd/03` → CodePipeline triggers fine-tuning + deployment
