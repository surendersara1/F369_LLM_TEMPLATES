<!-- Template Version: 1.0 | boto3: 1.35+ | redis: 5.0+ | numpy: 1.26+ -->

# Template 18 — Prompt Caching Patterns

## Purpose
Generate production-ready prompt caching infrastructure to reduce LLM inference latency and cost: Bedrock Converse API native prompt caching with `cachePoint` markers for system prompt and context caching, ElastiCache Redis semantic caching with embedding vectors and configurable similarity thresholds, DynamoDB exact-match caching with TTL-based expiration, cost reduction analysis comparing cached vs uncached invocations with CloudWatch metrics, and cache invalidation strategies (TTL-based, manual API, model-version-based).

---

## Role Definition

You are an expert AWS AI engineer specializing in LLM inference optimization and caching with expertise in:
- Bedrock Converse API: `converse()` with `cachePoint` content blocks for native prompt caching, cache token usage metrics (`cacheReadInputTokens`, `cacheWriteInputTokens`), TTL configuration
- Semantic caching: embedding vector generation using Bedrock Titan Embeddings, cosine similarity search, configurable similarity thresholds for cache hit/miss decisions
- ElastiCache (Redis OSS): vector similarity search with Redis Search module, HNSW indexing, `FT.CREATE` / `FT.SEARCH` for vector queries, connection pooling
- DynamoDB: exact-match key-value caching with TTL attribute for automatic expiration, conditional writes, batch operations
- Cost optimization: CloudWatch custom metrics for cache hit rate, cost savings tracking, per-model cost-per-invocation analysis
- Cache invalidation: TTL-based expiration, manual invalidation via API, model-version-based invalidation when underlying models change
- CloudWatch: custom metric publishing, metric math for cost analysis, dashboards for cache performance monitoring

Generate complete, production-deployable code.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

CACHE_STRATEGY:         [REQUIRED - bedrock_native | semantic | exact_match | all]
                        bedrock_native: Bedrock Converse API cachePoint markers for
                                        system prompt and context caching (no external cache)
                        semantic: ElastiCache Redis with embedding vectors for
                                  semantically similar query matching
                        exact_match: DynamoDB TTL-based cache for deterministic
                                     prompt-response pairs
                        all: generate all three caching strategies

MODEL_ID:               [REQUIRED - Bedrock model ID for inference]
                        Must support prompt caching for bedrock_native strategy.
                        Options (verify current availability):
                        - anthropic.claude-3-5-sonnet-20241022-v2:0 (min 1,024 tokens)
                        - anthropic.claude-3-7-sonnet-20250219-v1:0 (min 1,024 tokens)
                        - anthropic.claude-3-5-haiku-20241022-v1:0 (min 2,048 tokens)
                        - amazon.nova-pro-v1:0 (min 1,000 tokens)
                        - amazon.nova-lite-v1:0 (min 1,000 tokens)

EMBEDDING_MODEL_ID:     [OPTIONAL: amazon.titan-embed-text-v2:0]
                        Model for generating query embeddings (semantic strategy).
                        Options:
                        - amazon.titan-embed-text-v2:0 (1024 dimensions)
                        - amazon.titan-embed-text-v1 (1536 dimensions)
                        - cohere.embed-english-v3 (1024 dimensions)

ELASTICACHE_CLUSTER:    [OPTIONAL - ElastiCache Redis cluster endpoint]
                        Required when CACHE_STRATEGY is semantic or all.
                        Format: redis://hostname:6379

DYNAMODB_TABLE:         [OPTIONAL: {PROJECT_NAME}-prompt-cache-{ENV}]
                        DynamoDB table name for exact-match cache.

SIMILARITY_THRESHOLD:   [OPTIONAL: 0.92]
                        Cosine similarity threshold for semantic cache hits.
                        Range: 0.0 to 1.0. Higher = stricter matching.
                        Recommended: 0.90-0.95 for factual queries,
                                     0.85-0.90 for conversational queries.

TTL_SECONDS:            [OPTIONAL: 3600]
                        Cache entry time-to-live in seconds.
                        Default: 3600 (1 hour).
                        Bedrock native cache TTL is managed by the service (5 min default).

