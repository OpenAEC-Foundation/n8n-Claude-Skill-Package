# AI Node Types, Cluster Architecture, and Connection Types

## Agent Root Nodes

### Conversational Agent
- **Purpose**: General-purpose chat agent with optional tool use
- **Connections**: `ai_languageModel` (required), `ai_memory`, `ai_tool`, `ai_outputParser`
- **Best for**: Simple conversational interfaces, chatbots
- **Limitation**: Less structured than Tools Agent for complex tool orchestration

### OpenAI Functions Agent
- **Purpose**: Uses OpenAI's native function calling API for structured tool invocation
- **Connections**: `ai_languageModel` (required, OpenAI models only), `ai_memory`, `ai_tool`, `ai_outputParser`
- **Best for**: When using OpenAI models and need reliable structured tool calls
- **Limitation**: ONLY works with OpenAI chat models — NEVER use with Anthropic, Google, or other providers

### Plan and Execute Agent
- **Purpose**: First creates a plan of steps, then executes each step sequentially
- **Connections**: `ai_languageModel` (required), `ai_memory`, `ai_tool`, `ai_outputParser`
- **Best for**: Complex multi-step tasks that benefit from upfront planning
- **Trade-off**: Higher token usage due to planning phase, but more reliable for complex tasks

### ReAct Agent
- **Purpose**: Implements Reasoning + Acting pattern — thinks, acts, observes in a loop
- **Connections**: `ai_languageModel` (required), `ai_memory`, `ai_tool`, `ai_outputParser`
- **Best for**: Tasks requiring visible reasoning chains, debugging AI behavior
- **Trade-off**: More verbose output, useful for understanding agent decision-making

### SQL Agent
- **Purpose**: Generates and executes SQL queries against a connected database
- **Connections**: `ai_languageModel` (required), `ai_memory`
- **Best for**: Natural language database querying
- **CRITICAL**: ALWAYS use with read-only database credentials in production

### Tools Agent
- **Purpose**: Most flexible agent — uses any connected tools with any chat model
- **Connections**: `ai_languageModel` (required), `ai_memory`, `ai_tool`, `ai_outputParser`
- **Best for**: Default choice for any agent workflow
- **Recommendation**: ALWAYS start here unless you have a specific reason for another agent type

---

## Chain Root Nodes

### Basic LLM Chain
- **Purpose**: Simple prompt → LLM → response pipeline
- **Connections**: `ai_languageModel` (required), `ai_outputParser`, `ai_memory`
- **Best for**: Single-turn text generation, translation, formatting
- **NEVER** use when tool calling is needed — use an Agent instead

### Summarization Chain
- **Purpose**: Summarize long text or multiple documents
- **Connections**: `ai_languageModel` (required)
- **Best for**: Document summarization, meeting notes condensation

### Retrieval QA Chain (Question and Answer Chain)
- **Purpose**: RAG-based question answering over documents
- **Connections**: `ai_languageModel` (required), `ai_retriever` (required)
- **Best for**: Q&A over document collections, knowledge base queries

---

## Specialized Root Nodes

### Information Extractor
- **Purpose**: Extract structured data from unstructured text
- **Best for**: Parsing invoices, extracting entities, form data extraction

### Text Classifier
- **Purpose**: Categorize text into predefined categories
- **Best for**: Ticket routing, content categorization, intent detection

### Sentiment Analysis
- **Purpose**: Detect emotional tone of text
- **Best for**: Customer feedback analysis, social media monitoring

### LangChain Code
- **Purpose**: Write custom LangChain code for advanced use cases
- **Best for**: Custom chain composition, advanced prompt engineering, unsupported LangChain features

---

## Chat Model Providers (Sub-Nodes)

