<!-- Template Version: 1.0 | boto3: 1.35+ -->

# Template 15 — Multimodal Processing Pipelines

## Purpose
Generate production-ready multimodal AI processing pipelines for image, audio, and document modalities: Bedrock Claude vision analysis with Rekognition label/text detection, Transcribe speech-to-text with Bedrock summarization, Textract document extraction with Bedrock structured data parsing, SageMaker Processing job wrappers for batch workloads, and S3 event-driven triggers for automatic media processing.

---

## Role Definition

You are an expert AWS AI/ML engineer specializing in multimodal processing with expertise in:
- Bedrock Runtime: Claude vision analysis with base64-encoded images, multi-turn image conversations, structured output extraction
- Amazon Rekognition: label detection, text detection, face analysis, content moderation, custom labels
- Amazon Transcribe: batch and streaming transcription, custom vocabularies, speaker diarization, language identification
- Amazon Textract: document analysis with TABLES, FORMS, and QUERIES feature types, expense analysis, lending document analysis
- SageMaker Processing: ScriptProcessor and FrameworkProcessor for batch multimodal workloads with S3 input/output
- S3 event-driven architectures: EventBridge rules and S3 notifications for automatic media file processing
- File chunking and downsampling strategies for large media files that exceed API size limits
- IAM policies for cross-service access across Bedrock, Rekognition, Transcribe, Textract, and SageMaker

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED - must support Bedrock Claude vision, Rekognition, Transcribe, Textract]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

MODALITIES:             [REQUIRED - comma-separated list: image | audio | document]
                        image: Bedrock Claude vision + Rekognition label/text detection
                        audio: Transcribe speech-to-text + Bedrock summarization
                        document: Textract extraction + Bedrock structured parsing
                        Example: "image,audio,document"

PROCESSING_MODE:        [REQUIRED - realtime | batch]
                        realtime: S3 event triggers → Lambda processing per file
                        batch: SageMaker Processing jobs for bulk workloads