EMBEDDING_DIMENSIONS:   [OPTIONAL: 1024]
                        Embedding vector dimensions. Must match EMBEDDING_MODEL_ID output.

SYSTEM_PROMPT:          [OPTIONAL - static system prompt to cache with bedrock_native strategy]
                        Long system prompts (1K+ tokens) benefit most from caching.

MAX_CACHE_ENTRIES:       [OPTIONAL: 10000]
                        Maximum number of entries in semantic/exact-match cache.
```

---

## Task

Generate complete prompt caching infrastructure:

```
{PROJECT_NAME}-prompt-caching/
├── config.py                              # Central configuration
├── bedrock_native/
│   ├── cached_converse.py                 # Converse API with cachePoint markers
│   ├── cache_metrics.py                   # Parse cache token usage from responses
│   └── multi_turn_cache.py                # Multi-turn conversation with cached context
├── semantic_cache/
│   ├── embedding_client.py                # Bedrock Titan embedding generation
│   ├── redis_vector_store.py              # Redis vector similarity search
│   ├── semantic_cache_client.py           # Unified semantic cache get/set
│   └── create_redis_index.py             # Create Redis Search vector index
├── exact_match_cache/
│   ├── dynamodb_cache.py                  # DynamoDB get/put with TTL
│   ├── create_cache_table.py             # Create DynamoDB table with TTL
│   └── cache_key_utils.py                # Deterministic cache key generation
├── invalidation/
│   ├── ttl_manager.py                     # TTL-based expiration management
│   ├── manual_invalidation.py             # API-driven cache invalidation
│   └── model_version_invalidation.py      # Invalidate on model version change
├── cost_analysis/
│   ├── cost_tracker.py                    # Track cached vs uncached costs
│   ├── cloudwatch_metrics.py              # Publish cache metrics to CloudWatch
│   └── savings_report.py                  # Generate cost savings report
├── unified_cache.py                       # Unified cache interface (all strategies)
├── run_setup.py                           # CLI orchestrator
└── requirements.txt
```

**config.py**: Central configuration dataclass with all parameters. Load from environment variables or CLI args. Validate required fields. Construct resource names using `{PROJECT_NAME}-{component}-{ENV}` convention. Validate MODEL_ID supports prompt caching for bedrock_native strategy. Parse SIMILARITY_THRESHOLD as float.

**cached_converse.py**: Invoke Bedrock Converse API with native prompt caching:
- Build message payload with `cachePoint` content blocks after system prompt and static context
- Call `bedrock_runtime.converse()` with `system` containing text + `cachePoint`, and `messages` with user content
- Parse response `usage` for `cacheReadInputTokens`, `cacheWriteInputTokens`, `inputTokens`, `outputTokens`
- Log cache hit/miss status based on `cacheReadInputTokens > 0`
- Return response text and cache metrics

**cache_metrics.py**: Parse and aggregate cache token usage from Converse API responses:
- Extract `cacheReadInputTokens`, `cacheWriteInputTokens` from response `usage`
- Compute cache hit rate over a sliding window of requests
- Compute estimated cost savings based on cached vs standard input token pricing
- Publish metrics to CloudWatch

**multi_turn_cache.py**: Multi-turn conversation with cached system prompt and context:
- Maintain conversation history with `cachePoint` markers after the system prompt and after large context blocks
- On each turn, append new user message and re-send with cached prefix
- Track cumulative cache savings across the conversation
- Support up to 4 cache checkpoints per request (model limit)

**embedding_client.py**: Generate embedding vectors using Bedrock:
- Call `bedrock_runtime.invoke_model()` with Titan Embeddings model
- Accept text input, return numpy array of embedding vector
- Normalize vectors for cosine similarity computation
- Support batch embedding generation

**redis_vector_store.py**: Redis vector similarity search:
- Connect to ElastiCache Redis cluster with connection pooling
- Store cache entries as Redis hashes with embedding vector, response text, metadata, and TTL
- Search for similar vectors using `FT.SEARCH` with KNN query
- Return top-K results with similarity scores
- Filter results by SIMILARITY_THRESHOLD

**semantic_cache_client.py**: Unified semantic cache interface:
- `get(query)`: Generate embedding → search Redis → return cached response if similarity ≥ threshold
- `set(query, response, metadata)`: Generate embedding → store in Redis with TTL
- `invalidate(query)`: Remove specific cache entry
- `clear()`: Flush all cache entries
- On cache miss, invoke Bedrock model, store response, and return

**create_redis_index.py**: Create Redis Search vector index:
- Create index with `FT.CREATE` using HNSW algorithm for approximate nearest neighbor search
- Define schema: embedding vector field (VECTOR type), response text (TEXT), model_id (TAG), timestamp (NUMERIC), ttl (NUMERIC)
- Configure HNSW parameters: M=16, EF_CONSTRUCTION=200 for balanced speed/recall

**dynamodb_cache.py**: DynamoDB exact-match cache:
- `get(cache_key)`: Get item by cache key, check TTL validity
- `set(cache_key, response, ttl_seconds)`: Put item with TTL attribute
- `delete(cache_key)`: Delete specific cache entry
- `batch_get(cache_keys)`: Batch get for multiple keys
- Use conditional writes to prevent race conditions

**create_cache_table.py**: Create DynamoDB table `{PROJECT_NAME}-prompt-cache-{ENV}`:
- Partition key: `cache_key` (String)
- TTL attribute: `ttl` (Number — epoch seconds)
- On-demand billing mode
- Encryption at rest with AWS managed key
- Tags with Project and Environment

**cache_key_utils.py**: Deterministic cache key generation:
- Hash prompt text using SHA-256 for exact-match keys
- Include model_id in key to prevent cross-model cache hits
- Include optional namespace prefix for multi-tenant isolation
- Normalize whitespace and casing before hashing

**ttl_manager.py**: TTL-based expiration management:
- Configure per-strategy TTL defaults (Bedrock native: service-managed 5 min, semantic: configurable, exact-match: configurable)
- Support TTL override per cache entry
- Provide TTL refresh on cache hit (extend expiration)

**manual_invalidation.py**: API-driven cache invalidation:
- `invalidate_by_key(cache_key)`: Remove specific entry from DynamoDB or Redis
- `invalidate_by_prefix(prefix)`: Remove all entries matching a key prefix
- `invalidate_by_model(model_id)`: Remove all entries for a specific model
- `invalidate_all()`: Flush entire cache (with confirmation safeguard)

**model_version_invalidation.py**: Invalidate cache when model version changes:
- Store current model version in SSM Parameter Store
- On each cache set, tag entry with model version
- On cache get, compare entry model version with current version
- If mismatch, treat as cache miss and invalidate stale entry
- Support EventBridge rule trigger on model version update

**cost_tracker.py**: Track cached vs uncached invocation costs:
- Record per-invocation metrics: cache hit/miss, input tokens, cached tokens, output tokens, estimated cost
- Compute running totals: total invocations, cache hit rate, total cost, total savings
- Store tracking data in DynamoDB or local aggregation
- Support per-model and per-strategy cost breakdown

**cloudwatch_metrics.py**: Publish cache performance metrics to CloudWatch:
- Namespace: `{PROJECT_NAME}/PromptCache`
- Metrics: `CacheHitRate`, `CacheMissRate`, `CacheReadTokens`, `CacheWriteTokens`, `EstimatedSavingsUSD`, `CacheLatencyMs`, `InvocationCount`
- Dimensions: `Strategy` (bedrock_native/semantic/exact_match), `ModelId`, `Environment`
- Publish at 1-minute resolution

**savings_report.py**: Generate cost savings report:
- Query CloudWatch metrics for a configurable time range
- Compute total cost with caching vs estimated cost without caching
- Compute savings percentage and absolute dollar savings
- Break down by strategy, model, and time period
- Output as JSON and optional CSV

**unified_cache.py**: Unified cache interface that chains strategies:
- Check exact-match cache first (fastest, lowest cost)
- If miss, check semantic cache (medium latency)
- If miss, invoke Bedrock with native caching (cachePoint markers)
- Store response in all applicable caches
- Configurable strategy priority order
- Single `invoke(prompt, system_prompt, context)` method

---

## Output Format

Output ALL files with headers: `### FILE: [path]`

