# REQUIREMENTS

## What This Skill Package Must Achieve

### Primary Goal
Enable Claude to write correct, version-aware n8n workflow automation code — including custom node development, workflow JSON structures, expression syntax, and deployment configurations — without hallucinating APIs or using deprecated patterns.

### What Claude Should Do After Loading Skills
1. Recognize n8n context from user requests (workflow creation, node development, automation tasks)
2. Select the correct skill(s) automatically based on the request
3. Write correct TypeScript code for custom nodes and JSON for workflow definitions
4. Avoid known anti-patterns and common AI mistakes with n8n APIs
5. Follow best practices documented in the skill references

### Quality Guarantees
| Guarantee | Description |
|-----------|-------------|
| Version-correct | Code MUST target n8n v1.x APIs |
| API-accurate | All method signatures verified against official docs |
| Node-complete | Custom node skills show full INodeType implementation |
| Anti-pattern-free | Known mistakes are explicitly documented and avoided |
| Deterministic | Skills use ALWAYS/NEVER language, not suggestions |
| Self-contained | Each skill works independently without requiring other skills |

---

## Per-Area Requirements

### 1. Workflow Definition
| Requirement | Detail |
|-------------|--------|
| JSON structure | Correct workflow JSON format with nodes, connections, settings |
| Node configuration | Proper node parameter definitions, type options, defaults |
| Connections | Node connection wiring format, input/output indices |
| Execution order | Trigger nodes, regular nodes, execution flow |
| Critical | Workflow JSON MUST include correct connection format with source/destination indices |

### 2. Custom Node Development
| Requirement | Detail |
|-------------|--------|
| INodeType interface | Complete interface implementation with description, execute method |
| Properties | INodeProperties definitions with correct types, options, displayOptions |
| Credentials | ICredentialType implementation, authentication patterns |
| Node packaging | Package structure, n8n-nodes-base patterns, community node setup |
| Critical | execute() method return type MUST be INodeExecutionData[][] |

### 3. Expressions and Data
| Requirement | Detail |
|-------------|--------|
| Expression syntax | $json, $input, $node, $workflow, $execution references |
| Data transformation | Item-level data manipulation, binary data handling |
| Code node | JavaScript/TypeScript code node patterns, available variables |
| JMESPath | Built-in JMESPath expression support |
| Critical | Expression context — which variables are available in which context |

### 4. Credentials and Authentication
| Requirement | Detail |
|-------------|--------|
| Credential types | OAuth2, API key, basic auth, custom auth patterns |
| ICredentialType | Interface implementation, test methods, property definitions |
| Credential usage | How nodes access credentials, genericCredentialRequest |
| Security | Credential encryption, environment variable patterns |
| Critical | OAuth2 callback URL format and credential test method |

### 5. Execution Model
| Requirement | Detail |
|-------------|--------|
| Trigger types | Webhook, cron, polling, manual triggers |
| Execution modes | Regular, manual, retry, sub-workflow execution |
| Error handling | Try/catch in nodes, continueOnFail, error workflows |
| Queue mode | Redis-based queue mode for scaling |
| Critical | Understanding of item-based execution vs workflow-level execution |

### 6. REST API
| Requirement | Detail |
|-------------|--------|
| Endpoints | Workflow CRUD, execution management, credential operations |
| Authentication | API key authentication, JWT patterns |
| Webhook API | External webhook creation and management |
| Critical | Correct API base URL and authentication header format |

### 7. Deployment and Operations
| Requirement | Detail |
|-------------|--------|
| Docker | Docker Compose configurations, environment variables |
| Environment | Environment variable reference, configuration options |
| Scaling | Queue mode, worker processes, Redis configuration |
| Backup | Workflow export, database backup strategies |
| Critical | Docker Compose MUST include correct volume mounts and environment setup |

### 8. Error Handling and Debugging
| Requirement | Detail |
|-------------|--------|
| Execution errors | Node execution failures, timeout handling |
| Connection errors | API connection failures, retry strategies |
| Expression errors | Common expression mistakes, debugging techniques |
| Workflow errors | Circular dependencies, missing credentials, invalid connections |
| Critical | Error workflow configuration and continueOnFail patterns |

---

## Critical Requirements (apply to ALL skills)

- All TypeScript code MUST be compatible with n8n v1.x node API
- All workflow JSON MUST validate against n8n's workflow format
- Custom node examples MUST implement INodeType correctly
- Credential examples MUST implement ICredentialType correctly
- Code examples MUST be verified against official documentation

---

## Structural Requirements

### Skill Format
- SKILL.md < 500 lines (heavy content in references/)
- YAML frontmatter with name and description (including trigger words)
- English-only content
- Deterministic language (ALWAYS/NEVER, imperative)

### Skill Categories
| Category | Purpose | Must Include |
|----------|---------|--------------|
| syntax/ | How to write it | Node type signatures, expression syntax, JSON format |
| impl/ | How to build it | Decision trees, workflows, step-by-step guides |
| errors/ | How to handle failures | Error patterns, diagnostics, recovery |
| core/ | Cross-cutting | Architecture overview, execution model, concepts |
| agents/ | Orchestration | Validation checklists, auto-detection |

---

## Research Requirements (before creating any skill)

1. Official documentation MUST be consulted and referenced
2. Source code MUST be checked for accuracy when docs are ambiguous
3. Anti-patterns MUST be identified from real issues (GitHub issues, community forum)
4. Code examples MUST be verified (not hallucinated)
5. Version accuracy MUST be confirmed via WebFetch (D-006)

---

## Non-Requirements (explicitly out of scope)

- No n8n v0.x legacy coverage (only v1.x)
- No specific third-party node deep-dives (skills cover the node framework, not individual integrations)
- No GUI tutorials (skills cover code APIs and JSON structures, not UI walkthrough)
- No business process consulting (skills cover technical implementation, not workflow design strategy)
- No comparison with other automation platforms (Zapier, Make, etc.)
