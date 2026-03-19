# Workflow Design Patterns — Examples

## 1. Sub-Workflow Pattern

### Pattern: Reusable Notification Sub-Workflow

**Sub-workflow** ("Send Notification"):
```
Trigger → IF (channel = slack?) → [true] → Slack node
                                → [false] → Email node
```

**Sub-workflow settings:**
```json
{
    "settings": {
        "callerPolicy": "workflowsFromAList",
        "callerIds": "wf_order_processing,wf_user_signup,wf_daily_report"
    }
}
```

**Caller workflow:**
```json
{
    "nodes": [
        {
            "name": "Process Order",
            "type": "n8n-nodes-base.httpRequest",
            "position": [250, 300]
        },
        {
            "name": "Notify Team",
            "type": "n8n-nodes-base.executeWorkflow",
            "position": [450, 300],
            "parameters": {
                "source": "database",
                "workflowId": "wf_send_notification",
                "mode": "once"
            }
        }
    ],
    "connections": {
        "Process Order": {
            "main": [[{ "node": "Notify Team", "type": "main", "index": 0 }]]
        }
    }
}
```

### Pattern: Per-Item Sub-Workflow Processing

Use `mode: "each"` when the sub-workflow must run independently per item:

```json
{
    "name": "Enrich Each Contact",
    "type": "n8n-nodes-base.executeWorkflow",
    "parameters": {
        "source": "database",
        "workflowId": "wf_contact_enrichment",
        "mode": "each"
    }
}
```

Each input item triggers a separate sub-workflow execution. Results are collected into the output.

### Pattern: Fire-and-Forget Sub-Workflow

Use when the caller does not need the sub-workflow result:

```json
{
    "name": "Queue Background Task",
    "type": "n8n-nodes-base.executeWorkflow",
    "parameters": {
        "source": "database",
        "workflowId": "wf_background_processing",
        "waitForSubWorkflow": false
    }
}
```

---

## 2. Error Handling Patterns

### Pattern 1: Centralized Error Notification

**Error workflow:**
```json
{
    "name": "Error Handler — Slack Alert",
    "nodes": [
        {
            "name": "Error Trigger",
            "type": "n8n-nodes-base.errorTrigger",
            "position": [250, 300]
        },
        {
            "name": "Format Error Message",
            "type": "n8n-nodes-base.set",
            "position": [450, 300],
            "parameters": {
                "assignments": {
                    "assignments": [
                        {
                            "name": "text",
                            "value": "=Workflow '{{ $json.workflow.name }}' failed!\nError: {{ $json.execution.error.message }}\nExecution: {{ $json.execution.url }}",
                            "type": "string"
                        }
                    ]
                }
            }
        },
        {
            "name": "Send Slack Alert",
            "type": "n8n-nodes-base.slack",
            "position": [650, 300],
            "parameters": {
                "channel": "#alerts",
                "text": "={{ $json.text }}"
            }
        }
    ],
    "connections": {
        "Error Trigger": {
            "main": [[{ "node": "Format Error Message", "type": "main", "index": 0 }]]
        },
        "Format Error Message": {
            "main": [[{ "node": "Send Slack Alert", "type": "main", "index": 0 }]]
        }
    }
}
```

**Main workflow settings:**
```json
{
    "settings": {
        "errorWorkflow": "wf_error_handler_slack"
    }
}
```

### Pattern 2: Graceful Degradation with continueOnFail

