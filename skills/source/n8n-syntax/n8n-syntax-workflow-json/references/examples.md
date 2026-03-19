# Workflow JSON Examples

## Minimal Workflow (Trigger + One Node)

The simplest valid workflow with a trigger and a single processing node:

```json
{
    "id": "wf_minimal_001",
    "name": "Minimal Workflow",
    "active": false,
    "isArchived": false,
    "nodes": [
        {
            "id": "uuid-trigger-001",
            "name": "Manual Trigger",
            "type": "n8n-nodes-base.manualTrigger",
            "typeVersion": 1,
            "position": [250, 300],
            "parameters": {}
        },
        {
            "id": "uuid-set-001",
            "name": "Set Values",
            "type": "n8n-nodes-base.set",
            "typeVersion": 3.4,
            "position": [450, 300],
            "parameters": {
                "mode": "manual",
                "duplicateItem": false,
                "assignments": {
                    "assignments": [
                        {
                            "id": "assign-001",
                            "name": "greeting",
                            "value": "Hello, World!",
                            "type": "string"
                        }
                    ]
                }
            }
        }
    ],
    "connections": {
        "Manual Trigger": {
            "main": [
                [
                    { "node": "Set Values", "type": "main", "index": 0 }
                ]
            ]
        }
    },
    "settings": {
        "executionOrder": "v1"
    }
}
```

---

## Complete Workflow (Trigger + HTTP + Conditional)

A production workflow with a schedule trigger, HTTP request, and conditional branching:

```json
{
    "id": "wf_complete_001",
    "name": "Hourly API Check",
    "active": true,
    "isArchived": false,
    "nodes": [
        {
            "id": "uuid-schedule-001",
            "name": "Schedule Trigger",
            "type": "n8n-nodes-base.scheduleTrigger",
            "typeVersion": 1.2,
            "position": [250, 300],
            "parameters": {
                "rule": {
                    "interval": [
                        { "field": "hours", "hoursInterval": 1 }
                    ]
                }
            }
        },
        {
            "id": "uuid-http-001",
            "name": "HTTP Request",
            "type": "n8n-nodes-base.httpRequest",
            "typeVersion": 4.2,
            "position": [450, 300],
            "parameters": {
                "method": "GET",
                "url": "https://api.example.com/status",
                "authentication": "none",
                "options": {}
            }
        },
        {
            "id": "uuid-if-001",
            "name": "IF",
            "type": "n8n-nodes-base.if",
            "typeVersion": 2,
            "position": [650, 300],
            "parameters": {
                "conditions": {
                    "options": { "caseSensitive": true },
                    "conditions": [
                        {
                            "id": "cond-001",
                            "leftValue": "={{ $json.status }}",
                            "rightValue": "healthy",
                            "operator": {
                                "type": "string",
                                "operation": "equals"
                            }
                        }
                    ],
                    "combinator": "and"
                }
            }
        },
        {
            "id": "uuid-noop-001",
            "name": "All Good",
            "type": "n8n-nodes-base.noOp",
            "typeVersion": 1,
            "position": [850, 200],
            "parameters": {}
        },
        {
            "id": "uuid-email-001",
            "name": "Send Alert",
            "type": "n8n-nodes-base.emailSend",
            "typeVersion": 2.1,
            "position": [850, 400],
            "parameters": {
                "fromEmail": "alerts@example.com",
                "toEmail": "team@example.com",
                "subject": "API Status Alert",
                "emailType": "text",
                "message": "={{ 'API returned status: ' + $json.status }}"
            },
            "credentials": {
                "smtp": {
                    "id": "cred_smtp_001",
                    "name": "SMTP Credentials"
                }
            }
        }
    ],
    "connections": {
        "Schedule Trigger": {
            "main": [
                [
                    { "node": "HTTP Request", "type": "main", "index": 0 }
                ]
            ]
        },
        "HTTP Request": {
            "main": [
                [
                    { "node": "IF", "type": "main", "index": 0 }
                ]
            ]
        },
        "IF": {
            "main": [
                [
                    { "node": "All Good", "type": "main", "index": 0 }
                ],
                [
                    { "node": "Send Alert", "type": "main", "index": 0 }
                ]
            ]
        }
    },
    "settings": {
        "executionOrder": "v1",
        "timezone": "Europe/Amsterdam",
        "saveDataErrorExecution": "all",
        "saveDataSuccessExecution": "all",
        "errorWorkflow": "wf_error_handler_001"
    }
}
```

