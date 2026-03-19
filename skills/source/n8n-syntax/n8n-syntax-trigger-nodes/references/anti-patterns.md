# Trigger Node Anti-Patterns

> Common mistakes in n8n trigger node development, with correct alternatives.

## AP-1: Single-Wrapped Emit (CRITICAL)

**Wrong** — single array wrapping causes silent failure:

```typescript
// WRONG: Items never reach the workflow
this.emit([{ json: { data: 'hello' } }]);
```

**Correct** — ALWAYS double-wrap:

```typescript
// CORRECT: Outer array = outputs, inner array = items
this.emit([[ { json: { data: 'hello' } } ]]);
```

**Why**: `emit()` expects `INodeExecutionData[][]`. The outer array indexes outputs (most triggers have one output at index 0). The inner array contains the actual items. Single-wrapping means n8n interprets each item as a separate output rather than items within the first output.

---

## AP-2: Missing closeFunction

**Wrong** — interval leaks when workflow is deactivated:

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const id = setInterval(() => {
        this.emit([[ { json: { tick: true } } ]]);
    }, 60_000);

    return {};  // NO closeFunction — interval runs forever
}
```

**Correct** — ALWAYS clean up allocated resources:

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    const id = setInterval(() => {
        this.emit([[ { json: { tick: true } } ]]);
    }, 60_000);

    return {
        closeFunction: async () => {
            clearInterval(id);
        },
    };
}
```

**Why**: Without `closeFunction`, intervals, listeners, and connections persist after workflow deactivation. Over time, this causes memory leaks and ghost executions.

**Exception**: When using `this.helpers.registerCron()`, n8n handles cleanup automatically. No `closeFunction` needed for cron-registered triggers.

---

## AP-3: Missing manualTriggerFunction

**Wrong** — "Test Workflow" does nothing in the editor:

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    // Only sets up production listener
    const id = setInterval(doWork, 60_000);
    return { closeFunction: async () => clearInterval(id) };
}
```

**Correct** — ALWAYS handle manual mode:

```typescript
async trigger(this: ITriggerFunctions): Promise<ITriggerResponse> {
    if (this.getMode() === 'manual') {
        return {
            manualTriggerFunction: async () => {
                doWork();  // Execute once for testing
            },
        };
    }

    const id = setInterval(doWork, 60_000);
    return { closeFunction: async () => clearInterval(id) };
}
```

**Why**: Without `manualTriggerFunction`, clicking "Test Workflow" in the editor will hang indefinitely or produce no output. Users cannot test the trigger during development.

---

## AP-4: Returning Empty Array from poll()

**Wrong** — returning empty array triggers empty execution:

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const items = await fetchItems();
    if (items.length === 0) {
        return [[]];  // WRONG: Creates an execution with no data
    }
    return [items.map(i => ({ json: i }))];
}
```

**Correct** — ALWAYS return `null` when no new data:

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const items = await fetchItems();
    if (items.length === 0) {
        return null;  // CORRECT: No execution created
    }
    return [items.map(i => ({ json: i }))];
}
```

**Why**: Returning `[[]]` or `[]` creates empty workflow executions that consume resources and clutter the execution log. Returning `null` tells n8n "nothing happened" and no execution is created.

---

## AP-5: No Deduplication in Polling Triggers

**Wrong** — processes the same items on every poll:

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const items = await fetchAllItems();  // Gets ALL items every time
    return [items.map(i => ({ json: i }))];
}
```

**Correct** — track last-seen state with static data:

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const staticData = this.getWorkflowStaticData('node');
    const lastId = staticData.lastId as string | undefined;

    const items = await fetchItemsSince(lastId);
    if (items.length === 0) return null;

    staticData.lastId = items[items.length - 1].id;
    return [items.map(i => ({ json: i }))];
}
```

**Why**: Without deduplication, every poll cycle reprocesses all items. This causes duplicate workflow executions, duplicate API calls downstream, and duplicate data in target systems.

---

## AP-6: Missing Webhook Registration Cleanup

**Wrong** — webhook stays registered on external service after deactivation:

```typescript
webhookMethods = {
    default: {
        async checkExists(this: IHookFunctions) { return false; },
        async create(this: IHookFunctions) {
            const url = this.getNodeWebhookUrl('default');
            await registerOnExternalService(url);
            return true;
        },
        async delete(this: IHookFunctions) {
            return true;  // Does nothing — webhook remains on external service
        },
    },
};
```

**Correct** — store webhook ID and delete on deactivation:

```typescript
webhookMethods = {
    default: {
        async checkExists(this: IHookFunctions) {
            const staticData = this.getWorkflowStaticData('node');
            return !!staticData.webhookId;
        },
        async create(this: IHookFunctions) {
            const url = this.getNodeWebhookUrl('default');
            const response = await registerOnExternalService(url);
            // ALWAYS store the ID for later deletion
            const staticData = this.getWorkflowStaticData('node');
            staticData.webhookId = response.id;
            return true;
        },
        async delete(this: IHookFunctions) {
            const staticData = this.getWorkflowStaticData('node');
            if (staticData.webhookId) {
                await deleteFromExternalService(staticData.webhookId);
                delete staticData.webhookId;
            }
            return true;
        },
    },
};
```

**Why**: Orphaned webhooks on external services waste API rate limits, cause confusing 404/410 errors, and may trigger security alerts.

---

## AP-7: Missing inputs: [] on Trigger Node

**Wrong** — trigger node accepts incoming connections:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['trigger'],
    inputs: [NodeConnectionTypes.Main],  // WRONG: Triggers should not have inputs
    outputs: [NodeConnectionTypes.Main],
    // ...
};
```

