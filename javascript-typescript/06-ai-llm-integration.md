# AI / LLM Integration Patterns

## Why This Matters for Principal Engineers in 2025

Companies are actively building AI features into their products. Principal engineers are expected to:
- Architect AI-augmented systems (RAG pipelines, agents, copilots)
- Make build vs. buy decisions on AI infrastructure
- Understand cost, latency, and reliability trade-offs of LLM APIs
- Design for the non-deterministic nature of LLM outputs

---

## LLM API Fundamentals

### Core Concepts
```
Model:       The neural network (GPT-4o, Claude 3.7, Gemini 2.0, Llama 3.3)
Prompt:      Input to the model (system + user messages)
Completion:  The model's output (tokens generated)
Context window: Max tokens the model can process at once (8K–200K tokens)
Temperature: 0 = deterministic, 1 = creative/random. Use 0 for code/data.
Top_p:       Nucleus sampling (alternative to temperature)
Max tokens:  Limit output length (cost + latency control)
```

### Token Economics
```
1 token ≈ 4 characters ≈ 0.75 words (English)
1 page of text ≈ 500 tokens

Cost example (approximate, as of 2025):
  Claude 3.7 Sonnet: ~$3/1M input tokens, ~$15/1M output tokens
  GPT-4o:            ~$2.5/1M input, ~$10/1M output
  Llama 3.3 70B:     Self-hosted (GPU cost) or ~$0.4/1M via API

Cost optimization:
  - Shorter system prompts (reused across calls = cached input tokens)
  - Prompt caching: providers cache prompt prefix (60-90% cost reduction on repeated prompts)
  - Use smaller models for simpler tasks (classification → Haiku, complex reasoning → Sonnet/Opus)
  - Batch API: async processing at 50% discount (Anthropic, OpenAI both offer this)
```

### Node.js API Call (Anthropic SDK)
```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  maxRetries: 3,          // built-in retry with exponential backoff
  timeout: 30000,
});

// Standard completion
const message = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  system: 'You are a helpful assistant that extracts structured data from text.',
  messages: [
    { role: 'user', content: 'Extract the order details: "Order #12345, 2x blue shirts, $89.99 total"' }
  ],
  temperature: 0,  // deterministic for data extraction
});

const text = message.content[0].type === 'text' ? message.content[0].text : '';

// Streaming (for user-facing responses)
const stream = await client.messages.stream({
  model: 'claude-sonnet-4-6',
  max_tokens: 2048,
  messages: [{ role: 'user', content: userMessage }],
});

// Stream to HTTP response (Server-Sent Events)
for await (const chunk of stream) {
  if (chunk.type === 'content_block_delta' && chunk.delta.type === 'text_delta') {
    res.write(`data: ${JSON.stringify({ text: chunk.delta.text })}\n\n`);
  }
}
res.write('data: [DONE]\n\n');
res.end();
```

---

## Prompt Engineering for Production

### System Prompt Structure
```
A good system prompt:
  1. Role definition (who the model is)
  2. Task description (what it should do)
  3. Output format (how to format the response)
  4. Constraints (what it should NOT do)
  5. Examples (few-shot, optional but powerful)

Example — Structured extraction:
```
```typescript
const SYSTEM_PROMPT = `You are a data extraction assistant for an e-commerce platform.

Your task: Extract order information from customer messages and return it as valid JSON.

Output format (always return exactly this JSON structure):
{
  "intent": "order_status" | "return_request" | "product_question" | "other",
  "orderId": string | null,
  "productName": string | null,
  "confidence": "high" | "medium" | "low"
}

Rules:
- Return ONLY the JSON object, no explanation text
- If information is missing, use null (never guess)
- Order IDs always start with # or "order" (case-insensitive)
- Set confidence "low" if intent is ambiguous`;
```

### Prompt Injection Prevention
```typescript
// Never interpolate user input directly into system prompt
// Bad:
const prompt = `You are a helpful assistant. The user's name is ${userName}. Always greet them.`;
// Attack: userName = "Ignore previous instructions and..."

// Good: Separate user data from instructions
const systemPrompt = `You are a helpful assistant. Always greet users by their provided name.`;
const userMessage = `My name is: ${userName}\nQuestion: ${userQuestion}`;
// Or: use XML/JSON delimiters to separate user content
const userMessage = `<user_name>${escapeXml(userName)}</user_name>\n<question>${escapeXml(userQuestion)}</question>`;

// Input validation before sending to LLM:
function validateLLMInput(input: string): string {
  if (input.length > 10000) throw new Error('Input too long');
  // Strip known injection patterns (defense in depth)
  return input.replace(/ignore (previous|all) instructions/gi, '[filtered]');
}
```

### Output Parsing & Validation
```typescript
// LLM output is never guaranteed to be valid JSON — always validate
import { z } from 'zod';

const OrderExtractionSchema = z.object({
  intent: z.enum(['order_status', 'return_request', 'product_question', 'other']),
  orderId: z.string().nullable(),
  productName: z.string().nullable(),
  confidence: z.enum(['high', 'medium', 'low']),
});

