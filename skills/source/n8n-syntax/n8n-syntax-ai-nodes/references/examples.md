# AI Workflow Examples

## Example 1: Basic Chat Agent with Tools

**Use case**: A chatbot that can search Wikipedia, perform calculations, and maintain conversation history.

### Workflow Structure

```
[Chat Trigger] → [Tools Agent]
                     ├── ai_languageModel → [Chat OpenAI]
                     │                         model: "gpt-4"
                     │                         temperature: 0.7
                     ├── ai_memory → [Simple Memory]
                     │                  contextWindowLength: 10
                     ├── ai_tool → [Calculator]
                     └── ai_tool → [Wikipedia]
```

### Workflow JSON (Key Nodes)

```json
{
  "nodes": [
    {
      "name": "Chat Trigger",
      "type": "@n8n/n8n-nodes-langchain.chatTrigger",
      "typeVersion": 1,
      "position": [200, 300]
    },
    {
      "name": "Tools Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1,
      "position": [450, 300],
      "parameters": {
        "options": {
          "systemMessage": "You are a helpful assistant. You can search Wikipedia and perform calculations."
        }
      }
    },
    {
      "name": "Chat OpenAI",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [450, 500],
      "parameters": {
        "model": "gpt-4",
        "options": {
          "temperature": 0.7
        }
      }
    },
    {
      "name": "Simple Memory",
      "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
      "typeVersion": 1,
      "position": [600, 500],
      "parameters": {
        "contextWindowLength": 10
      }
    }
  ],
  "connections": {
    "Chat Trigger": {
      "main": [[{ "node": "Tools Agent", "type": "main", "index": 0 }]]
    },
    "Chat OpenAI": {
      "ai_languageModel": [[{ "node": "Tools Agent", "type": "ai_languageModel", "index": 0 }]]
    },
    "Simple Memory": {
      "ai_memory": [[{ "node": "Tools Agent", "type": "ai_memory", "index": 0 }]]
    },
    "Calculator": {
      "ai_tool": [[{ "node": "Tools Agent", "type": "ai_tool", "index": 0 }]]
    },
    "Wikipedia": {
      "ai_tool": [[{ "node": "Tools Agent", "type": "ai_tool", "index": 0 }]]
    }
  }
}
```

### Key Points
- Tools Agent is the default choice — works with any model provider
- ALWAYS include a system message describing available capabilities
- Simple Memory with `contextWindowLength: 10` keeps last 10 exchanges

---

## Example 2: RAG Document Insertion Pipeline

**Use case**: Ingest PDF documents into a Pinecone vector store for later retrieval.

### Workflow Structure

```
[Manual Trigger] → [Read Binary File] → [Extract Text] → [Vector Store: Insert Documents]
                                                              ├── ai_embedding → [Embeddings OpenAI]
                                                              │                     model: "text-embedding-3-small"
                                                              └── ai_document → [Default Data Loader]
                                                                                   └── ai_textSplitter → [Recursive Character Text Splitter]
                                                                                                            chunkSize: 400
                                                                                                            chunkOverlap: 50
```

### Configuration Details

**Vector Store node (Insert Documents operation)**:
```json
{
  "name": "Pinecone Insert",
  "type": "@n8n/n8n-nodes-langchain.vectorStorePinecone",
  "parameters": {
    "mode": "insert",
    "pineconeIndex": "my-knowledge-base",
    "options": {
      "pineconeNamespace": "documents"
    }
  }
}
```

**Recursive Character Text Splitter**:
```json
{
  "name": "Text Splitter",
  "type": "@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter",
  "parameters": {
    "chunkSize": 400,
    "chunkOverlap": 50
  }
}
```

### Key Points
- ALWAYS use the same embedding model for insertion and retrieval
- Chunk size 400 with overlap 50 is a good starting point
- ALWAYS use Recursive Character Text Splitter for document ingestion
- Add metadata to chunks for better filtering during retrieval

---

## Example 3: RAG Retrieval via Agent

**Use case**: A chatbot that answers questions by searching a knowledge base stored in Pinecone.

### Workflow Structure

```
[Chat Trigger] → [Tools Agent]
                     ├── ai_languageModel → [Chat OpenAI]
                     │                         model: "gpt-4"
                     ├── ai_memory → [Postgres Chat Memory]
                     │                  connectionString: "{{ $env.PG_CONNECTION }}"
                     │                  sessionIdColumn: "session_id"
                     └── ai_tool → [Vector Store Q&A Tool]
                                       name: "knowledge_base"
                                       description: "Search company documentation"
                                       └── ai_vectorStore → [Pinecone]
                                                               index: "my-knowledge-base"
                                                               └── ai_embedding → [Embeddings OpenAI]
                                                                                     model: "text-embedding-3-small"
```

### Key Points
- Vector Store Q&A Tool wraps the vector store as an agent tool
- The tool `name` and `description` MUST clearly describe what the knowledge base contains — the AI uses this to decide when to search
- ALWAYS use Postgres Chat Memory (or Redis) for production — Simple Memory is lost on restart
- ALWAYS use the SAME embedding model that was used during document insertion