### Connection Flow Explained

```
Schedule Trigger → HTTP Request → IF
                                   ├── output 0 (true)  → All Good
                                   └── output 1 (false) → Send Alert
```

---

## Multi-Output Workflow (Switch Node)

Switch nodes route items to different outputs based on conditions:

```json
{
    "id": "wf_switch_001",
    "name": "Route by Category",
    "active": false,
    "isArchived": false,
    "nodes": [
        {
            "id": "uuid-webhook-001",
            "name": "Webhook",
            "type": "n8n-nodes-base.webhook",
            "typeVersion": 2,
            "position": [250, 300],
            "parameters": {
                "path": "incoming",
                "httpMethod": "POST",
                "responseMode": "lastNode"
            },
            "webhookId": "webhook-uuid-001"
        },
        {
            "id": "uuid-switch-001",
            "name": "Route by Type",
            "type": "n8n-nodes-base.switch",
            "typeVersion": 3,
            "position": [450, 300],
            "parameters": {
                "rules": {
                    "rules": [
                        {
                            "outputKey": "Order",
                            "conditions": {
                                "conditions": [
                                    {
                                        "leftValue": "={{ $json.type }}",
                                        "rightValue": "order",
                                        "operator": { "type": "string", "operation": "equals" }
                                    }
                                ]
                            }
                        },
                        {
                            "outputKey": "Refund",
                            "conditions": {
                                "conditions": [
                                    {
                                        "leftValue": "={{ $json.type }}",
                                        "rightValue": "refund",
                                        "operator": { "type": "string", "operation": "equals" }
                                    }
                                ]
                            }
                        }
                    ]
                },
                "options": {
                    "fallbackOutput": "extra"
                }
            }
        },
        {
            "id": "uuid-order-001",
            "name": "Process Order",
            "type": "n8n-nodes-base.noOp",
            "typeVersion": 1,
            "position": [700, 150],
            "parameters": {}
        },
        {
            "id": "uuid-refund-001",
            "name": "Process Refund",
            "type": "n8n-nodes-base.noOp",
            "typeVersion": 1,
            "position": [700, 300],
            "parameters": {}
        },
        {
            "id": "uuid-fallback-001",
            "name": "Unknown Type",
            "type": "n8n-nodes-base.noOp",
            "typeVersion": 1,
            "position": [700, 450],
            "parameters": {}
        }
    ],
    "connections": {
        "Webhook": {
            "main": [
                [
                    { "node": "Route by Type", "type": "main", "index": 0 }
                ]
            ]
        },
        "Route by Type": {
            "main": [
                [
                    { "node": "Process Order", "type": "main", "index": 0 }
                ],
                [
                    { "node": "Process Refund", "type": "main", "index": 0 }
                ],
                [
                    { "node": "Unknown Type", "type": "main", "index": 0 }
                ]
            ]
        }
    },
    "settings": {
        "executionOrder": "v1"
    }
}
```

### Multi-Output Connection Pattern

- Output index 0 → first rule match ("Order")
- Output index 1 → second rule match ("Refund")
- Output index 2 → fallback (no rule matched)

---

## AI Workflow with Typed Connections

AI workflows use connection types OTHER than `"main"` to wire sub-nodes (models, tools, memory) into agent nodes:

```json
{
    "id": "wf_ai_agent_001",
    "name": "AI Agent with Tools",
    "active": false,
    "isArchived": false,
    "nodes": [
        {
            "id": "uuid-chat-trigger-001",
            "name": "Chat Trigger",
            "type": "@n8n/n8n-nodes-langchain.chatTrigger",
            "typeVersion": 1.1,
            "position": [250, 300],
            "parameters": {}
        },
        {
            "id": "uuid-agent-001",
            "name": "AI Agent",
            "type": "@n8n/n8n-nodes-langchain.agent",
            "typeVersion": 1.7,
            "position": [500, 300],
            "parameters": {
                "options": {
                    "systemMessage": "You are a helpful assistant."
                }
            }
        },
        {
            "id": "uuid-model-001",
            "name": "OpenAI Chat Model",
            "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
            "typeVersion": 1.2,
            "position": [500, 500],
            "parameters": {
                "model": "gpt-4",
                "options": {
                    "temperature": 0.7
                }
            },
            "credentials": {
                "openAiApi": {
                    "id": "cred_openai_001",
                    "name": "OpenAI Account"
                }
            }
        },
        {
            "id": "uuid-memory-001",
            "name": "Window Buffer Memory",
            "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
            "typeVersion": 1.3,
            "position": [650, 500],
            "parameters": {
                "sessionIdType": "fromInput",
                "contextWindowLength": 10
            }
        },
        {
            "id": "uuid-tool-001",
            "name": "Calculator",
            "type": "@n8n/n8n-nodes-langchain.toolCalculator",
            "typeVersion": 1,
            "position": [800, 500],
            "parameters": {}
        },
        {
            "id": "uuid-tool-002",
            "name": "HTTP Request Tool",
            "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
            "typeVersion": 1.1,
            "position": [950, 500],
            "parameters": {
                "method": "GET",
                "url": "https://api.example.com/search",
                "description": "Search the example API for information"
            }
        }
    ],
    "connections": {
        "Chat Trigger": {
            "main": [
                [
                    { "node": "AI Agent", "type": "main", "index": 0 }
                ]
            ]
        },
        "OpenAI Chat Model": {
            "ai_languageModel": [
                [
                    { "node": "AI Agent", "type": "ai_languageModel", "index": 0 }
                ]
            ]
        },
        "Window Buffer Memory": {
            "ai_memory": [
                [
                    { "node": "AI Agent", "type": "ai_memory", "index": 0 }
                ]
            ]
        },
        "Calculator": {
            "ai_tool": [
                [
                    { "node": "AI Agent", "type": "ai_tool", "index": 0 }
                ]
            ]
        },
        "HTTP Request Tool": {
            "ai_tool": [
                [
                    { "node": "AI Agent", "type": "ai_tool", "index": 0 }
                ]
            ]
        }
    },
    "settings": {
        "executionOrder": "v1"
    }
}
```

### AI Connection Flow Explained

```
Chat Trigger ──main──> AI Agent
                         ↑
OpenAI Chat Model ──ai_languageModel──┘
Window Buffer Memory ──ai_memory──────┘
Calculator ──ai_tool──────────────────┘
HTTP Request Tool ──ai_tool───────────┘
```

Key observations:
- The trigger connects via `"main"` (standard data flow)
- AI sub-nodes connect via their specific `ai_*` type
- Multiple tools share the same `ai_tool` connection type
- The source node's connection type MUST match the target node's declared input type
- Multiple tools can connect to the same `ai_tool` input at `index: 0` -- the agent accepts multiple tool connections

---

## Fan-Out Pattern (One Node to Many)

Send the same data to multiple downstream nodes in parallel:

```json
{
    "connections": {
        "Trigger": {
            "main": [
                [
                    { "node": "Save to DB", "type": "main", "index": 0 },
                    { "node": "Send Email", "type": "main", "index": 0 },
                    { "node": "Post to Slack", "type": "main", "index": 0 }
                ]
            ]
        }
    }
}
```

All three target nodes receive the same input data and execute in parallel.

---

## Merge Pattern (Many Nodes to One)

Multiple nodes connecting to different inputs of a Merge node:

```json
{
    "connections": {
        "Branch A": {
            "main": [
                [
                    { "node": "Merge", "type": "main", "index": 0 }
                ]
            ]
        },
        "Branch B": {
            "main": [
                [
                    { "node": "Merge", "type": "main", "index": 1 }
                ]
            ]
        }
    }
}
```

Note: "Branch A" connects to input `index: 0` and "Branch B" connects to input `index: 1`. The Merge node uses the input index to distinguish its data sources.