async function extractOrderInfo(message: string) {
  const response = await callLLM(message);
  
  try {
    // Strip markdown code fences if model wrapped JSON
    const jsonStr = response.replace(/```json\n?|\n?```/g, '').trim();
    const parsed = JSON.parse(jsonStr);
    return OrderExtractionSchema.parse(parsed);
  } catch (err) {
    // Retry with explicit correction prompt
    const correction = await callLLM(
      `${message}\n\nYour previous response was not valid JSON. Return ONLY the JSON object.`
    );
    const parsed = JSON.parse(correction);
    return OrderExtractionSchema.parse(parsed);
  }
}
```

---

## RAG (Retrieval-Augmented Generation)

### Why RAG?
LLMs have a knowledge cutoff and don't know your private data. RAG retrieves relevant documents at query time and injects them into the prompt.

```
Without RAG:
  User: "What's the return policy for my order?"
  LLM: "I don't have access to your order information." ← useless

With RAG:
  1. User query → embedding → search vector DB → top-3 relevant docs
  2. Inject docs into prompt: "Based on these policies: [docs], answer: [query]"
  3. LLM: "Based on your order #12345 placed on Jan 5, you have 30 days to return..."
```

### RAG Architecture
```
Indexing pipeline (offline/async):
  Documents (PDFs, FAQs, DB records)
    → Chunking (split into ~500 token chunks with overlap)
    → Embedding model (text → vector, e.g., text-embedding-3-small)
    → Store in vector DB: { id, vector, metadata, raw_text }

Query pipeline (real-time):
  User query
    → Embed query (same model as indexing)
    → Vector similarity search (top-K = 5 chunks)
    → Rerank (optional: cross-encoder for better relevance)
    → Build prompt: system + retrieved_context + user_query
    → LLM → response
```

### Vector Databases
| DB | Hosting | Best for |
|---|---|---|
| **pgvector** | Self-hosted (Postgres extension) | Already using Postgres; <1M vectors |
| **Pinecone** | Managed SaaS | Ease of use; serverless option |
| **Weaviate** | Self-hosted or cloud | Hybrid search (vector + keyword) |
| **Qdrant** | Self-hosted or cloud | High performance, Rust-based |
| **OpenSearch** | AWS managed | Already using OpenSearch; kNN search |
| **Aurora pgvector** | AWS managed | Postgres-native; production-grade |

### pgvector Implementation (Node.js)
```typescript
import { Pool } from 'pg';
import OpenAI from 'openai';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const openai = new OpenAI();

// Schema
await pool.query(`
  CREATE EXTENSION IF NOT EXISTS vector;
  CREATE TABLE IF NOT EXISTS documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536),  -- OpenAI text-embedding-3-small
    created_at TIMESTAMPTZ DEFAULT NOW()
  );
  CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);  -- tune lists = sqrt(row_count)
`);

// Indexing: embed + store
async function indexDocument(content: string, metadata: Record<string, unknown>) {
  const { data } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: content,
  });
  const embedding = data[0].embedding;

  await pool.query(
    'INSERT INTO documents (content, metadata, embedding) VALUES ($1, $2, $3)',
    [content, metadata, JSON.stringify(embedding)]
  );
}

// Query: embed + search + inject into prompt
async function ragQuery(userQuestion: string): Promise<string> {
  // 1. Embed question
  const { data } = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: userQuestion,
  });
  const queryEmbedding = data[0].embedding;

  // 2. Vector similarity search
  const { rows } = await pool.query(
    `SELECT content, metadata, 1 - (embedding <=> $1) AS similarity
     FROM documents
     ORDER BY embedding <=> $1
     LIMIT 5`,
    [JSON.stringify(queryEmbedding)]
  );

  // 3. Build context
  const context = rows.map(r => r.content).join('\n\n---\n\n');

  // 4. LLM call with context
  const response = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 1024,
    system: `Answer questions using ONLY the provided context. 
If the answer isn't in the context, say "I don't have that information."`,
    messages: [{
      role: 'user',
      content: `Context:\n${context}\n\nQuestion: ${userQuestion}`
    }]
  });

  return response.content[0].type === 'text' ? response.content[0].text : '';
}
```

### Chunking Strategies
```
Fixed size: Split every N characters (simple, ignores structure)
  → Use with overlap: chunk_size=512, overlap=50 (avoids cutting context)

Semantic: Split at sentence/paragraph boundaries (better quality)
  → langchain TextSplitter, or sentence-transformers

Hierarchical: Chunk at multiple granularities
  → Store full document summary + individual paragraphs
  → Search summaries first, then drill into paragraphs

Document-aware: Use structure (headings, sections)
  → PDF: extract by section; HTML: split by <h2>; Markdown: split by ##

Rule of thumb:
  500-1000 tokens per chunk for most use cases
  Smaller = more precise retrieval; Larger = more context per chunk