---

## Example 4: RAG Retrieval via Chain (Direct)

**Use case**: Direct question-answering over documents without agent reasoning.

### Workflow Structure

```
[Webhook Trigger] → [Retrieval QA Chain]
                         ├── ai_languageModel → [Chat Anthropic]
                         │                         model: "claude-sonnet-4-20250514"
                         └── ai_retriever → [Vector Store Retriever]
                                                topK: 4
                                                └── ai_vectorStore → [PGVector]
                                                                        └── ai_embedding → [Embeddings OpenAI]
```

### When to Use Chain vs Agent for RAG
- **Retrieval QA Chain**: ALWAYS retrieve from vector store, simpler, fewer tokens, faster
- **Agent with Vector Store Tool**: Agent DECIDES when to search, can combine with other tools, more flexible

---

## Example 5: Custom Code Tool for Agent

**Use case**: Give an agent the ability to call a custom API.

### Custom Code Tool Configuration

```json
{
  "name": "Fetch Customer Data",
  "type": "@n8n/n8n-nodes-langchain.toolCode",
  "parameters": {
    "name": "fetch_customer",
    "description": "Fetch customer data by customer ID. Input should be a customer ID string.",
    "code": "const response = await fetch(`https://api.example.com/customers/${input}`, {\n  headers: { 'Authorization': 'Bearer ' + $env.API_KEY }\n});\nreturn JSON.stringify(await response.json());"
  }
}
```

### Key Points
- Tool `name` MUST be descriptive and snake_case
- Tool `description` MUST clearly explain what the tool does AND what input format it expects
- The AI agent reads the description to decide when and how to use the tool
- Return data as a JSON string for the agent to parse

---

## Example 6: Human-in-the-Loop Approval

**Use case**: An AI agent that requires human approval before executing sensitive operations.

### Workflow Structure

```
[Chat Trigger] → [Tools Agent]
                     ├── ai_languageModel → [Chat OpenAI]
                     └── ai_tool → [Custom Code Tool: Delete Record]
                                       ├── Require Approval → [Slack Notification]
                                       │                          message: "Agent wants to delete {{$tool.parameters.recordId}}"
                                       ├── On Approve → [Execute Delete]
                                       └── On Deny → [Return Denial Message]
```

### Approval Context Variables
- `$tool.name` — The identifier of the tool the AI wants to execute
- `$tool.parameters` — The parameter values the AI determined for the tool call
- `$fromAI()` — Function for dynamic parameter specification in tool nodes

### System Prompt Example
```
You are a helpful assistant that can manage records.
When you use the delete_record tool, a human will review
your request before it executes. Explain why you want to
delete the record so the reviewer has context.
```

### Key Points
- ALWAYS inform the AI about the approval process in the system prompt
- ALWAYS include context in the notification so the reviewer can make an informed decision
- 9 supported notification channels: Chat, Slack, Discord, Telegram, MS Teams, Gmail, WhatsApp, Google Chat, MS Outlook

---

## Example 7: Structured Output with Parser

**Use case**: Extract structured data from unstructured text using an LLM.

### Workflow Structure

```
[Trigger] → [Basic LLM Chain]
                 ├── ai_languageModel → [Chat OpenAI]
                 └── ai_outputParser → [Structured Output Parser]
                                          schema: { name: string, email: string, company: string }
```

### Structured Output Parser Configuration

```json
{
  "name": "Structured Parser",
  "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
  "parameters": {
    "schemaType": "fromJson",
    "jsonSchema": "{\n  \"type\": \"object\",\n  \"properties\": {\n    \"name\": { \"type\": \"string\" },\n    \"email\": { \"type\": \"string\" },\n    \"company\": { \"type\": \"string\" }\n  },\n  \"required\": [\"name\", \"email\"]\n}"
  }
}
```

### Key Points
- Use Auto-fixing Output Parser if the LLM occasionally returns malformed output
- The parser adds formatting instructions to the prompt automatically
- Structured Output Parser enforces a JSON schema on the LLM response

---

## Example 8: Multi-Model Workflow

**Use case**: Use different models for different tasks in the same workflow.

### Workflow Structure

```
[Trigger] → [Classify Text]                      → [IF: Category]
                 ├── ai_languageModel → [Groq]        ├── "summary" → [Summarization Chain]
                 │      (fast, cheap)                  │                   └── ai_languageModel → [Claude]
                 └── ai_outputParser → [Structured]    │                         (high quality)
                                                       └── "translate" → [Basic LLM Chain]
                                                                             └── ai_languageModel → [GPT-4]
```

### Key Points
- Use fast/cheap models (Groq, GPT-3.5) for classification and routing
- Use powerful models (Claude, GPT-4) for complex generation tasks
- Each chain/agent can have its OWN model — they do NOT share