S3_INPUT_PATH:          [REQUIRED - s3://bucket/path/to/input/media/]
S3_OUTPUT_PATH:         [REQUIRED - s3://bucket/path/to/output/results/]

INSTANCE_TYPE:          [OPTIONAL: ml.m5.xlarge - SageMaker Processing instance type for batch mode]
                        Options:
                        - ml.m5.xlarge (general purpose, cost-effective)
                        - ml.m5.2xlarge (higher memory for large documents)
                        - ml.c5.2xlarge (compute-optimized for image processing)
INSTANCE_COUNT:         [OPTIONAL: 1 - number of SageMaker Processing instances]

VISION_MODEL:           [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]
                        Options (must support vision):
                        - anthropic.claude-3-5-sonnet-20241022-v2:0 (Claude 3.5 Sonnet v2)
                        - anthropic.claude-3-5-haiku-20241022-v1:0 (Claude 3.5 Haiku)
                        - anthropic.claude-3-haiku-20240307-v1:0 (Claude 3 Haiku — lower cost)
                        - amazon.nova-pro-v1:0 (Amazon Nova Pro)
                        - amazon.nova-lite-v1:0 (Amazon Nova Lite)

SUMMARIZATION_MODEL:    [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]
EXTRACTION_MODEL:       [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]

IMAGE_ANALYSIS_PROMPT:  [OPTIONAL: "Analyze this image and describe all objects, text, scenes, and relevant details in structured JSON."]
AUDIO_SUMMARY_PROMPT:   [OPTIONAL: "Summarize the following transcript. Include key topics, action items, and speaker highlights."]
DOCUMENT_EXTRACT_PROMPT:[OPTIONAL: "Extract all structured data from this document. Return JSON with fields, tables, and key-value pairs."]

MAX_FILE_SIZE_MB:       [OPTIONAL: 20 - maximum file size before chunking/downsampling]
TRANSCRIBE_LANGUAGE:    [OPTIONAL: en-US - language code for Transcribe]
TEXTRACT_FEATURES:      [OPTIONAL: TABLES,FORMS,QUERIES - comma-separated Textract feature types]
TEXTRACT_QUERIES:       [OPTIONAL - JSON list of Textract query strings]
                        Example: ["What is the invoice number?", "What is the total amount?"]

ENABLE_CONTENT_MODERATION: [OPTIONAL: false - enable Rekognition content moderation for images]
NOTIFICATION_TOPIC_ARN: [OPTIONAL - SNS topic ARN for processing completion notifications]
```

---

## Task

Generate complete multimodal processing pipelines:

```
{PROJECT_NAME}-multimodal/
├── config.py                              # Central configuration
├── pipelines/
│   ├── image/
│   │   ├── bedrock_vision.py              # Bedrock Claude vision analysis
│   │   ├── rekognition_analysis.py        # Rekognition label + text detection
│   │   └── image_pipeline.py             # Combined image processing orchestrator
│   ├── audio/
│   │   ├── transcribe_job.py              # Transcribe speech-to-text
│   │   ├── bedrock_summarizer.py          # Bedrock transcript summarization
│   │   └── audio_pipeline.py             # Combined audio processing orchestrator
│   └── document/
│       ├── textract_analysis.py           # Textract document extraction
│       ├── bedrock_extractor.py           # Bedrock structured data extraction
│       └── document_pipeline.py          # Combined document processing orchestrator
├── processing/
│   ├── sagemaker_processor.py             # SageMaker Processing job wrapper
│   └── batch_processing_script.py         # Script that runs inside SageMaker Processing
├── triggers/
│   ├── s3_event_handler.py                # Lambda handler for S3 event triggers
│   └── setup_event_rules.py              # Create EventBridge rules for S3 events
├── utils/
│   ├── media_utils.py                     # File validation, chunking, downsampling
│   └── s3_utils.py                        # S3 read/write helpers
├── cleanup/
│   └── delete_resources.py                # Delete Transcribe jobs, EventBridge rules
├── run_pipeline.py                        # CLI orchestrator
└── requirements.txt
```


**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate that selected MODALITIES are supported and PROCESSING_MODE is valid.

**bedrock_vision.py**: Analyze images using Bedrock Claude vision:
- Read image from S3, encode as base64
- Validate file size against MAX_FILE_SIZE_MB; if exceeded, resize image maintaining aspect ratio
- Call `bedrock_runtime.invoke_model()` with `anthropic.claude` model, passing base64 image in the `messages` content block with `type: "image"` and `source.type: "base64"`
- Parse structured JSON response from Claude
- Support multiple image formats: JPEG, PNG, GIF, WebP
- Return analysis results with confidence scores and bounding box descriptions

**rekognition_analysis.py**: Structured image analysis using Rekognition:
- Call `rekognition.detect_labels()` with S3 image reference for object and scene detection
- Call `rekognition.detect_text()` for OCR text extraction from images
- If ENABLE_CONTENT_MODERATION, call `rekognition.detect_moderation_labels()` for safety filtering
- Merge results into unified JSON with labels, text regions, and moderation flags
- Filter labels by configurable minimum confidence threshold (default 80%)

**image_pipeline.py**: Orchestrate combined image processing:
- Run Bedrock vision analysis for semantic understanding
- Run Rekognition for structured label and text extraction
- Merge results into a single output JSON per image
- Write results to S3_OUTPUT_PATH under `images/{filename}_results.json`
- Emit CloudWatch metrics: processing time, file count, error count

**transcribe_job.py**: Speech-to-text using Amazon Transcribe:
- Call `transcribe.start_transcription_job()` with S3 media URI, language code, and output bucket
- Configure speaker diarization via `Settings.ShowSpeakerLabels` if multiple speakers detected
- Poll job status with exponential backoff until `COMPLETED` or `FAILED`
- Download and parse transcript JSON from S3 output location
- Support audio formats: MP3, MP4, WAV, FLAC, OGG, AMR, WebM
- For files exceeding Transcribe limits (4 hours / 2 GB), split audio into segments

**bedrock_summarizer.py**: Summarize transcripts using Bedrock:
- Accept transcript text from Transcribe output
- If transcript exceeds model context window, chunk into overlapping segments and summarize each, then produce a final combined summary
- Call `bedrock_runtime.invoke_model()` with SUMMARIZATION_MODEL and AUDIO_SUMMARY_PROMPT
- Return structured summary with key topics, action items, and speaker highlights

**audio_pipeline.py**: Orchestrate combined audio processing:
- Run Transcribe for speech-to-text conversion
- Run Bedrock summarization on the transcript
- Merge transcript and summary into output JSON
- Write results to S3_OUTPUT_PATH under `audio/{filename}_results.json`

**textract_analysis.py**: Document extraction using Amazon Textract:
- Call `textract.start_document_analysis()` with S3 document reference and TEXTRACT_FEATURES
- If TEXTRACT_QUERIES provided, include `QueriesConfig` with query strings
- Poll with `textract.get_document_analysis()` using exponential backoff until `SUCCEEDED`
- Parse Textract response blocks: extract tables as structured arrays, forms as key-value pairs, query answers as named fields
- Support document formats: PDF, JPEG, PNG, TIFF
- For multi-page PDFs, handle pagination via `NextToken` in `get_document_analysis()`

**bedrock_extractor.py**: Structured data extraction using Bedrock:
- Accept raw Textract output (tables, forms, text)
- Call `bedrock_runtime.invoke_model()` with EXTRACTION_MODEL and DOCUMENT_EXTRACT_PROMPT
- Instruct model to produce clean structured JSON from noisy Textract output
- Validate extracted JSON schema before returning

**document_pipeline.py**: Orchestrate combined document processing:
- Run Textract for raw document extraction
- Run Bedrock for structured data parsing from Textract output
- Merge raw extraction and structured output into result JSON
- Write results to S3_OUTPUT_PATH under `documents/{filename}_results.json`

**sagemaker_processor.py**: SageMaker Processing job wrapper for batch mode:
- Create a `ScriptProcessor` with INSTANCE_TYPE and INSTANCE_COUNT
- Configure S3 input (S3_INPUT_PATH) and output (S3_OUTPUT_PATH) channels
- Submit `batch_processing_script.py` as the processing script
- Pass MODALITIES and model configuration as environment variables
- Monitor job status and return results location

**batch_processing_script.py**: Script that runs inside SageMaker Processing container:
- Read all media files from `/opt/ml/processing/input/`
- Route each file to the appropriate pipeline based on file extension and MODALITIES config
- Write all results to `/opt/ml/processing/output/`
- Log progress and errors to stdout (captured by CloudWatch)

**s3_event_handler.py**: Lambda handler for real-time S3 event processing:
- Triggered by S3 object creation events via EventBridge
- Detect modality from file extension (`.jpg/.png` → image, `.mp3/.wav` → audio, `.pdf/.docx` → document)
- Route to the appropriate pipeline
- Write results to S3_OUTPUT_PATH
- Send SNS notification on completion if NOTIFICATION_TOPIC_ARN configured
- On error, log to CloudWatch and optionally send SNS failure notification

**setup_event_rules.py**: Create EventBridge rules for S3 event triggers:
- Create EventBridge rule matching `Object Created` events on the input S3 bucket
- Filter by file extension prefixes matching enabled MODALITIES
- Target the `s3_event_handler` Lambda function
- Configure DLQ for failed event deliveries

**media_utils.py**: File validation and preprocessing utilities:
- `validate_file_size(s3_path, max_mb)`: Check file size against limit
- `resize_image(image_bytes, max_dimension)`: Resize large images maintaining aspect ratio using Pillow
- `split_audio(s3_path, segment_minutes)`: Split long audio files into segments
- `detect_modality(filename)`: Map file extension to modality type
- `encode_image_base64(image_bytes)`: Base64 encode image for Bedrock vision API

**s3_utils.py**: S3 read/write helpers:
- `read_file(bucket, key)`: Read file bytes from S3
- `write_json(bucket, key, data)`: Write JSON results to S3
- `list_files(bucket, prefix, extensions)`: List files matching extensions
- `get_file_metadata(bucket, key)`: Get file size and content type

**delete_resources.py**: Clean up Transcribe jobs, EventBridge rules, and SageMaker Processing jobs. Remove temporary S3 artifacts.

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Modalities:** Each modality pipeline is independent — generate only the pipelines for the MODALITIES specified. Image pipeline requires Bedrock Claude vision model access and Rekognition. Audio pipeline requires Transcribe and Bedrock. Document pipeline requires Textract and Bedrock.

**File Size Limits:** Bedrock Claude vision accepts images up to 20 MB (base64 encoded). Rekognition accepts images up to 15 MB via S3 or 5 MB via bytes. Transcribe accepts audio up to 2 GB / 4 hours. Textract accepts documents up to 500 pages (async API). Always validate file size before API calls and apply chunking or downsampling when limits are exceeded.

**Processing Modes:** In `realtime` mode, each file is processed individually via Lambda triggered by S3 events. Lambda timeout should be set to 900 seconds (15 min max) for large files. In `batch` mode, SageMaker Processing jobs handle bulk workloads with configurable instance types and counts.

**Security:** IAM roles must follow least privilege: separate roles for Lambda execution (Bedrock, Rekognition, Transcribe, Textract, S3 read/write) and SageMaker Processing (S3, Bedrock, CloudWatch). Use VPC endpoints for Bedrock, Rekognition, Transcribe, and Textract in private subnets. Encrypt all S3 output with SSE-S3 or SSE-KMS. Enable CloudTrail for all API calls.

**Cost:** Bedrock Claude vision is billed per input/output token — images consume tokens based on resolution. Rekognition is billed per image analyzed. Transcribe is billed per second of audio. Textract is billed per page. SageMaker Processing is billed per instance-hour. Use batch mode for large volumes to reduce per-file overhead. Monitor costs with CloudWatch metrics.

**Async Operations:** Transcribe and Textract are asynchronous — always poll with exponential backoff (initial 5s, max 60s, backoff factor 2). Never use tight polling loops. Include timeout limits (default 30 minutes) to prevent infinite polling.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- Lambda: `{PROJECT_NAME}-multimodal-processor-{ENV}`
- SageMaker Processing job: `{PROJECT_NAME}-multimodal-batch-{ENV}-{timestamp}`
- Transcribe job: `{PROJECT_NAME}-transcribe-{ENV}-{filename_hash}`
- EventBridge rule: `{PROJECT_NAME}-s3-media-trigger-{ENV}`
- S3 output prefix: `{S3_OUTPUT_PATH}/{modality}/{filename}_results.json`

---

## Code Scaffolding Hints

**Bedrock Claude vision — analyze image with base64 encoding:**
```python
import base64
import json
import boto3

bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)

def analyze_image(image_bytes, prompt, model_id=VISION_MODEL):
    """Analyze an image using Bedrock Claude vision."""
    # Encode image as base64
    image_base64 = base64.standard_b64encode(image_bytes).decode("utf-8")

    # Determine media type from image header bytes
    if image_bytes[:8] == b'\x89PNG\r\n\x1a\n':
        media_type = "image/png"
    elif image_bytes[:2] == b'\xff\xd8':
        media_type = "image/jpeg"
    elif image_bytes[:4] == b'GIF8':
        media_type = "image/gif"
    elif image_bytes[:4] == b'RIFF':
        media_type = "image/webp"
    else:
        media_type = "image/jpeg"  # default fallback

    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 4096,
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": media_type,
                            "data": image_base64,
                        },
                    },
                    {
                        "type": "text",
                        "text": prompt,
                    },
                ],
            }
        ],
    })

    response = bedrock_runtime.invoke_model(
        modelId=model_id,
        body=body,
        contentType="application/json",
        accept="application/json",
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]
```

**Rekognition — detect labels and text:**
```python
rekognition = boto3.client("rekognition", region_name=AWS_REGION)

