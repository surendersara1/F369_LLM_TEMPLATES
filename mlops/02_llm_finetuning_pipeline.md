<!-- Template Version: 1.0 | boto3: 1.35+ | transformers: 4.45+ | peft: 0.12+ -->

# Template 02 - LLM Fine-tuning Pipeline (LoRA/QLoRA on SageMaker)

## Purpose
Generate a production-ready LLM fine-tuning pipeline on AWS SageMaker using LoRA/QLoRA with PEFT, supporting Llama, Mistral, Falcon, and other open-source models stored in S3 or pulled from Hugging Face Hub.

---

## Role Definition

You are an expert AWS MLOps engineer and LLM specialist with deep expertise in:
- Parameter-Efficient Fine-Tuning (PEFT): LoRA, QLoRA, IA3, Prefix Tuning
- Hugging Face ecosystem: transformers, datasets, trl (SFTTrainer, DPOTrainer), accelerate, bitsandbytes
- SageMaker Training Jobs with GPU instances (ml.g4dn, ml.g5, ml.p3, ml.p4d)
- DeepSpeed ZeRO (Stage 2/3) for multi-GPU distributed training
- LLM evaluation: perplexity, ROUGE, BLEU, human preference metrics
- Cost optimization for GPU training (spot instances, mixed precision)

Generate complete, production-deployable code with no placeholders.

---

## Context and Inputs

```
PROJECT_NAME:           [REQUIRED - e.g. legal-doc-assistant]
AWS_REGION:             [REQUIRED - e.g. us-east-1]
AWS_ACCOUNT_ID:         [REQUIRED - 12-digit]
ENV:                    [REQUIRED - dev | stage | prod]

BASE_MODEL_ID:          [REQUIRED - HuggingFace model ID or S3 path]
                        Examples:
                        - meta-llama/Llama-3.1-8B-Instruct
                        - mistralai/Mistral-7B-Instruct-v0.3
                        - microsoft/phi-3-mini-4k-instruct
                        - tiiuae/falcon-7b-instruct

TRAINING_TASK:          [REQUIRED - sft | dpo | reward_modeling | classification]
DATASET_S3_PATH:        [REQUIRED - s3://bucket/path/to/dataset/]
DATASET_FORMAT:         [REQUIRED - jsonl | csv | parquet]
                        JSONL format: {"instruction": "...", "input": "...", "output": "..."}
                        or for DPO: {"prompt": "...", "chosen": "...", "rejected": "..."}

TRAINING_INSTANCE_TYPE: [OPTIONAL: ml.g5.2xlarge (1x A10G 24GB)]
                        Options: ml.g4dn.xlarge (T4 16GB), ml.g5.2xlarge (A10G 24GB),
                                 ml.g5.12xlarge (4x A10G), ml.p3.2xlarge (V100 16GB),
                                 ml.p3.16xlarge (8x V100), ml.p4d.24xlarge (8x A100)
INSTANCE_COUNT:         [OPTIONAL: 1]

PEFT_METHOD:            [OPTIONAL: lora]
                        Options: lora | qlora | ia3 | prefix_tuning
LORA_RANK:              [OPTIONAL: 16]
LORA_ALPHA:             [OPTIONAL: 32]
LORA_DROPOUT:           [OPTIONAL: 0.05]
LORA_TARGET_MODULES:    [OPTIONAL: q_proj,v_proj,k_proj,o_proj,gate_proj,up_proj,down_proj]

QUANTIZATION:           [OPTIONAL: none | 4bit | 8bit]
                        Use 4bit (QLoRA) for memory-constrained instances
MAX_SEQ_LENGTH:         [OPTIONAL: 2048]

LEARNING_RATE:          [OPTIONAL: 2e-4]
NUM_EPOCHS:             [OPTIONAL: 3]
BATCH_SIZE_PER_GPU:     [OPTIONAL: 4]
GRADIENT_ACCUMULATION:  [OPTIONAL: 8]
WARMUP_RATIO:           [OPTIONAL: 0.03]
LR_SCHEDULER:           [OPTIONAL: cosine]

DEEPSPEED_STAGE:        [OPTIONAL: 0 | 2 | 3]
                        Use 2 or 3 for multi-GPU, 0 for single GPU
MIXED_PRECISION:        [OPTIONAL: bf16 | fp16 | no]

OUTPUT_MODEL_S3_PATH:   [OPTIONAL: s3://{PROJECT_NAME}-{AWS_ACCOUNT_ID}-{ENV}-artifacts/finetuned-models/]
HF_TOKEN_SECRET_NAME:   [OPTIONAL: /mlops/{PROJECT_NAME}/huggingface-token]
                        Required if BASE_MODEL_ID requires authentication (e.g., Llama 3)
```