| Provider | Node Name | Key Features |
|----------|-----------|--------------|
| OpenAI | Chat OpenAI | GPT-4, GPT-3.5, function calling support |
| Anthropic | Chat Anthropic | Claude models, large context windows |
| Azure OpenAI | Chat Azure OpenAI | Enterprise Azure deployment |
| Google Gemini | Chat Google Gemini | Gemini models, multimodal |
| Google Vertex | Chat Google Vertex AI | Enterprise Google Cloud |
| AWS Bedrock | Chat AWS Bedrock | Multiple model providers via AWS |
| Groq | Chat Groq | Ultra-fast inference |
| Ollama | Chat Ollama | Local model hosting, privacy-first |
| Mistral | Chat Mistral | Mistral models, open-weight |
| DeepSeek | Chat DeepSeek | DeepSeek models |
| Cohere | Chat Cohere | Enterprise NLP models |
| OpenRouter | Chat OpenRouter | Multi-provider routing |

---

## Embedding Model Providers (Sub-Nodes)

| Provider | Node Name | Notes |
|----------|-----------|-------|
| OpenAI | Embeddings OpenAI | `text-embedding-ada-002` (general), `text-embedding-3-large` (complex topics) |
| Cohere | Embeddings Cohere | Multilingual support |
| Google | Embeddings Google | Vertex AI embeddings |
| HuggingFace | Embeddings HuggingFace | Open-source models |
| Mistral | Embeddings Mistral | Mistral embedding models |
| Ollama | Embeddings Ollama | Local embedding generation |
| Azure OpenAI | Embeddings Azure OpenAI | Enterprise Azure deployment |

**CRITICAL**: ALWAYS use the SAME embedding model for both insertion and retrieval. Mixing models produces incompatible vector spaces.

---

## Memory Backends (Sub-Nodes)

| Memory Type | Persistence | Key Config | Best For |
|-------------|-------------|------------|----------|
| Simple Memory | In-memory only | `contextWindowLength` (default 5) | Prototyping, stateless chats |
| Chat Memory Manager | Configurable | Advanced settings | Custom memory management |
| MongoDB Chat Memory | MongoDB | Connection string, collection | MongoDB-based applications |
| Redis Chat Memory | Redis | Host, port, session key | High-performance, distributed |
| Postgres Chat Memory | PostgreSQL | Connection, table name | PostgreSQL-based applications |
| Motorhead | Motorhead server | Server URL | Managed memory service |
| Zep | Zep service | API URL, API key | Managed memory with search |
| Xata | Xata database | Database URL, API key | Serverless database memory |

---

## Vector Store Backends (Sub-Nodes)

| Store | Type | Key Features |
|-------|------|--------------|
| In-Memory | Ephemeral | Testing only — data lost on restart |
| Pinecone | Managed cloud | Fully managed, highly scalable |
| Qdrant | Self-hosted/cloud | Open-source, rich filtering |
| Supabase | Managed | Built on PostgreSQL, easy setup |
| PGVector | Self-hosted | PostgreSQL extension, use existing DB |
| Chroma | Self-hosted | Open-source, simple API |
| Weaviate | Self-hosted/cloud | Hybrid search (vector + keyword) |
| Milvus | Self-hosted/cloud | High-performance, GPU-accelerated |
| MongoDB Atlas | Managed | Atlas Vector Search |
| Azure AI Search | Managed | Azure ecosystem integration |
| Redis | Self-hosted/cloud | Fast, in-memory with persistence |

### Vector Store Operations

| Operation | Purpose | Required Sub-Nodes |
|-----------|---------|-------------------|
| **Insert Documents** | Add documents to vector store | `ai_embedding`, `ai_document` |
| **Get Many** | Retrieve similar documents by query | `ai_embedding` |
| **Retrieve (as sub-node)** | Provide retrieval to agents/chains | `ai_embedding` |

---

## Text Splitters (Sub-Nodes)

| Splitter | Strategy | When to Use |
|----------|----------|-------------|
| Character Text Splitter | Fixed character count | Simple, predictable splitting |
| Recursive Character Text Splitter | Recursive by separators (`\n\n`, `\n`, ` `, `""`) | **DEFAULT CHOICE** — respects document structure |
| Token Text Splitter | Token count (model-specific) | When exact token limits matter |