def detect_labels(bucket, key, max_labels=20, min_confidence=80.0):
    """Detect objects and scenes in an image using Rekognition."""
    response = rekognition.detect_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MaxLabels=max_labels,
        MinConfidence=min_confidence,
        Features=["GENERAL_LABELS", "IMAGE_PROPERTIES"],
    )
    return [
        {
            "name": label["Name"],
            "confidence": label["Confidence"],
            "parents": [p["Name"] for p in label.get("Parents", [])],
            "categories": [c["Name"] for c in label.get("Categories", [])],
        }
        for label in response["Labels"]
    ]

def detect_text(bucket, key):
    """Detect text in an image using Rekognition."""
    response = rekognition.detect_text(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
    )
    return [
        {
            "text": detection["DetectedText"],
            "type": detection["Type"],
            "confidence": detection["Confidence"],
            "geometry": detection["Geometry"]["BoundingBox"],
        }
        for detection in response["TextDetections"]
        if detection["Type"] == "LINE"
    ]

def detect_moderation_labels(bucket, key, min_confidence=60.0):
    """Detect unsafe content in an image using Rekognition."""
    response = rekognition.detect_moderation_labels(
        Image={"S3Object": {"Bucket": bucket, "Name": key}},
        MinConfidence=min_confidence,
    )
    return [
        {
            "name": label["Name"],
            "confidence": label["Confidence"],
            "parent": label.get("ParentName", ""),
            "taxonomy_level": label.get("TaxonomyLevel", 0),
        }
        for label in response["ModerationLabels"]
    ]