---

## Requirements & Constraints

**Bedrock Native Caching:** Cache checkpoints require a minimum token count per model (1,024 for Claude 3.5 Sonnet v2 / Claude 3.7 Sonnet, 2,048 for Claude 3.5 Haiku, 1,000 for Nova models). Maximum 4 cache checkpoints per request. Cache TTL is 5 minutes by default (resets on each cache hit). Place `cachePoint` markers after static content (system prompt, large context documents) — not after dynamic user queries. Cache is per-session and per-model; changing the prefix invalidates the cache.

**Semantic Caching:** ElastiCache Redis must have the Redis Search module enabled for vector similarity search. HNSW index provides approximate nearest neighbor search — tune M and EF_CONSTRUCTION for your latency/recall tradeoff. Embedding generation adds latency (~50-100ms per query) — only use semantic caching when the cost savings from cache hits outweigh the embedding overhead. Similarity threshold tuning is critical: too low causes incorrect cache hits, too high causes excessive misses.

**Exact-Match Caching:** DynamoDB TTL deletion is eventually consistent (items may persist up to 48 hours after TTL expiration). Use conditional checks on TTL in application code for immediate expiration enforcement. Cache keys must be deterministic — normalize input before hashing. Maximum item size is 400KB — truncate or compress large responses.

