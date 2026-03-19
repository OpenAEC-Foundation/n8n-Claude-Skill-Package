# AI Workflow Anti-Patterns

## AP-01: Missing Chat Model Sub-Node

**Symptom**: Agent or chain node fails with "No language model connected" error.

**Wrong**:
```
[Trigger] → [Tools Agent]  ← No ai_languageModel connected
                └── ai_tool → [Calculator]
```

**Correct**:
```
[Trigger] → [Tools Agent]
                ├── ai_languageModel → [Chat OpenAI]  ← ALWAYS required
                └── ai_tool → [Calculator]
```

**Rule**: EVERY agent and chain root node MUST have a Chat Model sub-node connected to the `ai_languageModel` port.

---

## AP-02: Mismatched Embedding Models in RAG

**Symptom**: Retrieval returns irrelevant results or empty results despite documents being inserted.

**Wrong**:
```
Insertion: [Vector Store Insert] → ai_embedding → [OpenAI text-embedding-ada-002]
Retrieval: [Vector Store Query]  → ai_embedding → [Cohere embed-english-v3.0]
```

**Correct**:
```
Insertion: [Vector Store Insert] → ai_embedding → [OpenAI text-embedding-3-small]
Retrieval: [Vector Store Query]  → ai_embedding → [OpenAI text-embedding-3-small]  ← SAME model
```

**Rule**: ALWAYS use the EXACT same embedding model and provider for both insertion and retrieval. Different models produce incompatible vector spaces.

---

## AP-03: Skipping Text Splitter in RAG Insertion

**Symptom**: Poor retrieval quality, chunks too large, or context window exceeded.

**Wrong**:
```
[Documents] → [Vector Store Insert]
                  ├── ai_embedding → [Embeddings]
                  └── ai_document → [Default Data Loader]  ← No text splitter
```

**Correct**:
```
[Documents] → [Vector Store Insert]
                  ├── ai_embedding → [Embeddings]
                  └── ai_document → [Default Data Loader]
                                       └── ai_textSplitter → [Recursive Character Text Splitter]
                                                                chunkSize: 400
                                                                chunkOverlap: 50
```

**Rule**: ALWAYS attach a text splitter to the document loader. Without splitting, entire documents become single vectors, making retrieval imprecise.

---

## AP-04: Using In-Memory Vector Store in Production

**Symptom**: Vector store data disappears after n8n restart or worker process recycle.

**Wrong**:
```
Production workflow → [In-Memory Vector Store]  ← Data lost on restart
```

**Correct**:
```
Production workflow → [Pinecone / PGVector / Qdrant]  ← Persistent storage
```

**Rule**: NEVER use In-Memory Vector Store in production workflows. It is for prototyping and testing ONLY. ALWAYS use a persistent vector store (Pinecone, PGVector, Qdrant, Supabase, etc.) for any workflow that will run in production.

---

## AP-05: Using Basic LLM Chain When Tools Are Needed

**Symptom**: LLM cannot perform actions, search, or interact with external systems.

**Wrong**:
```
[Trigger] → [Basic LLM Chain]
                ├── ai_languageModel → [Chat Model]
                └── ai_tool → [Wikipedia]  ← Basic LLM Chain CANNOT use tools
```

**Correct**:
```
[Trigger] → [Tools Agent]
                ├── ai_languageModel → [Chat Model]
                └── ai_tool → [Wikipedia]  ← Agent CAN use tools
```

**Rule**: Basic LLM Chain does NOT support tool connections. If you need the AI to use tools, ALWAYS use an Agent node instead.

---

## AP-06: Using OpenAI Functions Agent with Non-OpenAI Models

**Symptom**: Agent fails or produces unexpected results because the model does not support OpenAI function calling format.

**Wrong**:
```
[Trigger] → [OpenAI Functions Agent]
                └── ai_languageModel → [Chat Anthropic]  ← Wrong provider
```

**Correct** (Option A — use Tools Agent):
```
[Trigger] → [Tools Agent]
                └── ai_languageModel → [Chat Anthropic]  ← Works with any provider
```

**Correct** (Option B — use matching provider):
```
[Trigger] → [OpenAI Functions Agent]
                └── ai_languageModel → [Chat OpenAI]  ← Correct provider
```

**Rule**: OpenAI Functions Agent ONLY works with OpenAI chat models. For any other provider, use Tools Agent.

---

## AP-07: No System Prompt in Agent

**Symptom**: Agent behaves unpredictably, uses tools inappropriately, or provides inconsistent responses.

**Wrong**:
```json
{
  "name": "Tools Agent",
  "parameters": {}
}
```

**Correct**:
```json
{
  "name": "Tools Agent",
  "parameters": {
    "options": {
      "systemMessage": "You are a customer support assistant. You can search the knowledge base using the search_docs tool. Always cite your sources. If you don't know the answer, say so."
    }
  }
}
```