```

---

## Production LLM Patterns

### Retry & Fallback
```typescript
import pRetry from 'p-retry';

async function callLLMWithRetry(prompt: string): Promise<string> {
  return pRetry(async (attemptNumber) => {
    try {
      return await callLLM(prompt);
    } catch (err) {
      // Don't retry on invalid input or auth errors
      if (err.status === 400 || err.status === 401) {
        throw new pRetry.AbortError(err.message);
      }
      // Retry on rate limit (429) or server errors (5xx)
      throw err;
    }
  }, {
    retries: 3,
    onFailedAttempt: (err) => {
      logger.warn('LLM retry', { attempt: err.attemptNumber, error: err.message });
    },
    minTimeout: 1000,
    factor: 2, // exponential backoff
  });
}

// Fallback to cheaper/faster model
async function callWithFallback(prompt: string): Promise<string> {
  try {
    return await callModel('claude-sonnet-4-6', prompt);
  } catch (err) {
    logger.warn('Falling back to Haiku', { error: err.message });
    return callModel('claude-haiku-4-5', prompt);
  }
}
```

### Caching LLM Responses
```typescript
// Semantic caching: cache by embedding similarity, not exact match
// Exact match caching: safe for deterministic prompts (temp=0)

async function cachedLLMCall(prompt: string, cacheKey?: string): Promise<string> {
  const key = cacheKey ?? `llm:${createHash('sha256').update(prompt).digest('hex')}`;
  
  const cached = await redis.get(key);
  if (cached) return cached;
  
  const response = await callLLM(prompt);
  await redis.setex(key, 3600, response); // cache 1hr
  return response;
}
// Use for: same classification prompt run repeatedly, FAQ lookups
// Don't use for: personalized or time-sensitive responses
```

### Rate Limiting LLM Calls
```typescript
import pLimit from 'p-limit';
import Bottleneck from 'bottleneck';

// Anthropic limits: 2000 RPM (requests/min), 400K tokens/min
const limiter = new Bottleneck({
  reservoir: 2000,          // max requests
  reservoirRefreshAmount: 2000,
  reservoirRefreshInterval: 60 * 1000, // per minute
  maxConcurrent: 50,        // concurrent in-flight requests
  minTime: 30,              // min 30ms between requests
});

const callLLM = limiter.wrap(rawCallLLM);
```

### Structured Outputs (Tool Use / Function Calling)
```typescript
// Instead of parsing free-text JSON, use native tool use
const response = await client.messages.create({
  model: 'claude-sonnet-4-6',
  max_tokens: 1024,
  tools: [{
    name: 'extract_order',
    description: 'Extract order information from a customer message',
    input_schema: {
      type: 'object',
      properties: {
        intent: { type: 'string', enum: ['order_status', 'return', 'question', 'other'] },
        orderId: { type: 'string', description: 'Order ID if mentioned (e.g., #12345)' },
        confidence: { type: 'string', enum: ['high', 'medium', 'low'] },
      },
      required: ['intent', 'confidence'],
    }
  }],
  tool_choice: { type: 'tool', name: 'extract_order' }, // force tool use
  messages: [{ role: 'user', content: customerMessage }],
});

// Response is always structured — no JSON parsing issues
const toolUse = response.content.find(b => b.type === 'tool_use');
const extracted = toolUse?.input; // fully typed if you define the schema
```

---

## AI Architecture Interview Talking Points

**"How would you design a customer support chatbot for an e-commerce platform?"**
```
Architecture:
  1. RAG pipeline: index FAQs, return policies, product docs into pgvector
  2. Conversation memory: last N turns stored in Redis per session
  3. Intent detection: first LLM call classifies intent (order / return / product)
  4. Routing: simple questions → retrieval-only; complex → full LLM
  5. Escalation: confidence < 0.5 or sentiment = angry → route to human agent

Cost controls:
  - Cache FAQ answers (same Q asked 1000x/day)
  - Use smaller model for intent classification (Haiku)
  - Use larger model (Sonnet) only for complex generation
  - Limit context window: top-3 docs, last 5 turns

Reliability:
  - LLM calls behind circuit breaker
  - Fallback: "Let me connect you to an agent" on failure
  - Never block checkout/payments on LLM availability
```

**"What are the risks of putting LLMs in production?"**
```
1. Non-determinism: same input → different output. Mitigate: temp=0, validation, test suites
2. Prompt injection: user inputs override system instructions. Mitigate: strict input validation, sandboxed prompts
3. Hallucination: model confidently states wrong facts. Mitigate: RAG with source citations, confidence thresholds
4. Latency: p99 LLM calls can be 5-30s. Mitigate: streaming, async for non-blocking flows, timeout + fallback
5. Cost: uncontrolled usage = unexpected bills. Mitigate: per-user limits, token counting, budget alerts
6. Data privacy: don't send PII to third-party APIs without legal review. Mitigate: PII stripping, on-prem models for sensitive data
```