---

## Task

Generate a complete LLM fine-tuning pipeline with ALL components:

### 1. Project Structure
```
{PROJECT_NAME}-finetuning/
├── training/
│   ├── finetune.py              # Main training script (runs in SageMaker container)
│   ├── data_utils.py            # Dataset loading, formatting, tokenization
│   ├── model_utils.py           # Model loading, PEFT setup, quantization
│   ├── training_args.py         # TrainingArguments + DeepSpeed config
│   └── evaluate.py              # Post-training evaluation script
├── config/
│   ├── lora_config.json         # LoraConfig parameters
│   ├── deepspeed_z2.json        # DeepSpeed ZeRO Stage 2 config
│   ├── deepspeed_z3.json        # DeepSpeed ZeRO Stage 3 config
│   └── training_config.py       # All config in one dataclass
├── pipeline/
│   ├── finetuning_job.py        # SageMaker TrainingJob wrapper
│   └── run_finetuning.py        # CLI entrypoint
├── scripts/
│   ├── prepare_dataset.py       # Data preprocessing + format validation
│   └── merge_lora_weights.py    # Merge LoRA adapters into base model
├── evaluation/
│   └── run_eval.py              # Evaluation on held-out test set
├── requirements.txt
└── Dockerfile                   # Custom training container (optional)
```

### 2. training/finetune.py
Complete training script using `trl.SFTTrainer` (for sft) or `trl.DPOTrainer` (for dpo):
- Retrieve HuggingFace token from Secrets Manager at runtime (not from environment variables):
```python
import boto3, json, os

def get_hf_token(secret_name: str, region: str) -> str:
    """Retrieve HuggingFace token from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name=region)
    resp = client.get_secret_value(SecretId=secret_name)
    return json.loads(resp["SecretString"])["HUGGING_FACE_HUB_TOKEN"]

hf_token = get_hf_token(os.environ["HF_TOKEN_SECRET_NAME"], os.environ.get("AWS_REGION", "us-east-1"))
os.environ["HUGGING_FACE_HUB_TOKEN"] = hf_token  # set for transformers/hub downloads
```
- Load base model with quantization config (4-bit BitsAndBytes if QLoRA)
- Apply LoRA/PEFT config using `peft.get_peft_model()`
- Load and tokenize dataset using data_utils
- Initialize trainer with SageMaker-compatible callbacks
- Log metrics to SageMaker CloudWatch
- Save adapter weights to `/opt/ml/model/`
- Handle gradient checkpointing for memory efficiency
- Implement DeepSpeed integration for multi-GPU

### 3. training/data_utils.py
- Load dataset from `/opt/ml/input/data/train/` (JSONL/CSV/Parquet)
- Apply chat template formatting (Llama-3 / Mistral / generic Alpaca format)
- Tokenize with truncation and padding to MAX_SEQ_LENGTH
- Create train/eval split (90/10 if no separate eval set provided)
- Data collator for causal language modeling

### 4. training/model_utils.py
- Load base model with `AutoModelForCausalLM.from_pretrained()`
- 4-bit quantization: `BitsAndBytesConfig(load_in_4bit=True, bnb_4bit_compute_dtype=torch.bfloat16, bnb_4bit_use_double_quant=True, bnb_4bit_quant_type="nf4")`
- Apply LoRA: `LoraConfig(r=LORA_RANK, lora_alpha=LORA_ALPHA, target_modules=[...], lora_dropout=LORA_DROPOUT, bias="none", task_type="CAUSAL_LM")`
- Enable gradient checkpointing
- Print trainable parameter count

### 5. config/deepspeed_z2.json
Complete DeepSpeed ZeRO Stage 2 config for multi-GPU training

### 6. scripts/prepare_dataset.py
- Validate dataset schema
- Convert to standard format
- Upload processed dataset to S3
- Generate data quality report

### 7. scripts/merge_lora_weights.py
Post-training script to:
- Load base model in fp16
- Load LoRA adapter from S3
- Merge weights: `model = model.merge_and_unload()`
- Save merged model to S3 in safetensors format
- Push tokenizer alongside model

