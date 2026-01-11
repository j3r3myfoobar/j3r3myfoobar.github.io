---
layout: single
title: "Building a Virtual Me: RAG-Powered Resume Chatbot on AWS"
date: 2026-01-11
categories: [blogging]
---

## The Resume Problem

I've been frustrated with traditional resumes for a while now. Not because mine is bad, but because the format itself is a bit broken.

**Two core problems:**

1. **Resumes don't translate experience well.** You spend hours crafting bullet points that technically describe what you did, but they fail to capture the *why* behind your decisions, the trade-offs you considered, or the context that made your work meaningful. A line like "Implemented serverless RAG pipeline" does not tell you much.

2. **Resumes are boring.** Reading a resume is like reading a phone book. It's a static, one-dimensional artifact that forces recruiters to play detective, inferring things that should be explicit. Want to know some details about they why and the how trade-offs? You'd have to schedule a call. 

I wanted something better. Something that could answer questions about my experience for me.

So I built **Virtual Me**, an AI chatbot that represents me, fee with my actual resume and technical knowledge, using Retrieval Augmented Generation (RAG) .

You can try it here: [chat.lemaire.tel](https://chat.lemaire.tel)

The rest of this post are some technical details I learnt along the way.

---

## Request/Response Flow with JSON Examples

Here is the complete end-to-end flow from user input to AI response, including actual JSON payloads exchanged between components, CORS handling, validation, RAG pipeline execution, and error cases.

### Request Flow Diagram

```
User Input
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. Web UI (Deep Chat)                                       │
│    POST https://api.lemaire.tel/chat                        │
│    Content-Type: application/json                           │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Route 53                                                  │
│    DNS: api.lemaire.tel → API Gateway Endpoint              │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. API Gateway HTTP API                                      │
│    Transforms HTTP → Lambda Event (API Gateway v2 format)   │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. Lambda (lambda_function.py)                               │
│    - Validates with Pydantic (ChatRequest)                  │
│    - Truncates to last 20 messages                          │
│    - Invokes run_rag_pipeline(question)                     │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. LangGraph Workflow (rag/pipeline.py)                      │
│    ┌──────────────┐      ┌──────────────┐                  │
│    │ retrieve_node│ ───► │ generate_node│                  │
│    └──────────────┘      └──────────────┘                  │
│           │                      │                          │
│           ↓                      ↓                          │
│      DynamoDB              Amazon Bedrock                   │
└─────────────────────────────────────────────────────────────┘
    ↓
Response (reverse path)
```

### CORS Preflight Handling

Before the actual POST request, browsers send a preflight OPTIONS request when making cross-origin calls (`chat.lemaire.tel` → `api.lemaire.tel`). The Lambda function explicitly returns 200 OK with CORS headers to allow the request:

```python
# lambda_function.py handles OPTIONS preflight
if event.get("requestContext", {}).get("http", {}).get("method") == "OPTIONS":
    return {
        "statusCode": 200,
        "headers": {
            "Access-Control-Allow-Origin": "https://chat.lemaire.tel",
            "Access-Control-Allow-Methods": "POST, OPTIONS",
            "Access-Control-Allow-Headers": "content-type"
        }
    }
```

### Step 1: Web UI Request (Deep Chat Format)

The Deep Chat UI sends a JSON payload with the conversation history:

```json
{
  "messages": [
    {
      "role": "user",
      "text": "What is your experience with AWS?"
    }
  ]
}
```

For conversations with history:

```json
{
  "messages": [
    {
      "role": "user",
      "text": "What technologies do you work with?"
    },
    {
      "role": "ai",
      "text": "I work with Python, AWS Lambda, Terraform, and Amazon Bedrock..."
    },
    {
      "role": "user",
      "text": "Tell me more about your AWS experience"
    }
  ]
}
```

### Step 2: API Gateway Event (Lambda Input)

API Gateway transforms the HTTP request into a Lambda event (AWS API Gateway v2 format):

```json
{
  "version": "2.0",
  "routeKey": "POST /chat",
  "rawPath": "/chat",
  "requestContext": {
    "accountId": "123456789012",
    "apiId": "abc123xyz",
    "domainName": "api.lemaire.tel",
    "requestId": "abc-123-def-456",
    "http": {
      "method": "POST",
      "path": "/chat",
      "protocol": "HTTP/1.1",
      "sourceIp": "203.0.113.42",
      "userAgent": "Mozilla/5.0..."
    },
    "time": "10/Jan/2026:14:23:45 +0000",
    "timeEpoch": 1736517825000
  },
  "headers": {
    "content-type": "application/json",
    "host": "api.lemaire.tel",
    "origin": "https://chat.lemaire.tel"
  },
  "body": "{\"messages\":[{\"role\":\"user\",\"text\":\"What is your experience with AWS?\"}]}",
  "isBase64Encoded": false
}
```

### Step 3: Lambda Processing

Lambda validates, truncates, and extracts the request:

```python
# lambda_function.py
body = json.loads(event.get("body", "{}"))

# Validation: Pydantic ensures schema validity. Malformed requests fail early.
chat_request = ChatRequest(**body)

# Truncation: Only last 20 messages retained to limit token usage and costs.
# Older context is irrelevant for immediate queries.
messages = chat_request.messages
if len(messages) > 20:
    messages = messages[-20:]

# Extract question and invoke RAG pipeline
question = messages[-1].text
answer = run_rag_pipeline(question)  # Invokes LangGraph workflow
```

### Step 4: LangGraph Internal State

The RAG pipeline (`rag/pipeline.py`) implements a LangGraph state machine with two nodes:
- `retrieve_node`: Calls DynamoDBRetriever, fetches chunks, flattens to context string
- `generate_node`: Injects context into System Prompt, calls Bedrock

State flows through the workflow:

**Initial State (after retrieve_node):**
```python
{
  "question": "What is your experience with AWS?",
  "context": "Jeremy Lemaire\n\nAWS Solutions Architect...",
  "messages": [
    HumanMessage(content="You are a helpful assistant representing Jeremy..."),
    HumanMessage(content="What is your experience with AWS?")
  ]
}
```

**After generate_node:**
```python
{
  "question": "What is your experience with AWS?",
  "context": "...",
  "messages": [
    HumanMessage(content="You are a helpful assistant..."),
    HumanMessage(content="What is your experience with AWS?"),
    AIMessage(content="I have 8 years of experience as an AWS Solutions Architect...")
  ]
}
```

### Step 5: DynamoDB Query (Internal)

The embedding for "What is your experience with AWS?" is computed:
```python
query_embedding = [0.123, -0.456, 0.789, ...]  # 1024-dimensional vector
```

DynamoDB scan retrieves all items and computes cosine similarity client-side:
```python
# Results sorted by similarity score
[
  {
    "id": "doc_0_abc123",
    "text": "Jeremy Lemaire\n\nAWS Solutions Architect...",
    "similarity": 0.87
  },
  {
    "id": "doc_3_def456",
    "text": "Certifications:\n- AWS Certified Solutions Architect...",
    "similarity": 0.82
  },
  {
    "id": "doc_1_ghi789",
    "text": "Technical Skills:\n- Python, Terraform, AWS Lambda...",
    "similarity": 0.78
  }
]
# Top 3 are concatenated into the context string
```

### Step 6: Bedrock API Call (Internal)

Generate Node calls Amazon Bedrock:

```json
{
  "modelId": "us.amazon.nova-lite-v2:0",
  "messages": [
    {
      "role": "user",
      "content": "You are a Virtual Clone representing Jeremy Lemaire. Answer ONLY using information from the CONTEXT below.\n\nCONTEXT:\nJeremy Lemaire\n\nAWS Solutions Architect..."
    },
    {
      "role": "user",
      "content": "What is your experience with AWS?"
    }
  ],
  "inferenceConfig": {
    "temperature": 0.1,
    "maxTokens": 500
  }
}
```

**Bedrock Response:**
```json
{
  "output": {
    "message": {
      "role": "assistant",
      "content": [
        {
          "text": "I have 8 years of experience as an AWS Solutions Architect..."
        }
      ]
    }
  },
  "usage": {
    "inputTokens": 234,
    "outputTokens": 47,
    "totalTokens": 281
  }
}
```

---

## Architecture & Lifecycle: Cold vs Warm Start

### Cold Start (~4 seconds)
Occurs when no active container exists (after ~15 mins inactivity).
1. **Python Imports**: `import langchain` (~1.5s).
2. **Module Level Initialization**: Global singleton variables initialized.
3. **Handler Execution**:
   - Detects `_vector_store is None`.
   - **Initialization**: Establishes DynamoDB connection (SSL handshake ~0.5s), compiles graph (~0.5s).
   - **Total Latency**: ~4s

### Warm Start (<400ms)
Occurs on subsequent requests to the same container.
1. **Container State**: Memory preserved.
2. **No Imports**: Modules already loaded.
3. **Persisted Globals**: `_vector_store` already initialized.
   ```python
   def get_retriever():
       global _vector_store
       if _vector_store:
           return _vector_store  # Immediate return
   ```
4. **Execution**: Direct `graph.invoke()`.
   - Runtime limited to API I/O: ~20ms DynamoDB scan + ~300ms Bedrock generation.

**Optimization**: Module-level global variables reuse connections across invocations.

---

## Technical Key Points

### DynamoDB as a Vector Store

Vector search requires comparing the query against **every single document** to determine semantic proximity.

- **Implementation**: Full table scan with client-side cosine similarity
- **Process**: Loads all 50 chunks into memory (~10ms) and computes cosine similarity in Python
- **Scalability**: Effective up to ~1000 chunks

**Why this works**: For <1000 items, brute-force scanning is practically free and requires zero maintenance. 1000 chunks × 4KB = 4MB data. DynamoDB scans 1MB per request = 4 round-trips (~60ms) + Python cosine calculation (~40ms) = ~100ms total.

**When it doesn't**: Beyond 1000 chunks, you need Approximate Nearest Neighbor (ANN) algorithms via AWS OpenSearch, ChromaDB, or PostgreSQL with pgvector. ANN reduces search from O(N) to O(log n).

### Binary Embedding Optimization
Embeddings are **1024 floats** (Titan V2 default).
```json
[0.123456789, 0.23456789, ...] // JSON representation ≈ 12KB per row
```

Packed into binary using `struct.pack`:
- Standard float = 4 bytes. 1024 * 4 = **4KB**
- **Result**: 3x reduction in storage size and read throughput costs
- **Impact**: Saves ~8GB for 1M rows

### LangGraph vs LangChain

**LangChain (The "Traditionnal" Way)**
```python
# Hidden state flow. A bit opaque
chain = retriever | prompt | llm | parser
result = chain.invoke("question")
```

**With LangGraph**
```python
# Clearly defined state schema
class GraphState(TypedDict):
    question: str
    context: str
    messages: List[BaseMessage]

# Pure function nodes
def retrieve(state):
    docs = retriever.get_relevant_documents(state["question"])
    return {"context": format_docs(docs)}

def generate(state):
    response = llm.invoke(prompt.format(context=state["context"]))
    return {"messages": [response]}

# Explicit control flow
workflow.add_edge("retrieve", "generate")
```

**Advantage**: Full visibility into data transformations at every step.

---

## Few performance tricks

A bit of overengineering for what I need but that was fun to look at.

### L1 Cold Start Optimization

AWS uses **Firecracker** micro-VMs for Lambda execution environments. Once created, the environment is frozen and reused. I exploit this by using global variables to persist state:

```python
# src/rag/dynamodb_retriever.py
_vector_store = None

def get_vector_store():
    global _vector_store
    if _vector_store is None:
        # EXPENSIVE: Runs only on new environment (~4s)
        _vector_store = DynamoDBVectorStore(...)
    return _vector_store
```

This saves **4 seconds** on every warm start.

### Context Sliding Window

Unbounded conversation to avoid **Token Explosion**.

If I send 100 messages of history, the 101st request pays for processing all 100 previous turns. Costs grow linearly.

**Solution**: Strict cap at 20 messages.

```python
if len(messages) > 20:
    messages = messages[-20:]
```

Deterministic cost ceiling: Max cost per turn = (20 msgs × avg_tokens) + new_query

### RAG Hyperparameters

#### Top K = 3
- **Why not 1?** Markdown splitting separates headers from content. Retrieving 3 captures surrounding semantic hierarchy.
- **Why not 10?** Signal-to-noise ratio degrades. LLMs are known to ignore information buried in the middle ("Lost in the Middle" problem). 10 chunks = 3x more input tokens.

#### Temperature = 0.1
- **Goal**: Determinism.
- **Logic**: The LLM should act as a **Retrieval Engine**, not a Creative Writer.
  - `0.1`: "According to the text, Jeremy studied AWS." (Fact)
  - `0.9`: "Jeremy, a cloud wizard, soared through the AWS skies..." (Hallucination)

### AWS Adaptive Retries

Standard retries (fixed interval) worsen AWS throttling scenarios (Thundering Herd problem).

```python
from botocore.config import Config

BEDROCK_RETRY_CONFIG = Config(
    retries={
        'max_attempts': 3,
        'mode': 'adaptive'  # Dynamic backoff for HTTP 429
    }
)
```

### AWS X-Ray Tracing

Lambda X-Ray is enabled:
```hcl
resource "aws_lambda_function" "virtual_me" {
  tracing_config {
    mode = "Active"
  }
}
```

This automatically traces:
- Lambda execution time and cold starts
- DynamoDB Scan operations with latency
- Bedrock InvokeModel calls with token counts and latency
- All boto3 SDK calls

**No Python SDK required** - Lambda's X-Ray integration handles it automatically.

**API Gateway Limitation**: I use **HTTP API (v2)** instead of REST API (v1) because:
- **70% cheaper**: $1.00/million vs $3.50/million requests
- **Simpler CORS**: Native configuration vs manual OPTIONS handling

Trade-off: HTTP API v2 does **not support X-Ray tracing**. Only Lambda traces are captured.

---

## Resource Utilization

### Lambda Memory: 512MB

**Breakdown:**
- Python Runtime + Boto3: ~120MB
- LangChain + Dependencies: ~180MB
- Graph Compilation & Working State: ~50MB
- **Total Overhead**: ~350MB
- **Safety Margin**: ~160MB (for embedding processing and JSON overhead)

Using less than 512MB leads to `Memory Limit Exceeded` during LangChain initialization.

---

## Data Storage Strategy

### DynamoDB Schema

**Item Structure:**
```json
{
  "id": "doc_0_abc123",
  "text": "I have 8 years...",       // Retrieved & sent to LLM
  "embedding": <Binary Blob>,    // Search key (cosine similarity)
  "metadata": "{\"source\": \"...\"}"
}
```

### DynamoDB vs Dedicated Vector DB

| Feature | DynamoDB (My Approach) | Vector DB (Chroma) |
| :--- | :--- | :--- |
| **Search Logic** | Client-side (Python) - fetch everything, compute similarity | Server-side - DB engine finds neighbors |
| **Scalability** | O(N) - slower as data grows | O(log n) - instant even with millions of rows |
| **Cost** | High read cost (pay to read every row) | Optimized for search |
| **Maintenance** | Zero | Run dedicated cluster ($hundreds/month) |

**Why DynamoDB**: "Serverless Poor Man's Vector DB". For <1000 items, brute-force scanning is practically free and requires zero maintenance.

---

## Cost Estimate

For personal use (~100 conversations/month): **~$3-5/month**

| Service | Estimated Cost |
|---------|---------------|
| Lambda + API Gateway | $0.30 - $1.00 |
| DynamoDB | $1.00 - $5.00 |
| Bedrock (Nova 2 Lite) | $0.50 - $2.00 |
| S3 + CloudFront | $0.15 - $0.50 |
| Route53 | $0.50 |

**Idle Cost**: $0. Pure pay-per-request serverless architecture.

---

## Conclusion

Traditional resumes are broken. They don't capture context, trade-offs, or technical depth. They're boring documents that force everyone to play guessing games.

The architecture is fully serverless:
- **Compute**: Lambda (pay-per-request)
- **Storage**: DynamoDB (pay-per-request)
- **Inference**: Bedrock (pay-per-token)
- **Idle Cost**: $0

It's cheap, fast (400ms warm start), and actually represents how I think about my work - with context and technical nuance.

Try it: [chat.lemaire.tel](https://chat.lemaire.tel)

Source code: [github.com/ox00004a/virtualme](https://github.com/ox00004a/virtualme) 