**Cache Invalidation:** Always invalidate cache when the underlying model changes (model version update, fine-tuned model redeployment). TTL-based invalidation is the simplest and most reliable strategy. Manual invalidation should be exposed as an API for operational use. Model-version-based invalidation requires tracking the current model version in SSM Parameter Store.

**Security:** Encrypt ElastiCache Redis in transit (TLS) and at rest. DynamoDB table must use encryption at rest. Do not cache responses containing PII unless the cache storage meets the same compliance requirements as the source data. Use VPC endpoints for Bedrock, ElastiCache, and DynamoDB API calls in private subnets.

**Cost:** Bedrock native caching charges reduced rates for cached input tokens (varies by model). Semantic caching adds ElastiCache hosting cost + Bedrock embedding invocation cost per query. Exact-match caching adds DynamoDB read/write costs. Monitor cache hit rate — caching is only cost-effective above ~30% hit rate for semantic caching and ~10% for exact-match caching.

**Naming:** All resources follow `{PROJECT_NAME}-{component}-{ENV}` convention:
- DynamoDB table: `{PROJECT_NAME}-prompt-cache-{ENV}`
- CloudWatch namespace: `{PROJECT_NAME}/PromptCache`
- SSM parameter: `/{PROJECT_NAME}/{ENV}/model-version`
- Redis index: `{PROJECT_NAME}-{ENV}-vector-idx`

---

## Code Scaffolding Hints


**Bedrock Converse API with cachePoint — system prompt caching:**
```python
import boto3
import json
import logging

bedrock_runtime = boto3.client("bedrock-runtime", region_name=AWS_REGION)
logger = logging.getLogger(__name__)

def converse_with_cache(model_id: str, system_prompt: str, user_message: str, conversation_history: list = None) -> dict:
    """Invoke Bedrock Converse API with native prompt caching via cachePoint markers."""

    # System prompt with cachePoint — caches the system instructions
    system = [
        {"text": system_prompt},
        {"cachePoint": {"type": "default"}},
    ]

    # Build messages with optional cached context
    messages = []
    if conversation_history:
        messages.extend(conversation_history)

    messages.append({
        "role": "user",
        "content": [{"text": user_message}],
    })

    response = bedrock_runtime.converse(
        modelId=model_id,
        system=system,
        messages=messages,
        inferenceConfig={
            "maxTokens": 1024,
            "temperature": 0.7,
            "topP": 0.9,
        },
    )

    # Extract cache metrics from response usage
    usage = response.get("usage", {})
    cache_read_tokens = usage.get("cacheReadInputTokens", 0)
    cache_write_tokens = usage.get("cacheWriteInputTokens", 0)
    input_tokens = usage.get("inputTokens", 0)
    output_tokens = usage.get("outputTokens", 0)

    cache_hit = cache_read_tokens > 0
    logger.info(
        "Cache %s | read=%d write=%d input=%d output=%d",
        "HIT" if cache_hit else "MISS",
        cache_read_tokens, cache_write_tokens, input_tokens, output_tokens,
    )

    # Extract response text
    output_message = response["output"]["message"]
    response_text = "".join(
        block.get("text", "") for block in output_message["content"]
    )

    return {
        "response": response_text,
        "cache_hit": cache_hit,
        "cache_read_tokens": cache_read_tokens,
        "cache_write_tokens": cache_write_tokens,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "stop_reason": response.get("stopReason", ""),
    }
```

