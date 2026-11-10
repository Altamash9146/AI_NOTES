# Section 7: Azure OpenAI + AI (8 Qs)

> [← Back to index](README.md)


### 1. Azure OpenAI integration — code flow
**Flow:** Client → Next.js route / Node API (auth + rate limit) → build messages (system prompt + history + user message) → Azure OpenAI **deployment** → stream back → save to DB.
```ts
import { AzureOpenAI } from 'openai';

const client = new AzureOpenAI({
  endpoint: process.env.AZURE_OPENAI_ENDPOINT,        // https://<resource>.openai.azure.com
  apiKey: process.env.AZURE_OPENAI_API_KEY,           // or Entra ID (managed identity) — better
  apiVersion: process.env.AZURE_OPENAI_API_VERSION,
  deployment: process.env.AZURE_OPENAI_DEPLOYMENT,    // YOUR deployment name, not model name
});

const res = await client.chat.completions.create({
  model: process.env.AZURE_OPENAI_DEPLOYMENT!,
  messages: [{ role: 'system', content: SYSTEM_PROMPT }, ...history, { role: 'user', content: userMsg }],
  temperature: 0.3,
  max_tokens: 800,
});
```
**Azure-specific points:** you call a **deployment name**, not just a model name. It has an **API version**, regional quota (TPM/RPM), built-in **content filtering**, private networking/VNet, and data stays in your Azure tenant (an enterprise/compliance reason to choose it). Keys live in **Key Vault**, or use Managed Identity.

### 2. Streaming OpenAI responses in Next.js
**Option A: Vercel AI SDK (fastest to build)**
```ts
// app/api/chat/route.ts
import { streamText } from 'ai';
import { createAzure } from '@ai-sdk/azure';
const azure = createAzure({ resourceName: process.env.AZURE_RESOURCE!, apiKey: process.env.AZURE_OPENAI_API_KEY! });

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = streamText({
    model: azure(process.env.AZURE_OPENAI_DEPLOYMENT!),
    system: SYSTEM_PROMPT,
    messages,
    onFinish: async ({ text, usage }) => { await saveMessage(text, usage); },  // persist after stream
  });
  return result.toUIMessageStreamResponse();   // (older versions: toDataStreamResponse())
}
// client: const { messages, sendMessage, status } = useChat();
```
**Option B: raw SSE / ReadableStream**
```ts
const stream = await client.chat.completions.create({ ..., stream: true });
const encoder = new TextEncoder();
return new Response(new ReadableStream({
  async start(controller) {
    for await (const chunk of stream) {
      const token = chunk.choices[0]?.delta?.content ?? '';
      if (token) controller.enqueue(encoder.encode(`data: ${JSON.stringify({ token })}\n\n`));
    }
    controller.enqueue(encoder.encode('data: [DONE]\n\n'));
    controller.close();
  },
}), { headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' } });
```
Client side: `fetch` + `res.body.getReader()` + `TextDecoder`, appending tokens to state. Use **AbortController** for a "Stop" button. (`EventSource` only supports GET, so use fetch for POST.)

### 3. Token limit, cost, rate limit
- **Token limit:** count tokens with `tiktoken`. Keep only the last N messages, plus a **running summary** of older ones. Trim RAG context to the top-k chunks. Set `max_tokens` for the output.
- **Cost:** use a smaller/cheaper model for simple tasks (classification, titles) and the big model only when needed. Cache repeated answers (Redis, keyed on a prompt hash, or semantic caching). Keep system prompts short. Log `usage.prompt_tokens/completion_tokens` per user and set **daily quotas per user/plan**.
- **Rate limit (429):** Azure has TPM/RPM per deployment. Handle it with **retry + exponential backoff** (respect the `retry-after` header), a queue (BullMQ) to smooth spikes, per-user rate limits in your API, and fallback to a second deployment/region.