```json
{
    "nodes": [
        {
            "name": "Fetch External Data",
            "type": "n8n-nodes-base.httpRequest",
            "position": [250, 300],
            "continueOnFail": true,
            "retryOnFail": true,
            "maxTries": 3,
            "waitBetweenTries": 2000,
            "parameters": {
                "method": "GET",
                "url": "https://api.external.com/data"
            }
        },
        {
            "name": "Check for Error",
            "type": "n8n-nodes-base.if",
            "position": [450, 300],
            "parameters": {
                "conditions": {
                    "conditions": [
                        {
                            "leftValue": "={{ $json.error }}",
                            "rightValue": "",
                            "operator": { "type": "string", "operation": "isNotEmpty" }
                        }
                    ],
                    "combinator": "and"
                }
            }
        },
        {
            "name": "Use Cached Data",
            "type": "n8n-nodes-base.set",
            "position": [650, 200],
            "parameters": {
                "assignments": {
                    "assignments": [
                        { "name": "data", "value": "fallback value", "type": "string" },
                        { "name": "source", "value": "cache", "type": "string" }
                    ]
                }
            }
        },
        {
            "name": "Process Fresh Data",
            "type": "n8n-nodes-base.set",
            "position": [650, 400]
        }
    ],
    "connections": {
        "Fetch External Data": {
            "main": [[{ "node": "Check for Error", "type": "main", "index": 0 }]]
        },
        "Check for Error": {
            "main": [
                [{ "node": "Use Cached Data", "type": "main", "index": 0 }],
                [{ "node": "Process Fresh Data", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

### Pattern 3: Validation with Stop And Error

```json
{
    "nodes": [
        {
            "name": "Webhook",
            "type": "n8n-nodes-base.webhook",
            "position": [250, 300]
        },
        {
            "name": "Validate Input",
            "type": "n8n-nodes-base.if",
            "position": [450, 300],
            "parameters": {
                "conditions": {
                    "conditions": [
                        {
                            "leftValue": "={{ $json.body.email }}",
                            "rightValue": "",
                            "operator": { "type": "string", "operation": "isNotEmpty" }
                        },
                        {
                            "leftValue": "={{ $json.body.name }}",
                            "rightValue": "",
                            "operator": { "type": "string", "operation": "isNotEmpty" }
                        }
                    ],
                    "combinator": "and"
                }
            }
        },
        {
            "name": "Stop — Missing Fields",
            "type": "n8n-nodes-base.stopAndError",
            "position": [650, 200],
            "parameters": {
                "errorType": "errorMessage",
                "errorMessage": "=Required fields missing: email={{ $json.body.email ? 'present' : 'MISSING' }}, name={{ $json.body.name ? 'present' : 'MISSING' }}"
            }
        },
        {
            "name": "Process Valid Data",
            "type": "n8n-nodes-base.set",
            "position": [650, 400]
        }
    ],
    "connections": {
        "Webhook": {
            "main": [[{ "node": "Validate Input", "type": "main", "index": 0 }]]
        },
        "Validate Input": {
            "main": [
                [{ "node": "Process Valid Data", "type": "main", "index": 0 }],
                [{ "node": "Stop — Missing Fields", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

---

## 3. Branching Patterns

### Pattern: Multi-Way Routing with Switch

Route orders to different processors based on status:

```json
{
    "name": "Route by Order Status",
    "type": "n8n-nodes-base.switch",
    "position": [450, 300],
    "parameters": {
        "mode": "rules",
        "rules": {
            "values": [
                {
                    "conditions": {
                        "conditions": [{
                            "leftValue": "={{ $json.status }}",
                            "rightValue": "pending",
                            "operator": { "type": "string", "operation": "equals" }
                        }]
                    },
                    "outputKey": "Pending"
                },
                {
                    "conditions": {
                        "conditions": [{
                            "leftValue": "={{ $json.status }}",
                            "rightValue": "processing",
                            "operator": { "type": "string", "operation": "equals" }
                        }]
                    },
                    "outputKey": "Processing"
                },
                {
                    "conditions": {
                        "conditions": [{
                            "leftValue": "={{ $json.status }}",
                            "rightValue": "shipped",
                            "operator": { "type": "string", "operation": "equals" }
                        }]
                    },
                    "outputKey": "Shipped"
                }
            ]
        },
        "fallbackOutput": "extra"
    }
}
```

### Pattern: Parallel Processing with Merge

Fetch data from two APIs in parallel, then combine:

```json
{
    "nodes": [
        {
            "name": "Trigger",
            "type": "n8n-nodes-base.scheduleTrigger",
            "position": [250, 300]
        },
        {
            "name": "Fetch Users",
            "type": "n8n-nodes-base.httpRequest",
            "position": [450, 200],
            "parameters": { "url": "https://api.example.com/users" }
        },
        {
            "name": "Fetch Orders",
            "type": "n8n-nodes-base.httpRequest",
            "position": [450, 400],
            "parameters": { "url": "https://api.example.com/orders" }
        },
        {
            "name": "Join Users + Orders",
            "type": "n8n-nodes-base.merge",
            "position": [650, 300],
            "parameters": {
                "mode": "combine",
                "mergeByFields": {
                    "values": [{ "field1": "id", "field2": "userId" }]
                },
                "joinMode": "keepMatches",
                "outputDataFrom": "both"
            }
        }
    ],
    "connections": {
        "Trigger": {
            "main": [
                [
                    { "node": "Fetch Users", "type": "main", "index": 0 },
                    { "node": "Fetch Orders", "type": "main", "index": 0 }
                ]
            ]
        },
        "Fetch Users": {
            "main": [[{ "node": "Join Users + Orders", "type": "main", "index": 0 }]]
        },
        "Fetch Orders": {
            "main": [[{ "node": "Join Users + Orders", "type": "main", "index": 1 }]]
        }
    }
}
```

---

## 4. Loop Patterns

### Pattern: Rate-Limited API Calls

Process 10 items at a time with a 1-second pause between batches:

```json
{
    "nodes": [
        {
            "name": "Get All Contacts",
            "type": "n8n-nodes-base.httpRequest",
            "position": [250, 300]
        },
        {
            "name": "Loop Over Items",
            "type": "n8n-nodes-base.splitInBatches",
            "position": [450, 300],
            "parameters": {
                "batchSize": 10
            }
        },
        {
            "name": "Enrich Contact",
            "type": "n8n-nodes-base.httpRequest",
            "position": [650, 200],
            "parameters": {
                "url": "=https://api.enrichment.com/contact/{{ $json.email }}"
            }
        },
        {
            "name": "Wait 1 Second",
            "type": "n8n-nodes-base.wait",
            "position": [850, 200],
            "parameters": {
                "resume": "timeInterval",
                "amount": 1,
                "unit": "seconds"
            }
        },
        {
            "name": "Process Results",
            "type": "n8n-nodes-base.set",
            "position": [650, 400]
        }
    ],
    "connections": {
        "Get All Contacts": {
            "main": [[{ "node": "Loop Over Items", "type": "main", "index": 0 }]]
        },
        "Loop Over Items": {
            "main": [
                [{ "node": "Enrich Contact", "type": "main", "index": 0 }],
                [{ "node": "Process Results", "type": "main", "index": 0 }]
            ]
        },
        "Enrich Contact": {
            "main": [[{ "node": "Wait 1 Second", "type": "main", "index": 0 }]]
        },
        "Wait 1 Second": {
            "main": [[{ "node": "Loop Over Items", "type": "main", "index": 0 }]]
        }
    }
}
```

---

## 5. Wait and Resume Patterns

### Pattern: Human Approval via Email

```json
{
    "nodes": [
        {
            "name": "New Purchase Request",
            "type": "n8n-nodes-base.webhook",
            "position": [250, 300]
        },
        {
            "name": "Send Approval Email",
            "type": "n8n-nodes-base.emailSend",
            "position": [450, 300],
            "parameters": {
                "toEmail": "manager@company.com",
                "subject": "=Approval needed: {{ $json.body.item }}",
                "text": "=Please approve this purchase:\n\nItem: {{ $json.body.item }}\nAmount: {{ $json.body.amount }}\n\nApprove: {{ $execution.resumeUrl }}?approved=true\nReject: {{ $execution.resumeUrl }}?approved=false"
            }
        },
        {
            "name": "Wait for Approval",
            "type": "n8n-nodes-base.wait",
            "position": [650, 300],
            "parameters": {
                "resume": "webhook",
                "options": {
                    "responseMode": "onReceived"
                }
            }
        },
        {
            "name": "Check Decision",
            "type": "n8n-nodes-base.if",
            "position": [850, 300],
            "parameters": {
                "conditions": {
                    "conditions": [{
                        "leftValue": "={{ $json.query.approved }}",
                        "rightValue": "true",
                        "operator": { "type": "string", "operation": "equals" }
                    }],
                    "combinator": "and"
                }
            }
        },
        {
            "name": "Process Approved",
            "type": "n8n-nodes-base.set",
            "position": [1050, 200]
        },
        {
            "name": "Send Rejection Notice",
            "type": "n8n-nodes-base.emailSend",
            "position": [1050, 400]
        }
    ],
    "connections": {
        "New Purchase Request": {
            "main": [[{ "node": "Send Approval Email", "type": "main", "index": 0 }]]
        },
        "Send Approval Email": {
            "main": [[{ "node": "Wait for Approval", "type": "main", "index": 0 }]]
        },
        "Wait for Approval": {
            "main": [[{ "node": "Check Decision", "type": "main", "index": 0 }]]
        },
        "Check Decision": {
            "main": [
                [{ "node": "Process Approved", "type": "main", "index": 0 }],
                [{ "node": "Send Rejection Notice", "type": "main", "index": 0 }]
            ]
        }
    }
}
```

---

## 6. Scheduling Patterns

### Pattern: Daily Report at 9 AM

```json
{
    "name": "Daily at 9 AM",
    "type": "n8n-nodes-base.scheduleTrigger",
    "parameters": {
        "rule": {
            "interval": [{
                "field": "cronExpression",
                "expression": "0 9 * * *"
            }]
        }
    }
}
```

### Pattern: Every 15 Minutes During Business Hours

```json
{
    "name": "Business Hours Check",
    "type": "n8n-nodes-base.scheduleTrigger",
    "parameters": {
        "rule": {
            "interval": [{
                "field": "cronExpression",
                "expression": "*/15 9-17 * * 1-5"
            }]
        }
    }
}
```

### Pattern: Workflow-Level Timezone

ALWAYS set timezone in workflow settings for scheduled workflows:

```json
{
    "settings": {
        "timezone": "Europe/Amsterdam"
    }
}
```

---

## 7. Combined Pattern: Complete Order Processing Pipeline

```
Schedule Trigger (every 5 min)
    → Fetch New Orders (HTTP Request)
    → Loop Over Items (batch size: 5)
        → [Loop] → Validate Order (IF)
            → [true] → Enrich Customer (Execute Workflow: wf_customer_enrichment)
                → Process Payment (HTTP Request, retryOnFail: true, maxTries: 3)
            → [false] → Stop And Error ("Invalid order data")
        → Back to Loop Over Items
    → [Done] → Merge Results
    → Send Summary (Slack)
```

Error workflow: Error Trigger → Format Message → Slack Alert + Database Log

This pattern combines scheduling, looping, sub-workflows, validation, retry logic, and error handling into a production-ready pipeline.