**Rule**: ALWAYS provide a system prompt that describes the agent's role, available tools, expected behavior, and constraints.

---

## AP-08: Vague Tool Descriptions

**Symptom**: Agent uses wrong tools, passes incorrect parameters, or never uses a tool that should be used.

**Wrong**:
```json
{
  "name": "my_tool",
  "description": "Does stuff"
}
```

**Correct**:
```json
{
  "name": "search_customer_records",
  "description": "Search the customer database by customer name or email address. Input should be a search query string (e.g., 'john.doe@example.com' or 'John Doe'). Returns customer ID, name, email, and account status."
}
```

**Rule**: Tool `description` MUST clearly explain: (1) what the tool does, (2) what input format is expected, and (3) what output to expect. The AI agent relies on this description to decide when and how to use the tool.

---

## AP-09: Missing Memory in Multi-Turn Conversations

**Symptom**: Agent cannot reference previous messages, repeats questions, or loses context.

**Wrong**:
```
[Chat Trigger] → [Tools Agent]
                     └── ai_languageModel → [Chat Model]
                     ← No memory connected
```

**Correct**:
```
[Chat Trigger] → [Tools Agent]
                     ├── ai_languageModel → [Chat Model]
                     └── ai_memory → [Postgres Chat Memory]  ← Conversation persistence
```

**Rule**: For multi-turn conversations, ALWAYS connect a memory sub-node. For production, ALWAYS use a persistent memory backend (PostgresChat, Redis, Zep) — Simple Memory is lost on restart.

---

## AP-10: Connecting Sub-Nodes to Wrong Connector Types

**Symptom**: Connection rejected in editor, or node fails at runtime with type mismatch error.

**Wrong**: Connecting a memory node to an `ai_tool` connector, or an embedding model to `ai_languageModel`.

**Rule**: Sub-nodes ONLY connect to their matching connector type:
- Chat Models → `ai_languageModel`
- Memory → `ai_memory`
- Tools → `ai_tool`
- Embeddings → `ai_embedding`
- Output Parsers → `ai_outputParser`
- Retrievers → `ai_retriever`
- Text Splitters → `ai_textSplitter`
- Vector Stores → `ai_vectorStore`
- Document Loaders → `ai_document`

---

## AP-11: Not Validating AI Agent Output

**Symptom**: Downstream nodes fail because agent output is unexpected, malformed, or contains hallucinated data.

**Wrong**:
```
[Agent] → [HTTP Request: Create Record]  ← Trusts agent output blindly
```

**Correct**:
```
[Agent] → [IF: Validate Output] → [HTTP Request: Create Record]
              └── Invalid → [Error Handler / Human Review]
```

**Rule**: NEVER trust AI agent output for critical operations without validation. ALWAYS add validation logic (IF node, Code node) between the agent and any destructive/irreversible action.

---

## AP-12: SQL Agent with Write Permissions

**Symptom**: AI agent accidentally modifies or deletes database records.

**Wrong**:
```
[SQL Agent] → connected with read-write database credentials
```

**Correct**:
```
[SQL Agent] → connected with READ-ONLY database credentials
```

**Rule**: ALWAYS use read-only database credentials with the SQL Agent in production. The AI may generate unexpected DELETE, UPDATE, or DROP statements.

---

## AP-13: Ignoring Token Limits in Memory

**Symptom**: API errors due to context window overflow, especially in long conversations.

**Wrong**:
```
[Agent] → ai_memory → [Simple Memory]
                          contextWindowLength: 100  ← Too many exchanges
```

**Correct**:
```
[Agent] → ai_memory → [Token Buffer Memory]
                          maxTokenLimit: 4000  ← Token-aware limit
```

**Rule**: For long-running conversations, use Token Buffer Memory or Summary Memory to prevent context window overflow. Simple Memory with a high `contextWindowLength` can exceed the model's token limit.

---

## AP-14: No Error Handling in AI Workflows

**Symptom**: Workflow fails silently when LLM API is down, rate limited, or returns errors.

**Wrong**:
```
[Trigger] → [Agent] → [Next Step]
```

**Correct**:
```
[Trigger] → [Agent] → [Next Step]
                  └── Error Output → [Notify Admin / Retry Logic]
```

**Rule**: ALWAYS configure error handling on AI nodes. LLM APIs can fail due to rate limits, outages, or token limits. Use the node's error output or workflow-level error handling to handle these gracefully.

---

## AP-15: Hardcoding API Keys in Workflow Parameters

**Symptom**: API keys visible in workflow JSON, shared when exporting workflows.

**Wrong**:
```json
{
  "parameters": {
    "apiKey": "sk-abc123..."
  }
}
```

**Correct**: Use n8n's credential system — credentials are encrypted and stored separately from workflow JSON.

**Rule**: NEVER put API keys, tokens, or secrets directly in node parameters. ALWAYS use n8n's credential system, which encrypts secrets at rest.