**Converse API with cachePoint — large context document caching:**
```python
def converse_with_context_cache(model_id: str, system_prompt: str, context_document: str, user_query: str) -> dict:
    """Cache both system prompt and a large context document using two cachePoint markers."""

    system = [
        {"text": system_prompt},
        {"cachePoint": {"type": "default"}},  # Checkpoint 1: cache system prompt
    ]

    messages = [
        {
            "role": "user",
            "content": [
                {"text": f"Reference document:\n\n{context_document}"},
                {"cachePoint": {"type": "default"}},  # Checkpoint 2: cache context document
                {"text": user_query},
            ],
        }
    ]

    response = bedrock_runtime.converse(
        modelId=model_id,
        system=system,
        messages=messages,
        inferenceConfig={"maxTokens": 1024, "temperature": 0.7},
    )

    usage = response.get("usage", {})
    return {
        "response": response["output"]["message"]["content"][0]["text"],
        "cache_read_tokens": usage.get("cacheReadInputTokens", 0),
        "cache_write_tokens": usage.get("cacheWriteInputTokens", 0),
        "input_tokens": usage.get("inputTokens", 0),
        "output_tokens": usage.get("outputTokens", 0),
    }
```

**Generate embedding vectors using Bedrock Titan:**
```python
import json
import numpy as np

def generate_embedding(text: str, model_id: str = "amazon.titan-embed-text-v2:0") -> np.ndarray:
    """Generate embedding vector using Bedrock Titan Embeddings."""
    response = bedrock_runtime.invoke_model(
        modelId=model_id,
        body=json.dumps({"inputText": text}),
        contentType="application/json",
    )
    result = json.loads(response["body"].read())
    embedding = np.array(result["embedding"], dtype=np.float32)

    # Normalize for cosine similarity
    norm = np.linalg.norm(embedding)
    if norm > 0:
        embedding = embedding / norm

    return embedding
```

**Redis vector similarity search — create index and query:**
```python
import redis
import numpy as np
import json
import time

redis_client = redis.Redis.from_url(ELASTICACHE_CLUSTER, decode_responses=False)

def create_vector_index(index_name: str, vector_dim: int = 1024):
    """Create Redis Search vector index for semantic caching."""
    try:
        redis_client.execute_command("FT.DROPINDEX", index_name)
    except redis.ResponseError:
        pass  # Index does not exist

    redis_client.execute_command(
        "FT.CREATE", index_name,
        "ON", "HASH",
        "PREFIX", "1", f"cache:",
        "SCHEMA",
        "embedding", "VECTOR", "HNSW", "6",
            "TYPE", "FLOAT32",
            "DIM", str(vector_dim),
            "DISTANCE_METRIC", "COSINE",
        "response", "TEXT",
        "model_id", "TAG",
        "model_version", "TAG",
        "timestamp", "NUMERIC",
        "ttl", "NUMERIC",
    )


def search_similar(query_embedding: np.ndarray, index_name: str, top_k: int = 1, threshold: float = 0.92) -> list:
    """Search for semantically similar cached responses."""
    query_blob = query_embedding.astype(np.float32).tobytes()

    results = redis_client.execute_command(
        "FT.SEARCH", index_name,
        f"(*)=>[KNN {top_k} @embedding $query_vec AS score]",
        "PARAMS", "2", "query_vec", query_blob,
        "SORTBY", "score",
        "RETURN", "3", "response", "score", "model_id",
        "DIALECT", "2",
    )

    matches = []
    # Results format: [total_count, key1, [field1, val1, ...], key2, ...]
    if results[0] > 0:
        for i in range(1, len(results), 2):
            fields = dict(zip(results[i + 1][::2], results[i + 1][1::2]))
            similarity = 1.0 - float(fields[b"score"])  # COSINE distance → similarity
            if similarity >= threshold:
                matches.append({
                    "key": results[i].decode(),
                    "response": fields[b"response"].decode(),
                    "similarity": similarity,
                })

    return matches


def store_in_cache(cache_key: str, query_embedding: np.ndarray, response: str, model_id: str, model_version: str, ttl_seconds: int = 3600):
    """Store a prompt-response pair in the semantic cache."""
    redis_client.hset(f"cache:{cache_key}", mapping={
        "embedding": query_embedding.astype(np.float32).tobytes(),
        "response": response,
        "model_id": model_id,
        "model_version": model_version,
        "timestamp": str(int(time.time())),
        "ttl": str(int(time.time()) + ttl_seconds),
    })
    redis_client.expire(f"cache:{cache_key}", ttl_seconds)
```

