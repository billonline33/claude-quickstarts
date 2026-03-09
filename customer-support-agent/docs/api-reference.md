# API Reference

## POST /api/chat

The only API endpoint. Handles all chat interactions including RAG retrieval and Claude inference.

**File:** `app/api/chat/route.ts`

---

## Request

**Content-Type:** `application/json`

```typescript
{
  messages: Array<{
    role: "user" | "assistant",
    content: string
  }>,
  model: string,           // Claude model ID (e.g. "claude-3-5-sonnet-20241022")
  knowledgeBaseId: string  // Bedrock KB ID, or "NONE" to skip RAG
}
```

**Example:**
```json
{
  "messages": [
    { "role": "user", "content": "How do I reset my password?" }
  ],
  "model": "claude-3-5-sonnet-20241022",
  "knowledgeBaseId": "ABCD1234EF"
}
```

---

## Response

**Content-Type:** `application/json`

```typescript
{
  id: string,                    // UUID for this response
  response: string,              // Main assistant reply (may contain markdown)
  thinking: string,              // Claude's internal reasoning (for LeftSidebar)
  user_mood: "positive" | "negative" | "neutral" | "curious" | "frustrated" | "confused",
  suggested_questions: string[], // 2-3 follow-up questions to display
  matched_categories?: string[], // Matched support categories from categories.json
  redirect_to_agent: {
    should_redirect: boolean,
    reason?: string              // Explanation shown when redirect is triggered
  },
  debug: {
    context_used: boolean        // Whether RAG context was injected into the prompt
  }
}
```

**Example:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "response": "To reset your password, click **Forgot Password** on the login page...",
  "thinking": "The user wants to reset their password. This is a Technical/Account issue.",
  "user_mood": "neutral",
  "suggested_questions": [
    "What if I don't receive the reset email?",
    "How do I change my password once logged in?",
    "Can I use social login instead?"
  ],
  "matched_categories": ["Account", "Technical"],
  "redirect_to_agent": {
    "should_redirect": false
  },
  "debug": {
    "context_used": true
  }
}
```

---

## Response Headers

| Header | Type | Description |
|---|---|---|
| `x-rag-sources` | JSON string | Array of `RAGSource[]` from Bedrock retrieval |
| `X-Debug-Data` | JSON string | Timing metrics for the full request pipeline |

### `x-rag-sources` schema

```typescript
Array<{
  id: string,       // Bedrock chunk ID (from x-amz-bedrock-kb-chunk-id metadata)
  fileName: string, // Derived from S3 URI (last path segment)
  snippet: string,  // Text content of the retrieved chunk
  score: number     // Relevance score 0–1 (>0.6 = green, 0.4–0.6 = yellow, <0.4 = red)
}>
```

### `X-Debug-Data` schema

```typescript
{
  ragDuration: number,       // ms to retrieve from Bedrock
  claudeDuration: number,    // ms for Claude API call
  jsonParseDuration: number, // ms to parse and validate JSON response
  totalDuration: number      // ms end-to-end
}
```

---

## Error Responses

| Status | Cause |
|---|---|
| `500` | Claude API error, Bedrock error, or JSON parse failure |

On failure the route returns a fallback JSON response with `should_redirect: true` and an appropriate error message, rather than a raw HTTP error, so the chat UI degrades gracefully.

---

## Implementation Details

### JSON forcing trick
The API pre-seeds the assistant turn with `{` to make Claude output only valid JSON without preamble. The JSON is then sanitized before parsing to handle embedded newlines.

### RAG retrieval
Configured for top-3 results from Bedrock (`numberOfResults: 3`), but only the highest-scoring result is forwarded in the response header (`ragSources.slice(0, 1)`). All 3 are used in the system prompt context.

### Claude parameters
```typescript
{
  model: <from request>,
  max_tokens: 1000,
  temperature: 0.3,    // Low for consistent, factual support responses
  system: <systemPrompt with RAG context injected>
}
```

### Supported models (as configured in ChatArea.tsx)
| Label | Model ID |
|---|---|
| Claude 3 Haiku | `claude-3-haiku-20240307` |
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` |
| Claude 3.5 Sonnet | `claude-3-5-sonnet-20241022` |