**Correct** — triggers ALWAYS have empty inputs:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['trigger'],
    inputs: [],                          // CORRECT: No inputs
    outputs: [NodeConnectionTypes.Main],
    // ...
};
```

**Why**: Trigger nodes start workflows — they have no upstream node to receive data from. Setting inputs allows invalid connections in the editor that will never receive data.

---

## AP-8: Missing group: ['trigger']

**Wrong** — node does not appear as trigger in UI:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['input'],          // WRONG: Not recognized as trigger
    // ...
};
```

**Correct** — ALWAYS include 'trigger' in group:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Trigger',
    name: 'myTrigger',
    group: ['trigger'],        // CORRECT: Shows in trigger section of node creator
    // ...
};
```

**Why**: The `group` array determines where the node appears in the UI node creator panel. Without `'trigger'`, the node appears in regular node sections and users cannot find it where expected.

---

## AP-9: Missing polling: true for Poll Triggers

**Wrong** — `poll()` method exists but n8n does not call it:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Polling Trigger',
    name: 'myPollingTrigger',
    group: ['trigger'],
    // polling: true is MISSING
    // ...
};

async poll(this: IPollFunctions) { /* ... */ }
```

**Correct** — ALWAYS set `polling: true` in description:

```typescript
description: INodeTypeDescription = {
    displayName: 'My Polling Trigger',
    name: 'myPollingTrigger',
    group: ['trigger'],
    polling: true,             // REQUIRED: Tells n8n to call poll()
    // ...
};
```

**Why**: Without `polling: true`, n8n does not recognize the node as a polling trigger and will never invoke the `poll()` method, even if it exists on the class.

---

## AP-10: Using this.emit() Inside poll()

**Wrong** — polling triggers do not have `emit()`:

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const items = await fetchItems();
    this.emit([[items]]);  // ERROR: emit() does not exist on IPollFunctions
    return null;
}
```

**Correct** — return data directly from poll():

```typescript
async poll(this: IPollFunctions): Promise<INodeExecutionData[][] | null> {
    const items = await fetchItems();
    return [items.map(i => ({ json: i }))];
}
```

**Why**: `IPollFunctions` does not expose `emit()`. The polling framework collects the return value from `poll()` and handles emission internally. Attempting to call `this.emit()` results in a runtime TypeError.

---

## AP-11: Forgetting webhookResponse for Rejected Requests

**Wrong** — invalid webhook requests still trigger workflow execution:

```typescript
async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const body = this.getBodyData();
    // No validation — all requests trigger the workflow
    return { workflowData: [[ { json: body } ]] };
}
```

**Correct** — validate and reject bad requests without triggering workflow:

```typescript
async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const body = this.getBodyData();
    const headers = this.getHeaderData();

    if (!headers['x-signature']) {
        return {
            webhookResponse: { status: 'error', message: 'Missing signature' },
            // workflowData is undefined — workflow does NOT execute
        };
    }

    return { workflowData: [[ { json: body } ]] };
}
```

**Why**: Without validation, anyone who knows the webhook URL can trigger workflow executions. ALWAYS validate incoming webhook requests (signatures, tokens, IP allowlists) and return an error response without setting `workflowData` to prevent execution.

---

## AP-12: Sending Response After noWebhookResponse

**Wrong** — double response causes HTTP error:

```typescript
async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const res = this.getResponseObject();
    res.status(200).json({ ok: true });  // Already sent response

    return {
        workflowData: [[ { json: { data: true } } ]],
        webhookResponse: { ok: true },      // WRONG: tries to send ANOTHER response
        // noWebhookResponse should be true
    };
}
```

**Correct** — set `noWebhookResponse: true` when sending manually:

```typescript
async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const res = this.getResponseObject();
    res.status(200).json({ ok: true });

    return {
        workflowData: [[ { json: { data: true } } ]],
        noWebhookResponse: true,  // CORRECT: tells n8n not to send another response
    };
}
```

**Why**: Setting `webhookResponse` when you already sent a response via `getResponseObject()` causes a "headers already sent" error. ALWAYS set `noWebhookResponse: true` when handling the response manually.