**DynamoDB exact-match cache with TTL:**
```python
import boto3
import hashlib
import json
import time

dynamodb = boto3.resource("dynamodb", region_name=AWS_REGION)
table = dynamodb.Table(f"{PROJECT_NAME}-prompt-cache-{ENV}")

def generate_cache_key(prompt: str, model_id: str, system_prompt: str = "") -> str:
    """Generate deterministic cache key from prompt + model."""
    normalized = f"{model_id}::{system_prompt.strip()}::{prompt.strip()}"
    return hashlib.sha256(normalized.encode()).hexdigest()


def cache_get(cache_key: str) -> dict | None:
    """Get cached response from DynamoDB. Returns None on miss or expired entry."""
    response = table.get_item(Key={"cache_key": cache_key})
    item = response.get("Item")

    if not item:
        return None

    # Check TTL in application code (DynamoDB TTL deletion is eventually consistent)
    if item.get("ttl", 0) <= int(time.time()):
        return None

    return {
        "response": item["response"],
        "model_id": item.get("model_id", ""),
        "cached_at": item.get("cached_at", 0),
    }


def cache_set(cache_key: str, prompt: str, response: str, model_id: str, model_version: str, ttl_seconds: int = 3600):
    """Store prompt-response pair in DynamoDB with TTL."""
    now = int(time.time())
    table.put_item(
        Item={
            "cache_key": cache_key,
            "prompt_hash": cache_key,
            "response": response,
            "model_id": model_id,
            "model_version": model_version,
            "cached_at": now,
            "ttl": now + ttl_seconds,
        }
    )


def cache_delete(cache_key: str):
    """Delete a specific cache entry."""
    table.delete_item(Key={"cache_key": cache_key})


def invalidate_by_model_version(model_id: str, current_version: str):
    """Scan and delete cache entries with outdated model versions."""
    response = table.scan(
        FilterExpression="model_id = :mid AND model_version <> :ver",
        ExpressionAttributeValues={
            ":mid": model_id,
            ":ver": current_version,
        },
        ProjectionExpression="cache_key",
    )
    with table.batch_writer() as batch:
        for item in response.get("Items", []):
            batch.delete_item(Key={"cache_key": item["cache_key"]})
```

**Create DynamoDB cache table:**
```python
dynamodb_client = boto3.client("dynamodb", region_name=AWS_REGION)

def create_cache_table(table_name: str):
    """Create DynamoDB table for exact-match prompt caching."""
    dynamodb_client.create_table(
        TableName=table_name,
        KeySchema=[
            {"AttributeName": "cache_key", "KeyType": "HASH"},
        ],
        AttributeDefinitions=[
            {"AttributeName": "cache_key", "AttributeType": "S"},
        ],
        BillingMode="PAY_PER_REQUEST",
        SSESpecification={"Enabled": True},
        Tags=[
            {"Key": "Project", "Value": PROJECT_NAME},
            {"Key": "Environment", "Value": ENV},
        ],
    )

    # Wait for table to be active
    waiter = dynamodb_client.get_waiter("table_exists")
    waiter.wait(TableName=table_name)

    # Enable TTL
    dynamodb_client.update_time_to_live(
        TableName=table_name,
        TimeToLiveSpecification={
            "Enabled": True,
            "AttributeName": "ttl",
        },
    )
```