### 4. System vs User prompt
- **System prompt:** sets the **role, rules, tone, format and boundaries**. The developer writes it and the user doesn't see it. *"You are a support assistant for X. Answer only from the provided context. If unsure, say you don't know. Reply in under 120 words, in JSON {answer, sources}."*
- **User prompt:** the actual question/input from the user.
- **Assistant messages:** previous model replies (history) and few-shot examples.
- **Good practices:** clear instructions, delimiters around context (`<context>...</context>`), few-shot examples, ask for structured output (JSON schema), low temperature for factual tasks, version your prompts in code and test them with an eval set.
- **Security:** treat user input and retrieved documents as **data, not instructions**, to defend against prompt injection.

### 5. Handling hallucination
1. **RAG:** answer only from retrieved documents, and say so in the system prompt.
2. **Allow "I don't know":** explicitly instruct the model that it's fine to say so.
3. **Citations:** require source IDs, and verify that the cited chunks exist.
4. **Low temperature** (0–0.3) for factual answers.
5. **Structured output + validation** (Zod / JSON schema). Reject and retry if the output is invalid.
6. **Tool calling** for facts like prices, order status or dates, instead of letting the model guess.
7. **Post-checks:** a second LLM call as a grader, or rule-based checks for critical domains. Human review for high-risk outputs.
8. **Evals:** a test set of questions with expected answers, run on every prompt change.

### 6. Storing chat history in Postgres
```prisma
model Conversation {
  id        String    @id @default(cuid())
  userId    String
  title     String?
  summary   String?            // rolling summary for long chats
  createdAt DateTime  @default(now())
  messages  Message[]
  @@index([userId, createdAt])
}
model Message {
  id             String   @id @default(cuid())
  conversationId String
  role           Role     // SYSTEM | USER | ASSISTANT | TOOL
  content        String
  promptTokens   Int?
  completionTokens Int?
  model          String?
  createdAt      DateTime @default(now())
  conversation   Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  @@index([conversationId, createdAt])
}
enum Role { SYSTEM USER ASSISTANT TOOL }
```
- Save the user message **before** calling the model, and save the assistant message in `onFinish` (after the stream completes). If the stream is aborted, save the partial text with a flag.
- When building context: load the last N messages + `summary`, fit them to the token budget, and update the summary every ~20 messages.
- Cursor pagination for the sidebar/history. Auto-generate the title with a cheap model call.

### 7. RAG — what and how to implement
**RAG = Retrieval-Augmented Generation:** find the relevant pieces of *your* data and give them to the LLM with the question, so it answers from facts instead of memory.
**Ingestion (offline):**
1. Load documents (PDF, docs, DB) → clean the text.
2. **Chunk** them (~500–1000 tokens, 10–20% overlap, split on headings).
3. **Embed** each chunk (`text-embedding-3-small` on Azure).
4. Store vector + text + metadata (docId, page, tenantId) in **pgvector** (Postgres) or Azure AI Search.

**Query (online):**
1. Embed the user question.
2. **Similarity search** (cosine) → top 5–10 chunks. Better: **hybrid search** (vector + keyword/BM25) + a **re-ranker**.
3. Build the prompt: system rules + `<context>` chunks + question.
4. The LLM answers with citations.
```sql
SELECT content, metadata FROM chunks
WHERE tenant_id = $1
ORDER BY embedding <=> $2      -- cosine distance (pgvector)
LIMIT 5;
```
**Mention:** filter by user/tenant permissions, re-index when documents change, and evaluate retrieval quality (did we fetch the right chunk?) separately from answer quality.

### 8. OpenAI API failure / timeout
- **Timeout:** set `timeout` on the client (e.g. 30–60 s) and wire up an `AbortController` (also cancel when the user leaves).
- **Retry** on 429/5xx/network errors with **exponential backoff + jitter** (the SDK's `maxRetries`). **Don't retry** 400 errors (bad request or content filter). Show a clear message instead.
- **Fallback:** a second Azure deployment/region → a smaller model → a cached/canned response ("AI is busy, please retry").
- **Circuit breaker** if Azure is down, so users aren't stuck waiting 60 s on every request.
- **Streaming failure mid-way:** keep the partial text, show a "Regenerate" button, log it.
- **Content filter errors** (Azure returns `content_filter`): handle them gracefully with a user-friendly message.
- Log the latency, the errors and the token usage, and alert on spikes.