```

**Transcribe — start transcription job with polling:**
```python
import time
import hashlib

transcribe = boto3.client("transcribe", region_name=AWS_REGION)

def start_transcription(s3_uri, output_bucket, output_key_prefix, language_code=TRANSCRIBE_LANGUAGE):
    """Start a Transcribe job and poll until complete."""
    # Generate unique job name from S3 URI
    job_name = f"{PROJECT_NAME}-transcribe-{ENV}-{hashlib.md5(s3_uri.encode()).hexdigest()[:12]}"

    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": s3_uri},
        MediaFormat=s3_uri.rsplit(".", 1)[-1],  # mp3, wav, mp4, flac, etc.
        LanguageCode=language_code,
        OutputBucketName=output_bucket,
        OutputKey=f"{output_key_prefix}/{job_name}.json",
        Settings={
            "ShowSpeakerLabels": True,
            "MaxSpeakerLabels": 10,
        },
    )

    # Poll with exponential backoff
    wait_time = 5
    max_wait = 60
    timeout = 1800  # 30 minutes
    elapsed = 0

    while elapsed < timeout:
        response = transcribe.get_transcription_job(TranscriptionJobName=job_name)
        status = response["TranscriptionJob"]["TranscriptionJobStatus"]

        if status == "COMPLETED":
            transcript_uri = response["TranscriptionJob"]["Transcript"]["TranscriptFileUri"]
            return {"status": "COMPLETED", "job_name": job_name, "transcript_uri": transcript_uri}
        elif status == "FAILED":
            reason = response["TranscriptionJob"].get("FailureReason", "Unknown")
            raise RuntimeError(f"Transcription failed: {reason}")

        time.sleep(wait_time)
        elapsed += wait_time
        wait_time = min(wait_time * 2, max_wait)

    raise TimeoutError(f"Transcription job {job_name} timed out after {timeout}s")