**CloudWatch cost metrics publishing:**
```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name=AWS_REGION)

def publish_cache_metrics(strategy: str, model_id: str, cache_hit: bool, input_tokens: int, cached_tokens: int, output_tokens: int, latency_ms: float):
    """Publish cache performance metrics to CloudWatch."""
    dimensions = [
        {"Name": "Strategy", "Value": strategy},
        {"Name": "ModelId", "Value": model_id},
        {"Name": "Environment", "Value": ENV},
    ]

    metrics = [
        {
            "MetricName": "CacheHit",
            "Dimensions": dimensions,
            "Value": 1.0 if cache_hit else 0.0,
            "Unit": "Count",
        },
        {
            "MetricName": "InvocationCount",
            "Dimensions": dimensions,
            "Value": 1.0,
            "Unit": "Count",
        },
        {
            "MetricName": "CacheLatencyMs",
            "Dimensions": dimensions,
            "Value": latency_ms,
            "Unit": "Milliseconds",
        },
        {
            "MetricName": "CachedInputTokens",
            "Dimensions": dimensions,
            "Value": float(cached_tokens),
            "Unit": "Count",
        },
    ]

    # Estimate cost savings (approximate — varies by model)
    # Standard input token cost vs cached input token cost
    if cache_hit and cached_tokens > 0:
        # Example: Claude 3.5 Sonnet standard=$0.003/1K, cached=$0.0003/1K (90% savings)
        standard_cost = (cached_tokens / 1000) * 0.003
        cached_cost = (cached_tokens / 1000) * 0.0003
        savings = standard_cost - cached_cost
        metrics.append({
            "MetricName": "EstimatedSavingsUSD",
            "Dimensions": dimensions,
            "Value": savings,
            "Unit": "None",
        })

    cloudwatch.put_metric_data(
        Namespace=f"{PROJECT_NAME}/PromptCache",
        MetricData=metrics,
    )


def get_cache_hit_rate(strategy: str, period_hours: int = 24) -> dict:
    """Query CloudWatch for cache hit rate over a time period."""
    from datetime import datetime, timedelta

    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=period_hours)

    response = cloudwatch.get_metric_statistics(
        Namespace=f"{PROJECT_NAME}/PromptCache",
        MetricName="CacheHit",
        Dimensions=[
            {"Name": "Strategy", "Value": strategy},
            {"Name": "Environment", "Value": ENV},
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=3600,
        Statistics=["Sum", "SampleCount"],
    )

    total_hits = sum(dp["Sum"] for dp in response.get("Datapoints", []))
    total_invocations = sum(dp["SampleCount"] for dp in response.get("Datapoints", []))

    return {
        "hit_rate": total_hits / total_invocations if total_invocations > 0 else 0.0,
        "total_hits": int(total_hits),
        "total_invocations": int(total_invocations),
        "period_hours": period_hours,
    }
```

