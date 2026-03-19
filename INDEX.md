# n8n Claude Skill Package — Skill Index

## Overview

**21 skills** organized across 5 categories for n8n v1.x workflow automation development.

---

## n8n-core/ (2 skills)

| Skill | Description |
|-------|-------------|
| [n8n-core-architecture](skills/source/n8n-core/n8n-core-architecture/SKILL.md) | Item-based execution model, INodeExecutionData, binary data, paired items, process modes, workflow settings |
| [n8n-core-api](skills/source/n8n-core/n8n-core-api/SKILL.md) | REST API endpoints, X-N8N-API-KEY authentication, cursor-based pagination, workflow/execution/credential management |

## n8n-syntax/ (8 skills)

| Skill | Description |
|-------|-------------|
| [n8n-syntax-workflow-json](skills/source/n8n-syntax/n8n-syntax-workflow-json/SKILL.md) | Workflow JSON structure, IConnections 3-level nesting, NodeConnectionTypes (main + 12 AI types) |
| [n8n-syntax-expressions](skills/source/n8n-syntax/n8n-syntax-expressions/SKILL.md) | All built-in variables ($json, $input, $node, $workflow, $execution), JMESPath, paired items |
| [n8n-syntax-extension-methods](skills/source/n8n-syntax/n8n-syntax-extension-methods/SKILL.md) | 80+ extension methods: string, array, number, object, boolean, DateTime/Luxon, BinaryFile |
| [n8n-syntax-node-types](skills/source/n8n-syntax/n8n-syntax-node-types/SKILL.md) | INodeType interface, INodeProperties (22 types), displayOptions, execute(), declarative nodes |
| [n8n-syntax-credentials](skills/source/n8n-syntax/n8n-syntax-credentials/SKILL.md) | ICredentialType, authenticate (4 injection methods), OAuth2, credential testing |
| [n8n-syntax-code-node](skills/source/n8n-syntax/n8n-syntax-code-node/SKILL.md) | Code node JavaScript/Python, runOnceForAllItems vs runOnceForEachItem, restrictions |
| [n8n-syntax-trigger-nodes](skills/source/n8n-syntax/n8n-syntax-trigger-nodes/SKILL.md) | Three trigger patterns (event/polling/webhook), ITriggerFunctions, webhook lifecycle |
| [n8n-syntax-ai-nodes](skills/source/n8n-syntax/n8n-syntax-ai-nodes/SKILL.md) | AI/LLM cluster nodes, 6 agent types, memory/vector stores, RAG patterns |

## n8n-impl/ (6 skills)

| Skill | Description |
|-------|-------------|
| [n8n-impl-custom-nodes](skills/source/n8n-impl/n8n-impl-custom-nodes/SKILL.md) | Custom node development, n8n-nodes-starter, community packaging, npm publishing |
| [n8n-impl-deployment](skills/source/n8n-impl/n8n-impl-deployment/SKILL.md) | Docker/Docker Compose, 100+ env vars, queue mode, Redis/BullMQ, CLI, backup |
| [n8n-impl-workflow-design](skills/source/n8n-impl/n8n-impl-workflow-design/SKILL.md) | Sub-workflows, error workflows, branching, merge/loop patterns, scheduling |
| [n8n-impl-integrations](skills/source/n8n-impl/n8n-impl-integrations/SKILL.md) | HTTP Request node, OAuth2 flows, pagination, generic credentials |
| [n8n-impl-webhooks](skills/source/n8n-impl/n8n-impl-webhooks/SKILL.md) | Webhook node, test vs production URLs, 4 response modes, Respond to Webhook |
| [n8n-impl-security](skills/source/n8n-impl/n8n-impl-security/SKILL.md) | Encryption, Docker Secrets, task runners, CORS, reverse proxy, audit |

## n8n-errors/ (3 skills)

| Skill | Description |
|-------|-------------|
| [n8n-errors-execution](skills/source/n8n-errors/n8n-errors-execution/SKILL.md) | Execution failures, continueOnFail, retry, error workflows, Stop And Error |
| [n8n-errors-expressions](skills/source/n8n-errors/n8n-errors-expressions/SKILL.md) | Expression errors, undefined references, type mismatches, context restrictions |
| [n8n-errors-connection](skills/source/n8n-errors/n8n-errors-connection/SKILL.md) | API failures, credential errors, timeouts, SSL, webhook URLs, Redis/queue |

## n8n-agents/ (2 skills)

| Skill | Description |
|-------|-------------|
| [n8n-agents-review](skills/source/n8n-agents/n8n-agents-review/SKILL.md) | Validation checklist: JSON structure, connections, expressions, credentials, deployment |
| [n8n-agents-project-scaffolder](skills/source/n8n-agents/n8n-agents-project-scaffolder/SKILL.md) | Generate complete setups: Docker Compose, custom nodes, workflow templates |

---

## Installation

Add to your project's `.claude/settings.json`:

```json
{
  "skills": {
    "n8n": {
      "path": "/path/to/n8n-Claude-Skill-Package/skills/source"
    }
  }
}
```

Or install individual skills by copying the skill directory to your project's `.claude/skills/` folder.