```

**Textract — start document analysis with polling:**
```python
textract = boto3.client("textract", region_name=AWS_REGION)

def analyze_document(bucket, key, feature_types=None, queries=None):
    """Start async Textract document analysis and poll until complete."""
    if feature_types is None:
        feature_types = ["TABLES", "FORMS"]

    start_params = {
        "DocumentLocation": {"S3Object": {"Bucket": bucket, "Name": key}},
        "FeatureTypes": feature_types,
    }

    if queries:
        start_params["QueriesConfig"] = {
            "Queries": [{"Text": q} for q in queries]
        }

    response = textract.start_document_analysis(**start_params)
    job_id = response["JobId"]

    # Poll with exponential backoff
    wait_time = 5
    max_wait = 60
    timeout = 1800
    elapsed = 0

    while elapsed < timeout:
        result = textract.get_document_analysis(JobId=job_id)
        status = result["JobStatus"]

        if status == "SUCCEEDED":
            # Collect all pages
            blocks = result.get("Blocks", [])
            next_token = result.get("NextToken")

            while next_token:
                result = textract.get_document_analysis(JobId=job_id, NextToken=next_token)
                blocks.extend(result.get("Blocks", []))
                next_token = result.get("NextToken")

            return parse_textract_blocks(blocks)

        elif status == "FAILED":
            raise RuntimeError(f"Textract analysis failed: {result.get('StatusMessage', 'Unknown')}")

        time.sleep(wait_time)
        elapsed += wait_time
        wait_time = min(wait_time * 2, max_wait)

    raise TimeoutError(f"Textract job {job_id} timed out after {timeout}s")


