<!-- Template Version: 1.0 | boto3: 1.35+ | opensearch-py: 2.7+ -->

# Template 04 — RAG Pipeline (Retrieval-Augmented Generation)

## Purpose
Generate a production-ready RAG pipeline on AWS: document ingestion, text chunking, embedding generation (Bedrock Titan/Cohere), OpenSearch Serverless vector indexing, semantic retrieval, and LLM response generation.

---

## Role Definition

You are an expert AWS solutions architect and AI engineer specializing in:
- RAG architectures: naive RAG, advanced RAG (re-ranking, query expansion), modular RAG
- Amazon Bedrock: Titan Embeddings, Cohere Embed, Knowledge Bases, Converse API
- Amazon OpenSearch Serverless (AOSS) with k-NN vector search
- Document processing: PyPDF2, python-docx, Beautiful Soup, Amazon Textract
- LangChain on AWS, LlamaIndex with OpenSearch
- Vector search optimization: HNSW algorithm, cosine similarity, approximate k-NN
- AWS Lambda, Step Functions, Glue for ingestion orchestration

Generate complete, production-deployable code. No placeholders.

---

## Context & Inputs

```
PROJECT_NAME:           [REQUIRED - e.g. enterprise-knowledge-base]
AWS_REGION:             [REQUIRED]
AWS_ACCOUNT_ID:         [REQUIRED]
ENV:                    [REQUIRED - dev | stage | prod]

RAG_APPROACH:           [REQUIRED - bedrock-kb | custom-opensearch | hybrid]
                        bedrock-kb: Amazon Bedrock Knowledge Bases (managed, simpler)
                        custom-opensearch: Custom OpenSearch Serverless pipeline (more control)
                        hybrid: Bedrock KB + custom re-ranking pipeline

DOCUMENT_SOURCES:       [REQUIRED - s3 | confluence | sharepoint | web | mixed]
DOCUMENTS_S3_PATH:      [REQUIRED - s3://bucket/path/to/documents/]
SUPPORTED_FILE_TYPES:   [OPTIONAL: pdf,txt,docx,html,md]

EMBEDDING_MODEL:        [OPTIONAL: amazon.titan-embed-text-v2:0]
                        Options: amazon.titan-embed-text-v2:0 (1536 dims)
                                 cohere.embed-english-v3 (1024 dims)
                                 cohere.embed-multilingual-v3 (non-English)
VECTOR_DIMENSIONS:      [OPTIONAL: 1536 for Titan v2, 1024 for Cohere]

CHUNK_SIZE:             [OPTIONAL: 512]
CHUNK_OVERLAP:          [OPTIONAL: 50]
CHUNKING_STRATEGY:      [OPTIONAL: fixed | semantic | hierarchical]

GENERATION_MODEL:       [OPTIONAL: anthropic.claude-3-5-sonnet-20241022-v2:0]
RETRIEVAL_TOP_K:        [OPTIONAL: 5]
RERANKING_ENABLED:      [OPTIONAL: false]
RERANKING_MODEL:        [OPTIONAL: cohere.rerank-english-v3:0]
MAX_TOKENS_GENERATION:  [OPTIONAL: 1024]
TEMPERATURE:            [OPTIONAL: 0.1]

OPENSEARCH_COLLECTION:  [OPTIONAL: {PROJECT_NAME}-{ENV}-vectors]
INDEX_NAME:             [OPTIONAL: {PROJECT_NAME}-{ENV}-index]
INGESTION_SCHEDULE:     [OPTIONAL: on-demand | daily | hourly]
```

---

## Task

### OPTION A: RAG_APPROACH = bedrock-kb

Generate Amazon Bedrock Knowledge Base setup:

```
bedrock_kb/
├── setup/
│   ├── create_knowledge_base.py    # Bedrock KB creation with AOSS
│   ├── create_data_source.py       # S3 data source configuration
│   └── sync_data_source.py         # Trigger ingestion sync
├── inference/
│   ├── rag_query.py                # RAG query using retrieve+generate
│   └── api_handler.py              # Lambda function for RAG API
├── evaluation/
│   └── ragas_eval.py               # RAG evaluation using RAGAS
└── cdk/
    └── bedrock_kb_stack.py         # CDK stack for all resources
```

**create_knowledge_base.py**: Create AOSS collection + index, create vector KB with `bedrock.create_knowledge_base()`, configure storage and embedding. IAM role with Bedrock, S3, AOSS permissions.

**rag_query.py**: Query using `bedrock-agent-runtime.retrieve_and_generate()`:
```python
response = bedrock_agent_runtime.retrieve_and_generate(
    input={"text": user_query},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": kb_id,
            "modelArn": f"arn:aws:bedrock:{region}::foundation-model/{GENERATION_MODEL}",
            "retrievalConfiguration": {
                "vectorSearchConfiguration": {"numberOfResults": RETRIEVAL_TOP_K}
            }
        }
    }
)
```

### OPTION B: RAG_APPROACH = custom-opensearch

Generate custom pipeline with full control:

```
custom_rag/
├── ingestion/
│   ├── document_loader.py          # Load PDF, DOCX, HTML, TXT, MD
│   ├── text_chunker.py             # Fixed / semantic / hierarchical chunking
│   ├── embedder.py                 # Bedrock embedding generation (batched)
│   ├── opensearch_indexer.py       # AOSS vector indexing (bulk upsert)
│   └── ingestion_pipeline.py       # Orchestrated ingestion flow
├── retrieval/
│   ├── retriever.py                # k-NN vector search + BM25 hybrid
│   ├── reranker.py                 # Cohere reranking (optional)
│   └── query_expander.py           # HyDE / query expansion
├── generation/
│   ├── prompt_builder.py           # Prompt template with context injection
│   ├── generator.py                # Bedrock Converse API call
│   └── response_parser.py          # Extract answer, citations, confidence
├── pipeline/
│   ├── rag_pipeline.py             # End-to-end RAG orchestration
│   └── lambda_handler.py           # Lambda entry point for API
├── evaluation/
│   ├── ragas_eval.py               # Faithfulness, relevancy, precision, recall
│   └── create_eval_dataset.py      # Generate Q&A pairs for evaluation
├── infrastructure/
│   ├── opensearch_setup.py         # Create AOSS collection + index
│   └── step_functions_ingestion.py # Step Functions for batch ingestion
└── tests/
    └── test_rag_pipeline.py
```

**document_loader.py**: Multi-format loader (PDF via PyPDF2+Textract, DOCX via python-docx, HTML via BeautifulSoup). Returns `{text, source, page, timestamp}`.

**text_chunker.py**: Three strategies:
- Fixed: `RecursiveCharacterTextSplitter(chunk_size, chunk_overlap)`
- Semantic: split on sentence boundaries, merge to size limit
- Hierarchical: parent (1024 tokens) + child (256 tokens) chunks

**embedder.py**: Batch Bedrock embedding (25 docs/call for Titan), retry with exponential backoff.

**opensearch_indexer.py**: AOSS bulk index with knn_vector mapping (HNSW, cosinesimil), upsert by document hash, delete stale docs.

**retriever.py**: Dense (k-NN) + sparse (BM25) hybrid retrieval with reciprocal rank fusion.

**prompt_builder.py**: System prompt enforcing answer-from-context-only with citation tracking.

**generator.py**: Bedrock Converse API with streaming support.

**ragas_eval.py**: RAGAS metrics: faithfulness, answer relevancy, context precision, context recall.

---

## Output Format

Output ALL files for the chosen RAG_APPROACH with headers:
```
### FILE: [path/filename.py]
```

---

## Requirements & Constraints

**Quality:** Chunk metadata must include source_file, page_number, chunk_id, ingestion_timestamp. Dedup by content hash. Citation tracking in responses.

**Performance:** Batch embedding calls (25 docs/call Titan, 96 Cohere). AOSS HNSW: ef_search=512, m=16. Retrieval latency target < 300ms p99.

**Security:** AOSS data access policy locked to Lambda execution role. CloudTrail for all Bedrock API calls. Document-level access control via metadata filtering.

**Cost:** Cache embeddings in DynamoDB (hash→vector). Avoid re-embedding unchanged docs. AOSS: $0.24/OCU-hour, monitor utilization.

---

## Code Scaffolding Hints

**AOSS index creation:**
```python
index_body = {
    "settings": {"index": {"knn": True, "knn.algo_param.ef_search": 512}},
    "mappings": {
        "properties": {
            "embedding": {"type": "knn_vector", "dimension": VECTOR_DIMENSIONS,
                          "method": {"name": "hnsw", "space_type": "cosinesimil",
                                     "engine": "nmslib", "parameters": {"ef_construction": 512, "m": 16}}},
            "text": {"type": "text"},
            "source": {"type": "keyword"},
            "chunk_id": {"type": "keyword"}
        }
    }
}
```

**Bedrock embedding:**
```python
response = bedrock_runtime.invoke_model(
    modelId="amazon.titan-embed-text-v2:0",
    body=json.dumps({"inputText": chunk_text, "dimensions": 1536, "normalize": True})
)
embedding = json.loads(response["body"].read())["embedding"]
```

**Bedrock Converse:**
```python
response = bedrock.converse(
    modelId=GENERATION_MODEL,
    messages=[{"role": "user", "content": [{"text": formatted_prompt}]}],
    system=[{"text": SYSTEM_PROMPT}],
    inferenceConfig={"maxTokens": MAX_TOKENS_GENERATION, "temperature": TEMPERATURE}
)
```

---

## Integration Points

- **Upstream**: `devops/04` → IAM permissions for Bedrock, AOSS, S3, Lambda
- **Upstream**: `iac/03` → Terraform IaC for AOSS + Bedrock KB infrastructure
- **Downstream**: `mlops/03` → custom SageMaker endpoint for generation
- **Downstream**: `devops/03` → CloudWatch dashboards for RAG latency/quality
- **Downstream**: `cicd/04` → GitHub Actions triggers re-ingestion on S3 changes