**Unified cache interface — chained strategy lookup:**
```python
import time
import logging

logger = logging.getLogger(__name__)

class UnifiedPromptCache:
    """Unified cache that chains exact-match → semantic → Bedrock native caching."""

    def __init__(self, config):
        self.config = config
        self.exact_cache = DynamoDBCache(config) if config.cache_strategy in ("exact_match", "all") else None
        self.semantic_cache = SemanticCache(config) if config.cache_strategy in ("semantic", "all") else None
        self.use_bedrock_native = config.cache_strategy in ("bedrock_native", "all")

    def invoke(self, prompt: str, system_prompt: str = "", context: str = "") -> dict:
        """Invoke with chained cache lookup: exact → semantic → Bedrock native."""
        start = time.time()

        # Layer 1: Exact-match cache (fastest)
        if self.exact_cache:
            cache_key = generate_cache_key(prompt, self.config.model_id, system_prompt)
            cached = self.exact_cache.get(cache_key)
            if cached:
                latency = (time.time() - start) * 1000
                publish_cache_metrics("exact_match", self.config.model_id, True, 0, 0, 0, latency)
                logger.info("Exact-match cache HIT (%.1fms)", latency)
                return {"response": cached["response"], "source": "exact_match", "latency_ms": latency}

        # Layer 2: Semantic cache (medium latency)
        if self.semantic_cache:
            matches = self.semantic_cache.search(prompt)
            if matches:
                latency = (time.time() - start) * 1000
                publish_cache_metrics("semantic", self.config.model_id, True, 0, 0, 0, latency)
                logger.info("Semantic cache HIT (similarity=%.3f, %.1fms)", matches[0]["similarity"], latency)
                return {"response": matches[0]["response"], "source": "semantic", "latency_ms": latency}

        # Layer 3: Bedrock invocation (with native caching if enabled)
        if self.use_bedrock_native and (system_prompt or context):
            result = converse_with_context_cache(
                self.config.model_id, system_prompt, context, prompt,
            )
        else:
            result = converse_with_cache(
                self.config.model_id, system_prompt or "You are a helpful assistant.", prompt,
            )

        latency = (time.time() - start) * 1000
        response_text = result["response"]

        # Store in caches for future hits
        if self.exact_cache:
            cache_key = generate_cache_key(prompt, self.config.model_id, system_prompt)
            self.exact_cache.set(cache_key, prompt, response_text, self.config.model_id, "v1", self.config.ttl_seconds)

        if self.semantic_cache:
            self.semantic_cache.store(prompt, response_text, self.config.model_id, "v1", self.config.ttl_seconds)

        publish_cache_metrics(
            "bedrock_native" if result.get("cache_hit") else "miss",
            self.config.model_id,
            result.get("cache_hit", False),
            result.get("input_tokens", 0),
            result.get("cache_read_tokens", 0),
            result.get("output_tokens", 0),
            latency,
        )

        return {"response": response_text, "source": "bedrock", "cache_hit": result.get("cache_hit", False), "latency_ms": latency}
```

**Cost savings report generation:**
```python
def generate_savings_report(period_hours: int = 720) -> dict:
    """Generate cost savings report comparing cached vs uncached invocations."""
    strategies = ["bedrock_native", "semantic", "exact_match"]
    report = {"period_hours": period_hours, "strategies": {}}

    for strategy in strategies:
        stats = get_cache_hit_rate(strategy, period_hours)
        if stats["total_invocations"] == 0:
            continue

        report["strategies"][strategy] = {
            "total_invocations": stats["total_invocations"],
            "cache_hits": stats["total_hits"],
            "cache_misses": stats["total_invocations"] - stats["total_hits"],
            "hit_rate": f"{stats['hit_rate']:.1%}",
        }

    # Query cumulative savings metric
    from datetime import datetime, timedelta
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=period_hours)

    savings_response = cloudwatch.get_metric_statistics(
        Namespace=f"{PROJECT_NAME}/PromptCache",
        MetricName="EstimatedSavingsUSD",
        Dimensions=[{"Name": "Environment", "Value": ENV}],
        StartTime=start_time,
        EndTime=end_time,
        Period=period_hours * 3600,
        Statistics=["Sum"],
    )

    total_savings = sum(dp["Sum"] for dp in savings_response.get("Datapoints", []))
    report["total_estimated_savings_usd"] = round(total_savings, 4)

    return report
```

---

## Integration Points

- **Upstream**: `mlops/04` → RAG pipeline provides large context documents ideal for Bedrock native caching (cache the retrieved context across follow-up queries)
- **Upstream**: `mlops/03` → LLM inference deployment provides the model endpoints and model IDs used for cached invocations
- **Upstream**: `devops/04` → IAM roles for Bedrock Runtime, ElastiCache, DynamoDB, and CloudWatch access
- **Downstream**: `devops/11` → Custom CloudWatch metrics from cache performance feed into model quality dashboards
- **Downstream**: `finops/01` → Cost savings metrics from caching feed into FinOps cost allocation and tracking