def parse_textract_blocks(blocks):
    """Parse Textract blocks into structured tables, forms, and text."""
    tables = []
    forms = {}
    text_lines = []
    query_answers = {}

    block_map = {b["Id"]: b for b in blocks}

    for block in blocks:
        if block["BlockType"] == "LINE":
            text_lines.append(block.get("Text", ""))

        elif block["BlockType"] == "KEY_VALUE_SET" and "KEY" in block.get("EntityTypes", []):
            key_text = ""
            value_text = ""
            for rel in block.get("Relationships", []):
                if rel["Type"] == "CHILD":
                    key_text = " ".join(
                        block_map[cid].get("Text", "") for cid in rel["Ids"] if cid in block_map
                    )
                if rel["Type"] == "VALUE":
                    for vid in rel["Ids"]:
                        val_block = block_map.get(vid, {})
                        for vrel in val_block.get("Relationships", []):
                            if vrel["Type"] == "CHILD":
                                value_text = " ".join(
                                    block_map[cid].get("Text", "")
                                    for cid in vrel["Ids"]
                                    if cid in block_map
                                )
            if key_text:
                forms[key_text.strip()] = value_text.strip()

        elif block["BlockType"] == "QUERY_RESULT":
            query_text = block.get("Query", {}).get("Text", "")
            answer_text = block.get("Text", "")
            if query_text:
                query_answers[query_text] = answer_text

    return {
        "text": "\n".join(text_lines),
        "forms": forms,
        "tables": tables,
        "query_answers": query_answers,
    }
```

**Image chunking — resize large images before Bedrock vision:**
```python
from io import BytesIO

def resize_image_if_needed(image_bytes, max_size_mb=20, max_dimension=2048):
    """Resize image if it exceeds size limits for Bedrock vision API."""
    size_mb = len(image_bytes) / (1024 * 1024)

    if size_mb <= max_size_mb:
        return image_bytes

    # Use Pillow to resize maintaining aspect ratio
    from PIL import Image

    img = Image.open(BytesIO(image_bytes))
    width, height = img.size

    # Calculate new dimensions maintaining aspect ratio
    if width > height:
        new_width = max_dimension
        new_height = int(height * (max_dimension / width))
    else:
        new_height = max_dimension
        new_width = int(width * (max_dimension / height))

    img = img.resize((new_width, new_height), Image.LANCZOS)

    # Save to bytes
    buffer = BytesIO()
    img_format = "JPEG" if img.mode == "RGB" else "PNG"
    img.save(buffer, format=img_format, quality=85)
    return buffer.getvalue()