**Configuration guidelines:**
- Chunk size: 200-500 tokens for fine-grained retrieval
- Chunk overlap: 10-20% of chunk size (e.g., 50 tokens for 300-token chunks)
- ALWAYS use Recursive Character Text Splitter unless you have a specific reason not to

---

## Output Parsers (Sub-Nodes)

| Parser | Purpose | Use Case |
|--------|---------|----------|
| Structured Output Parser | Enforce JSON schema on LLM output | When you need structured data (objects, arrays) |
| Auto-fixing Output Parser | Automatically retry and fix parsing errors | When LLM output may not perfectly match schema |
| Item List Output Parser | Parse comma-separated lists | When you need array output from text |

---

## Retrievers (Sub-Nodes)

| Retriever | Strategy | When to Use |
|-----------|----------|-------------|
| Vector Store Retriever | Basic similarity search | Default choice for most RAG |
| MultiQuery Retriever | Generates multiple query variants | When single query may miss relevant docs |
| Contextual Compression Retriever | Compresses/filters retrieved docs | When context window is limited |
| Workflow Retriever | Calls another n8n workflow | When retrieval logic is complex or custom |

---

## Tools (Sub-Nodes)

| Tool | Purpose | Notes |
|------|---------|-------|
| Calculator | Mathematical calculations | Basic arithmetic and math |
| Custom Code Tool | Run custom JS/Python code | Most flexible — write any tool logic |
| SearXNG | Privacy-focused web search | Self-hosted search engine |
| SerpApi | Google search results | Requires SerpApi API key |
| Wikipedia | Wikipedia article lookup | Free, no API key needed |
| Wolfram Alpha | Computational knowledge | Requires Wolfram Alpha API key |
| Vector Store Q&A Tool | Query vector store as agent tool | Wraps RAG as an agent-accessible tool |

---

## NodeConnectionType Constants (Source Code Reference)

```typescript
export const NodeConnectionTypes = {
    AiAgent: 'ai_agent',
    AiChain: 'ai_chain',
    AiDocument: 'ai_document',
    AiEmbedding: 'ai_embedding',
    AiLanguageModel: 'ai_languageModel',
    AiMemory: 'ai_memory',
    AiOutputParser: 'ai_outputParser',
    AiRetriever: 'ai_retriever',
    AiReranker: 'ai_reranker',
    AiTextSplitter: 'ai_textSplitter',
    AiTool: 'ai_tool',
    AiVectorStore: 'ai_vectorStore',
    Main: 'main',
} as const;
```

These are the typed connectors that appear as colored ports on AI nodes in the n8n editor. Sub-nodes ONLY connect to matching connector types on root nodes.

### Node Input Configuration for AI Nodes

```typescript
// Example: Tools Agent node inputs
inputs: [
    NodeConnectionTypes.Main,
    {
        type: NodeConnectionTypes.AiLanguageModel,
        displayName: 'Model',
        required: true,
        maxConnections: 1,
    },
    {
        type: NodeConnectionTypes.AiMemory,
        displayName: 'Memory',
        required: false,
        maxConnections: 1,
    },
    {
        type: NodeConnectionTypes.AiTool,
        displayName: 'Tool',
        required: false,
    },
    {
        type: NodeConnectionTypes.AiOutputParser,
        displayName: 'Output Parser',
        required: false,
        maxConnections: 1,
    },
]
```

### `usableAsTool` Property

Any regular node can be made available as an AI agent tool by setting `usableAsTool: true` in its `INodeTypeDescription`. This allows AI agents to invoke the node as a tool during execution.

```typescript
description: INodeTypeDescription = {
    // ...
    usableAsTool: true,  // Makes this node available as an AI agent tool
    // ...
};
```