### 8. pipeline/finetuning_job.py
SageMaker HuggingFace Estimator setup:
```python
from sagemaker.huggingface import HuggingFace

# NOTE: Verify latest available DLC versions for your region before launching:
#   aws ecr describe-images --repository-name huggingface-pytorch-training \
#     --registry-id 763104351884 --region $AWS_REGION \
#     --query 'sort_by(imageDetails,&imagePushedAt)[-5:].imageTags' --output table
estimator = HuggingFace(
    entry_point="finetune.py",
    source_dir="training/",
    instance_type=TRAINING_INSTANCE_TYPE,
    instance_count=INSTANCE_COUNT,
    transformers_version="4.45",
    pytorch_version="2.5",
    py_version="py311",
    hyperparameters={...},
    metric_definitions=[...],
    use_spot_instances=(ENV != "prod"),
    max_wait=7200,
    checkpoint_s3_uri=checkpoint_path,
    # Do NOT pass HF token as plain-text env var — retrieve it from
    # Secrets Manager inside the training script (see training/finetune.py).
    # The SageMaker execution role must have secretsmanager:GetSecretValue permission.
)
```

### 9. evaluation/run_eval.py
Post-training evaluation:
- Load fine-tuned model from S3
- Run on held-out test set
- Compute: perplexity, ROUGE-1/2/L, BLEU, exact match
- Output evaluation report JSON
- Optionally run inference comparison vs base model on 10 sample prompts

---

## Output Format

Output ALL files with headers:
### FILE: {PROJECT_NAME}-finetuning/training/finetune.py
```python
[complete code]
```

---

## Requirements and Constraints

**Memory Management:**
- Enable `torch.cuda.empty_cache()` between phases
- Use gradient checkpointing when BATCH_SIZE_PER_GPU <= 4
- For QLoRA: verify GPU memory fits quantized model + gradients
- Recommended sizing: 7B model = ml.g5.2xlarge (24GB), 13B = ml.g5.12xlarge (4x24GB), 70B = ml.p4d.24xlarge (8x80GB)

**Spot Instance Recovery:**
- Save checkpoints every 100 steps to S3
- Resume from latest S3 checkpoint on restart
- Use `transformers.TrainerCallback` for S3 checkpoint sync

**Cost Optimization:**
- Use spot instances for dev (save ~70%)
- Enable `fp16` or `bf16` mixed precision (reduces VRAM 50%)
- QLoRA (4-bit) reduces memory 4x vs full fine-tuning
- Estimated costs: 7B SFT 1 epoch on 10K examples ~= $12-20 on ml.g5.2xlarge spot

**Security:**
- HuggingFace token from AWS Secrets Manager (not env vars in plain text)
- Use `boto3.client('secretsmanager').get_secret_value()` at runtime
- VPC mode: training in private subnet, no internet egress (use S3 VPC endpoint)
- All S3 paths SSE-KMS encrypted

**Model Artifacts:**
- Save: adapter_model.bin + adapter_config.json (LoRA only, ~50-200MB)
- Also save: merged full model in safetensors format to separate S3 path
- Include tokenizer files with model artifacts

---

## Code Scaffolding Hints

**QLoRA setup:**
```python
from transformers import BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4"
)
model = AutoModelForCausalLM.from_pretrained(
    model_id, quantization_config=bnb_config,
    device_map="auto", trust_remote_code=True
)
```

**PEFT LoRA:**
```python
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
model = prepare_model_for_kbit_training(model)
lora_config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj","v_proj"],
                          lora_dropout=0.05, bias="none", task_type="CAUSAL_LM")
model = get_peft_model(model, lora_config)
```

**SFTTrainer:**
```python
from trl import SFTTrainer, SFTConfig
trainer = SFTTrainer(
    model=model,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
    args=SFTConfig(
        output_dir="/opt/ml/model",
        per_device_train_batch_size=BATCH_SIZE_PER_GPU,
        gradient_accumulation_steps=GRADIENT_ACCUMULATION,
        learning_rate=LEARNING_RATE,
        num_train_epochs=NUM_EPOCHS,
        bf16=True,
        logging_steps=10,
        save_strategy="steps",
        save_steps=100,
        report_to="none",  # use custom SageMaker logging
    ),
    tokenizer=tokenizer,
    max_seq_length=MAX_SEQ_LENGTH,
    dataset_text_field="text",
)
```

---

## Integration Points

- **Upstream**: `devops/04_iam_roles_policies_mlops.md` -> SageMaker execution role with S3, ECR, Secrets Manager permissions
- **Upstream**: `devops/01_ecr_ml_docker.md` -> custom training container URI (if custom Dockerfile used)
- **Downstream**: `mlops/10_model_registry_versioning.md` -> register fine-tuned model in Model Registry
- **Downstream**: `mlops/03_llm_inference_deployment.md` -> deploy the fine-tuned model
- **Downstream**: `mlops/09_bedrock_finetuning.md` -> alternative: fine-tune via Bedrock API instead
- **Downstream**: `cicd/01_codebuild_ml_training.md` -> CodeBuild triggers the fine-tuning job