```

**Transcript chunking — split long transcripts for Bedrock summarization:**
```python
def chunk_transcript(transcript_text, max_chunk_chars=80000, overlap_chars=2000):
    """Split long transcripts into overlapping chunks for summarization."""
    if len(transcript_text) <= max_chunk_chars:
        return [transcript_text]

    chunks = []
    start = 0
    while start < len(transcript_text):
        end = start + max_chunk_chars

        # Find sentence boundary near the end
        if end < len(transcript_text):
            boundary = transcript_text.rfind(". ", start + max_chunk_chars - 5000, end)
            if boundary > start:
                end = boundary + 1

        chunks.append(transcript_text[start:end])
        start = end - overlap_chars

    return chunks


def summarize_long_transcript(transcript_text, bedrock_runtime, model_id, prompt):
    """Summarize a long transcript using chunked summarization."""
    chunks = chunk_transcript(transcript_text)

    if len(chunks) == 1:
        # Single chunk — direct summarization
        return invoke_bedrock_text(bedrock_runtime, model_id, prompt, chunks[0])

    # Multi-chunk — summarize each, then combine
    chunk_summaries = []
    for i, chunk in enumerate(chunks):
        chunk_prompt = f"Summarize part {i + 1} of {len(chunks)} of this transcript:\n\n{chunk}"
        summary = invoke_bedrock_text(bedrock_runtime, model_id, chunk_prompt, "")
        chunk_summaries.append(summary)

    # Final combined summary
    combined = "\n\n---\n\n".join(chunk_summaries)
    final_prompt = f"Combine these partial summaries into a single coherent summary:\n\n{combined}"
    return invoke_bedrock_text(bedrock_runtime, model_id, final_prompt, "")


def invoke_bedrock_text(bedrock_runtime, model_id, prompt, context):
    """Invoke Bedrock model for text generation."""
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 4096,
        "messages": [
            {"role": "user", "content": f"{prompt}\n\n{context}" if context else prompt}
        ],
    })

    response = bedrock_runtime.invoke_model(
        modelId=model_id,
        body=body,
        contentType="application/json",
        accept="application/json",
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]
```

**SageMaker Processing job wrapper:**
```python
import sagemaker
from sagemaker.processing import ScriptProcessor, ProcessingInput, ProcessingOutput

def run_batch_processing(role_arn, input_s3, output_s3, modalities, instance_type=INSTANCE_TYPE, instance_count=INSTANCE_COUNT):
    """Run multimodal batch processing via SageMaker Processing."""
    processor = ScriptProcessor(
        role=role_arn,
        image_uri=f"763104351884.dkr.ecr.{AWS_REGION}.amazonaws.com/pytorch-inference:2.1.0-cpu-py310",
        command=["python3"],
        instance_type=instance_type,
        instance_count=instance_count,
        base_job_name=f"{PROJECT_NAME}-multimodal-batch-{ENV}",
        env={
            "MODALITIES": modalities,
            "VISION_MODEL": VISION_MODEL,
            "SUMMARIZATION_MODEL": SUMMARIZATION_MODEL,
            "EXTRACTION_MODEL": EXTRACTION_MODEL,
            "AWS_DEFAULT_REGION": AWS_REGION,
        },
    )

    processor.run(
        code="batch_processing_script.py",
        inputs=[
            ProcessingInput(
                source=input_s3,
                destination="/opt/ml/processing/input",
                input_name="media-input",
            )
        ],
        outputs=[
            ProcessingOutput(
                source="/opt/ml/processing/output",
                destination=output_s3,
                output_name="results-output",
            )
        ],
    )

    return processor.latest_processing_job.describe()
```

**S3 event trigger — Lambda handler:**
```python
import json
import os
import boto3

MODALITY_MAP = {
    ".jpg": "image", ".jpeg": "image", ".png": "image", ".gif": "image", ".webp": "image",
    ".mp3": "audio", ".wav": "audio", ".flac": "audio", ".ogg": "audio", ".mp4": "audio", ".m4a": "audio",
    ".pdf": "document", ".tiff": "document", ".tif": "document",
}

def lambda_handler(event, context):
    """Process S3 media uploads triggered by EventBridge."""
    detail = event.get("detail", {})
    bucket = detail.get("bucket", {}).get("name", "")
    key = detail.get("object", {}).get("key", "")

    if not bucket or not key:
        return {"statusCode": 400, "body": "Missing bucket or key"}

    # Detect modality from file extension
    ext = os.path.splitext(key)[1].lower()
    modality = MODALITY_MAP.get(ext)

    if not modality:
        return {"statusCode": 200, "body": f"Unsupported file type: {ext}"}

    enabled_modalities = os.environ.get("MODALITIES", "image,audio,document").split(",")
    if modality not in enabled_modalities:
        return {"statusCode": 200, "body": f"Modality {modality} not enabled"}

    try:
        if modality == "image":
            result = process_image(bucket, key)
        elif modality == "audio":
            result = process_audio(bucket, key)
        elif modality == "document":
            result = process_document(bucket, key)

        # Write results to output path
        output_bucket = os.environ.get("OUTPUT_BUCKET")
        output_prefix = os.environ.get("OUTPUT_PREFIX", "results")
        filename = os.path.splitext(os.path.basename(key))[0]
        output_key = f"{output_prefix}/{modality}/{filename}_results.json"

        s3 = boto3.client("s3")
        s3.put_object(
            Bucket=output_bucket,
            Key=output_key,
            Body=json.dumps(result, default=str),
            ContentType="application/json",
        )

        # Send SNS notification if configured
        topic_arn = os.environ.get("NOTIFICATION_TOPIC_ARN")
        if topic_arn:
            sns = boto3.client("sns")
            sns.publish(
                TopicArn=topic_arn,
                Subject=f"Multimodal Processing Complete: {modality}/{filename}",
                Message=json.dumps({
                    "status": "SUCCESS",
                    "modality": modality,
                    "input": f"s3://{bucket}/{key}",
                    "output": f"s3://{output_bucket}/{output_key}",
                }),
            )

        return {"statusCode": 200, "body": json.dumps(result, default=str)}

    except Exception as e:
        # Log error and optionally notify
        print(f"Error processing {modality} file s3://{bucket}/{key}: {e}")
        topic_arn = os.environ.get("NOTIFICATION_TOPIC_ARN")
        if topic_arn:
            sns = boto3.client("sns")
            sns.publish(
                TopicArn=topic_arn,
                Subject=f"Multimodal Processing FAILED: {modality}/{os.path.basename(key)}",
                Message=json.dumps({"status": "FAILED", "error": str(e), "input": f"s3://{bucket}/{key}"}),
            )
        raise
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM roles for Bedrock, Rekognition, Transcribe, Textract, SageMaker, S3, Lambda execution
- **Upstream**: `devops/02` → VPC configuration with endpoints for Bedrock, Rekognition, Transcribe, Textract in private subnets
- **Downstream**: `data/01` → Glue ETL for batch post-processing of extracted structured data into ML features
- **Downstream**: `mlops/07` → Feature Store ingestion of extracted image labels, transcript embeddings, document fields
- **Downstream**: `devops/10` → OpenTelemetry tracing for end-to-end multimodal pipeline observability
