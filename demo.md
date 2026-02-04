# Function Map: How All Functions Connect

This document lists **every significant function** in the codebase: **who calls it**, **what input it gets**, **what it does**, and **what output it returns (and to whom)**.

---

## 1. Entry & App

### `start()` — `src/index.ts`

| | |
|--|--|
| **Called by** | Nobody (entry: `npm run dev` runs this file; `start()` is invoked at load). |
| **Input** | None. Reads `process.env.PORT` (default 3001). |
| **Does** | Calls `connectMongo()`, then `app.listen(PORT, callback)`. On failure, `start().catch()` logs and `process.exit(1)`. |
| **Output** | Nothing returned. Side effects: MongoDB connected, HTTP server listening. Callback logs "Server running on http://localhost:${PORT}". |

---

### `connectMongo()` — `src/db/connect.ts`

| | |
|--|--|
| **Called by** | `start()` in `src/index.ts`. |
| **Input** | None. Uses `process.env.MONGODB_URI`. |
| **Does** | Connects Mongoose to `MONGODB_URI`. Throws if URI missing. |
| **Output** | `Promise<void>`. Resolves when connected. Throws to `start()`’s catch. |

---

### `isMongoConnected()` — `src/db/connect.ts`

| | |
|--|--|
| **Called by** | `app.get("/health", ...)` in `src/app.ts`. |
| **Input** | None. |
| **Does** | Returns whether `mongoose.connection.readyState === 1`. |
| **Output** | `boolean` → used by health handler to set `res.json({ ..., db: "connected" \| "disconnected" })`. |

---

### `app` (Express app) — `src/app.ts`

| | |
|--|--|
| **Called by** | `app.listen(PORT)` in `src/index.ts`. |
| **Input** | N/A (it’s the app; it receives HTTP requests). |
| **Does** | `express.json()`, mounts `conversationsRouter` at `/api/conversations`, `rulesRouter` at `/api/rules`, and `GET /health` which calls `isMongoConnected()` and responds with JSON. |
| **Output** | Express app. Responses go to HTTP clients (browser, frontend, or external systems). |

---

## 2. Schema (validation & data elements)

All schema **exports** are Zod schemas or types. They are **used by** route handlers via `.safeParse(req.body)` or by services. Below are the **functions** in schema.

### `loadRuleDataElements()` — `src/schema/ruleDataElements.ts`

| | |
|--|--|
| **Called by** | `getRuleDataElements()`, `getPathsForRuleType()`, `getSchemaForSystemPrompt()` in same file. |
| **Input** | None. Reads `rules_data_elements.json` from `process.cwd()`. |
| **Does** | Reads and parses JSON, caches in `cached`, returns array of `DataElement`. |
| **Output** | `DataElement[]` → used only inside this module by the three functions above. |

---

### `getRuleDataElements()` — `src/schema/ruleDataElements.ts`

| | |
|--|--|
| **Called by** | Nobody in this codebase (exported for external use). |
| **Input** | None. |
| **Does** | Returns `loadRuleDataElements()`. |
| **Output** | `DataElement[]`. |

---

### `getPathsForRuleType(ruleType)` — `src/schema/ruleDataElements.ts`

| | |
|--|--|
| **Called by** | `validateRule()` in `src/services/ruleEngine.ts`. |
| **Input** | `ruleType: RuleType` (`"provider_selection" \| "quote_evaluation"`). |
| **Does** | Loads data elements, filters by `ruleType`, collects all allowed field paths (e.g. `order.order_value`, `provider_quotes[].fee`) into a Set. |
| **Output** | `Set<string>` → used by `validateRule()` to check that the rule’s `var` paths are allowed. |

---

### `getSchemaForSystemPrompt()` — `src/schema/ruleDataElements.ts`

| | |
|--|--|
| **Called by** | `buildSystemPrompt()` in `src/services/chat.ts`, `buildExtractPrompt()` in `src/services/ruleExtract.ts`. |
| **Input** | None. |
| **Does** | Returns `JSON.stringify(loadRuleDataElements(), null, 2)`. |
| **Output** | `string` (JSON of allowed data fields) → injected into AI system prompts so the model only uses allowed paths. |

---

### Zod schemas (`.safeParse()`) — `src/schema/validation.ts`

Used by routes only. Caller passes `req.body` (or part of it). Output is either `{ success: true, data }` or `{ success: false, error }`.

| Schema | Used in | Input | Output to caller |
|--------|--------|--------|-------------------|
| `createConversationSchema` | `POST /api/conversations` | `req.body` | `parsed.data` → orgId; or 400 with `parsed.error.flatten().fieldErrors`. |
| `sendMessageSchema` | `POST /api/conversations/:id/messages` | `req.body` | `parsed.data` → content; or 400. |
| `previewRuleSchema` | `POST /api/rules/preview` | `req.body` | ruleType, condition; or 400. |
| `createRuleSchema` | `POST /api/rules` | `req.body` | name, description, ruleType, condition, action, enabled, orgId; or 400. |
| `updateRuleSchema` | `PATCH /api/rules/:id` | `req.body` | optional name, description, condition, action, enabled; or 400. |
| `evaluateRequestSchema` | `POST /api/rules/evaluate` | `req.body` | orgId, ruleType, context; or 400. |
| `extractedRuleSchema` | `processExtractedContent()` in ruleExtract.ts | parsed JSON from OpenAI | Validated structured rule or error. |

---

## 3. Models (Mongoose)

Models are used by route handlers. “Input” = what the handler passes; “Output” = what the handler gets back and then sends in the HTTP response.

### `Conversation.create({ orgId, messages, status })` — `src/models/Conversation.ts`

| | |
|--|--|
| **Called by** | `POST /` in `src/routes/conversations.ts`. |
| **Input** | `{ orgId: ObjectId, messages: [], status: "drafting" }`. |
| **Does** | Inserts one conversation document in MongoDB. |
| **Output** | Saved document (with `_id`, timestamps) → handler returns 201 with id, orgId, messages, status, createdAt to **client (App.tsx / api.ts)**. |

---

### `Conversation.findById(id)` — `src/models/Conversation.ts`

| | |
|--|--|
| **Called by** | `POST /:id/messages`, `GET /:id`, `POST /:id/generate-rule` in `src/routes/conversations.ts`. |
| **Input** | Conversation id from `req.params.id`. |
| **Does** | Queries MongoDB for that conversation. |
| **Output** | Document or null → handler returns 404 if null; otherwise uses it (messages, etc.) and responds to **client**. |

---

### `Conversation.findByIdAndDelete(id)` — not used in this codebase (Rule uses it).

### `Rule.find(filter)`, `Rule.findById(id)`, `Rule.create(...)`, `Rule.findByIdAndDelete(id)` — `src/models/Rule.ts`

| Method | Called by | Input | Output to |
|--------|-----------|--------|-----------|
| `Rule.find({ orgId, enabled?, ruleType }).sort(...)` | `POST /api/rules/evaluate`, `GET /api/rules` in rules.ts | orgId (ObjectId), optional enabled, ruleType (evaluate); orgId + optional enabled (list) | Route handler → then evaluateRule loop or res.json(rules.map(toRuleResponse)) to **client** or **external caller**. |
| `Rule.findById(id)` | `GET /api/rules/:id`, `PATCH /api/rules/:id`, `DELETE /api/rules/:id` | Rule id from params | Document or null → 404 or toRuleResponse to **client**. |
| `Rule.create({ name, description, ruleType, condition, action, enabled, orgId })` | `POST /api/rules` | Parsed body | New rule document → 201 + toRuleResponse to **client**. |
| `rule.save()` (after Object.assign) | `PATCH /api/rules/:id` | Updated fields | Saved document → res.json(toRuleResponse(rule)) to **client**. |
| `Rule.findByIdAndDelete(id)` | `DELETE /api/rules/:id` | Rule id | Deleted document or null → 204 or 404 to **client**. |

---

## 4. Services

### Chat — `src/services/chat.ts`

#### `buildSystemPrompt()` (internal)

| | |
|--|--|
| **Called by** | `getChatResponse()` in same file. |
| **Input** | None. |
| **Does** | Calls `getSchemaForSystemPrompt()`, replaces `{{SCHEMA}}` in SYSTEM_PROMPT template. |
| **Output** | `string` → passed as system message to OpenAI. |

---

#### `getChatResponse(messages)` — `src/services/chat.ts`

| | |
|--|--|
| **Called by** | `POST /api/conversations/:id/messages` in `src/routes/conversations.ts`. |
| **Input** | `messages: ChatMessage[]` (role + content for each turn). |
| **Does** | Builds system prompt, sends system + messages to `client.chat.completions.create()`, reads first choice content. Throws if no API key or empty content. |
| **Output** | `Promise<string>` (assistant reply) → **conversations route** appends it to conversation, saves, and returns it in the JSON response to **client (App.tsx)**. |

---

### Rule Extract — `src/services/ruleExtract.ts`

#### `buildExtractPrompt()` (internal)

| | |
|--|--|
| **Called by** | `extractRuleFromMessages()` in same file. |
| **Input** | None. |
| **Does** | Gets schema via `getSchemaForSystemPrompt()`, injects into EXTRACT_PROMPT. |
| **Output** | `string` → used as system message for extraction API call. |

---

#### `parseJsonFromResponse(content)` (internal)

| | |
|--|--|
| **Called by** | `processExtractedContent(content)` in same file. |
| **Input** | `content: string` (raw OpenAI response). |
| **Does** | Strips optional markdown code fence, then `JSON.parse`. |
| **Output** | `Record<string, unknown> \| null` → passed to processExtractedContent. |

---

#### `processExtractedContent(content)` (internal)

| | |
|--|--|
| **Called by** | `extractRuleFromMessages()` after OpenAI returns. |
| **Input** | `content: string` (API response body). |
| **Does** | Parses with `parseJsonFromResponse`; if `parsed.error` string, returns `{ ok: false, error }`; validates with `extractedRuleSchema.safeParse(parsed)`; converts with `convertStructuredRuleToJSONLogic(structured)`; builds action and name; returns ExtractedRule. |
| **Output** | `{ ok: true, rule: ExtractedRule } \| { ok: false, error: string }` → to **extractRuleFromMessages**, which returns it to **conversations route**. |

---

#### `extractRuleFromMessages(messages)` — `src/services/ruleExtract.ts`

| | |
|--|--|
| **Called by** | `POST /api/conversations/:id/generate-rule` in `src/routes/conversations.ts`. |
| **Input** | `messages: ChatMessage[]` (full conversation). |
| **Does** | If no OpenAI client, returns `{ ok: false, error }`. Builds extract prompt, sends conversation to OpenAI, gets content; calls `processExtractedContent(content)` which uses **ruleConverter** and Zod. |
| **Output** | `Promise<{ ok: true, rule } \| { ok: false, error }>` → **conversations route** uses it to either return errors or call validateRule/explainRule/testRuleOnSampleData and then respond to **client**. |

---

### Rule Converter — `src/services/ruleConverter.ts`

#### `buildJSONLogicCondition(cond)` (internal)

| | |
|--|--|
| **Called by** | `convertStructuredRuleToJSONLogic()` in same file. |
| **Input** | `cond: { field, operator, value }` (one condition). |
| **Does** | Builds JSONLogic node: `{ ">": [{ var: field }, value] }` etc. for each operator. |
| **Output** | `Record<string, unknown>` (single condition node) → used inside convertStructuredRuleToJSONLogic. |

---

#### `convertStructuredRuleToJSONLogic(structured)` — `src/services/ruleConverter.ts`

| | |
|--|--|
| **Called by** | `processExtractedContent()` in `src/services/ruleExtract.ts`. |
| **Input** | `structured: ExtractedRuleStructured` (ruleType, conditions[], primaryProvider, fallbackProvider, etc.). |
| **Does** | No conditions → returns `{ var: primaryProvider \| fallbackProvider \| "doordash" }`. One condition → `{ if: [condition, then, else] }`. Multiple → `{ and: [...] }` then same if/else. Uses `buildJSONLogicCondition` for each condition. |
| **Output** | `Record<string, unknown>` (JSONLogic condition) → returned to **processExtractedContent**, which puts it in `rule.condition` and returns to **extractRuleFromMessages** → **conversations route** (generate-rule). Same condition is later validated/explained/tested by ruleEngine. |

---

### Rule Engine — `src/services/ruleEngine.ts`

#### `normalizeArrayIndices(path)` (internal)

| | |
|--|--|
| **Called by** | `validateRule()` in same file. |
| **Input** | `path: string` (e.g. `provider_quotes[0].fee`). |
| **Does** | Replaces `[0]`, `[1]`, etc. with `[]`. |
| **Output** | `string` → used to match schema paths like `provider_quotes[].fee`. |

---

#### `collectVarPaths(node, out)` (internal)

| | |
|--|--|
| **Called by** | `validateRule()` in same file. |
| **Input** | `node`: JSONLogic tree; `out`: Set to mutate. |
| **Does** | Recursively finds all `{ "var": path }` and adds path to `out`. |
| **Output** | void (mutates `out`) → validateRule uses this set to check against allowed paths. |

---

#### `validateRule(condition, ruleType)` — `src/services/ruleEngine.ts`

| | |
|--|--|
| **Called by** | Conversations: `POST /:id/generate-rule`. Rules: `POST /preview`, `POST /`, `PATCH /:id` (when condition in body). |
| **Input** | `condition: Record<string, unknown>` (JSONLogic), `ruleType: RuleType`. |
| **Does** | Gets allowed paths via `getPathsForRuleType(ruleType)`, collects used paths with `collectVarPaths(condition, usedPaths)`, checks each (with `normalizeArrayIndices` for array indices), builds errors list. |
| **Output** | `{ valid: boolean, errors: string[] }` → **route** uses it to return 400 or to continue and call explainRule / testRuleOnSampleData / save. |

---

#### `explainNode(node)` (internal)

| | |
|--|--|
| **Called by** | `explainRule()` (and recursively by itself). |
| **Input** | `node`: one node of JSONLogic tree. |
| **Does** | Recursively turns JSONLogic into readable text (e.g. `if` → "If ... → ..., otherwise → ..."). |
| **Output** | `string` → explainRule returns it. |

---

#### `explainRule(condition)` — `src/services/ruleEngine.ts`

| | |
|--|--|
| **Called by** | Conversations: `POST /:id/generate-rule`. Rules: `POST /preview`, `POST /evaluate` (for matched rule’s reason). |
| **Input** | `condition: Record<string, unknown>` (JSONLogic). |
| **Does** | Returns `explainNode(condition)`. |
| **Output** | `string` (human-readable explanation) → **route** puts it in response body to **client** (preview/generate-rule) or to **external caller** (evaluate). |

---

#### `buildSampleContexts(ruleType)` (internal)

| | |
|--|--|
| **Called by** | `testRuleOnSampleData()` in same file. |
| **Input** | `ruleType: RuleType`. |
| **Does** | Builds 3 sample contexts (order_value 20000, 5000, 15000 cents, etc.). For quote_evaluation adds provider_quotes. |
| **Output** | `Record<string, unknown>[]` → passed to testRuleOnSampleData loop. |

---

#### `testRuleOnSampleData(condition, ruleType)` — `src/services/ruleEngine.ts`

| | |
|--|--|
| **Called by** | Conversations: `POST /:id/generate-rule`. Rules: `POST /preview`. |
| **Input** | `condition` (JSONLogic), `ruleType`. |
| **Does** | Gets sample contexts from `buildSampleContexts(ruleType)`, for each runs `jsonLogic.apply(condition, context)`, builds contextSummary (e.g. "Order value $200.00"), returns array of { contextSummary, context, result }. |
| **Output** | `SampleResult[]` → **route** sends as `sampleResults` in JSON to **client**. |

---

#### `evaluateRule(condition, context)` — `src/services/ruleEngine.ts`

| | |
|--|--|
| **Called by** | `POST /api/rules/evaluate` in `src/routes/rules.ts` (in a loop over each rule). |
| **Input** | `condition` (JSONLogic from rule document), `context: Record<string, unknown>` (real order/quote data from request body). |
| **Does** | `jsonLogic.apply(condition, context)`. |
| **Output** | `unknown` (e.g. `"doordash"` or a quote object) → **rules route** checks if truthy; first truthy wins and route returns result, matchedRule, reason, selectedProvider/selectedQuote to **caller**. |

---

## 5. Routes (handlers)

Handlers receive **input** from `req` (params, query, body) and **output** via `res.json(...)` or `res.status(...).send()` to the **HTTP client** (browser / frontend / external system).

### Conversations — `src/routes/conversations.ts`

| Handler | Method + path | Input from | Calls | Output to client |
|---------|----------------|------------|--------|-------------------|
| POST / | Create conversation | req.body → createConversationSchema | Conversation.create | 201: id, orgId, messages, status, createdAt. 400: validation/invalid orgId. |
| POST /:id/messages | Send message | req.params.id, req.body → sendMessageSchema | Conversation.findById, getChatResponse, conversation.save | 200: message, conversation. 404: not found. 400: validation. 502: chat error. |
| GET /:id | Get conversation | req.params.id | Conversation.findById | 200: id, orgId, messages, status, currentRuleDraft, createdAt, updatedAt. 404/400. |
| POST /:id/generate-rule | Preview rule | req.params.id | Conversation.findById, extractRuleFromMessages, validateRule, explainRule, testRuleOnSampleData | 200: valid, errors, explanation, sampleResults, ruleType, condition, action, name, description. Or valid: false with errors. 404/400. |

### Rules — `src/routes/rules.ts`

| Handler | Method + path | Input from | Calls | Output to client |
|---------|----------------|------------|--------|-------------------|
| POST /preview | Preview rule (no save) | req.body → previewRuleSchema | validateRule, explainRule, testRuleOnSampleData | 200: valid, errors, explanation, sampleResults, condition. 400: validation. |
| POST /evaluate | Runtime evaluate | req.body → evaluateRequestSchema | Rule.find, evaluateRule (loop), explainRule | 200: result, matchedRule, reason, selectedProvider/selectedQuote. Or result: null, reason: "No rule matched". 400: validation. |
| POST / | Create rule | req.body → createRuleSchema | validateRule, Rule.create, toRuleResponse | 201: full rule object. 400: validation/invalid rule. |
| GET / | List rules | req.query.orgId, req.query.enabled | Rule.find, toRuleResponse | 200: array of rule objects. 400: missing/invalid orgId. |
| GET /:id | Get one rule | req.params.id | Rule.findById, toRuleResponse | 200: rule object. 404/400. |
| PATCH /:id | Update rule | req.params.id, req.body → updateRuleSchema | Rule.findById, validateRule (if condition), rule.save, toRuleResponse | 200: rule object. 404/400. |
| DELETE /:id | Delete rule | req.params.id | Rule.findByIdAndDelete | 204. 404/400. |

### `toRuleResponse(rule)` — `src/routes/rules.ts`

| | |
|--|--|
| **Called by** | POST /, GET /, GET /:id, PATCH /:id in rules.ts. |
| **Input** | Mongoose rule document. |
| **Does** | Maps to plain object with id, name, description, ruleType, condition, action, enabled, orgId, createdAt, updatedAt. |
| **Output** | Plain object → passed to `res.json()` to **client**. |

---

## 6. Client (frontend) — `client/src/api.ts`

These functions are **called by** `App.tsx`. They **take** the arguments listed and **output** by returning the parsed JSON or **throwing** an Error (which App shows to the user).

| Function | Input | HTTP call | Output |
|----------|--------|-----------|--------|
| createConversation(orgId) | orgId: string | POST /api/conversations { orgId } | Promise\<Conversation\> → App sets conversation + messages. |
| sendMessage(conversationId, content) | id, content | POST /api/conversations/:id/messages { content } | Promise\<{ message, conversation }\> → App updates messages + conversation. |
| getConversation(conversationId) | id | GET /api/conversations/:id | Promise\<Conversation\> → App. |
| generateRuleFromConversation(conversationId) | id | POST /api/conversations/:id/generate-rule | Promise\<RulePreview\> → App sets rulePreview. |
| createRule(body) | name, description?, ruleType, condition, action?, enabled?, orgId | POST /api/rules | Promise\<Rule\> → App refreshes rules list. |
| getRules(orgId) | orgId | GET /api/rules?orgId= | Promise\<Rule[]\> → App sets rules. |
| deleteRule(id) | id | DELETE /api/rules/:id | Promise\<void\> → App refreshes rules. |

---

## 7. Call Graph (who calls whom)

```
start() [index.ts]
  └── connectMongo()
  └── app.listen()

app.get("/health")
  └── isMongoConnected()

POST /api/conversations
  └── createConversationSchema.safeParse(req.body)
  └── Conversation.create()

POST /api/conversations/:id/messages
  └── sendMessageSchema.safeParse(req.body)
  └── Conversation.findById(id)
  └── getChatResponse(messages)
        └── buildSystemPrompt()
              └── getSchemaForSystemPrompt()
                    └── loadRuleDataElements()
        └── client.chat.completions.create()
  └── conversation.save()

POST /api/conversations/:id/generate-rule
  └── Conversation.findById(id)
  └── extractRuleFromMessages(messages)
        └── buildExtractPrompt()
              └── getSchemaForSystemPrompt()
        └── client.chat.completions.create()
        └── processExtractedContent(content)
              └── parseJsonFromResponse(content)
              └── extractedRuleSchema.safeParse(parsed)
              └── convertStructuredRuleToJSONLogic(structured)
                    └── buildJSONLogicCondition(cond)  [per condition]
  └── validateRule(condition, ruleType)
        └── getPathsForRuleType(ruleType)
              └── loadRuleDataElements()
        └── collectVarPaths(condition, usedPaths)
        └── normalizeArrayIndices(path)  [per path]
  └── explainRule(condition)
        └── explainNode(condition)
  └── testRuleOnSampleData(condition, ruleType)
        └── buildSampleContexts(ruleType)
        └── jsonLogic.apply(condition, context)  [per context]

POST /api/rules/preview
  └── previewRuleSchema.safeParse(req.body)
  └── validateRule(condition, ruleType)
  └── explainRule(condition)
  └── testRuleOnSampleData(condition, ruleType)

POST /api/rules/evaluate
  └── evaluateRequestSchema.safeParse(req.body)
  └── Rule.find({ orgId, enabled, ruleType })
  └── for each rule: evaluateRule(rule.condition, context)
        └── jsonLogic.apply(condition, context)
  └── explainRule(rule.condition)  [when match]

POST /api/rules
  └── createRuleSchema.safeParse(req.body)
  └── validateRule(condition, ruleType)
  └── Rule.create(...)
  └── toRuleResponse(rule)

GET /api/rules
  └── Rule.find(filter)
  └── toRuleResponse(rule)  [each]

GET /api/rules/:id
  └── Rule.findById(id)
  └── toRuleResponse(rule)

PATCH /api/rules/:id
  └── updateRuleSchema.safeParse(req.body)
  └── Rule.findById(id)
  └── validateRule(updates.condition, rule.ruleType)  [if condition in body]
  └── rule.save()
  └── toRuleResponse(rule)

DELETE /api/rules/:id
  └── Rule.findByIdAndDelete(id)
```

---

## 8. Data flow summary

| Flow | Input enters | Functions in order | Output leaves to |
|------|----------------|--------------------|------------------|
| Create conversation | App: createConversation(orgId) | api POST → createConversationSchema → Conversation.create | res.json → App (conversation state). |
| Send message | App: sendMessage(id, content) | api POST → sendMessageSchema → findById → getChatResponse (buildSystemPrompt → getSchemaForSystemPrompt → OpenAI) → save | res.json (message + conversation) → App (messages). |
| Preview rule (from chat) | App: generateRuleFromConversation(id) | api POST generate-rule → findById → extractRuleFromMessages (OpenAI → processExtractedContent → ruleConverter) → validateRule → explainRule → testRuleOnSampleData | res.json (valid, errors, explanation, sampleResults, rule…) → App (rulePreview). |
| Preview rule (raw) | POST /api/rules/preview body | previewRuleSchema → validateRule → explainRule → testRuleOnSampleData | res.json → caller. |
| Save rule | App: createRule(body) | api POST /rules → createRuleSchema → validateRule → Rule.create → toRuleResponse | res.json(rule) → App (then getRules). |
| Runtime evaluate | POST /api/rules/evaluate body | evaluateRequestSchema → Rule.find → for each: evaluateRule(condition, context) → first truthy: explainRule | res.json(result, matchedRule, reason, selectedProvider/selectedQuote) → external system or client. |

This is the full map: **every function**, **how they are connected**, **what they do**, **what input they get**, and **what output they throw/return and to whom**.


Schema layer in detail
The schema layer has two parts:
Data elements (ruleDataElements.ts + rules_data_elements.json) — what fields rules are allowed to use.
Validation (validation.ts) — Zod schemas for HTTP bodies and for AI-extracted rules.
Part 1: Data elements (ruleDataElements.ts)
What it’s for
Defines the allowed field paths for rules (e.g. order.order_value, provider_quotes[].fee).
Used to:
Validate JSONLogic rules (only these paths may appear in var).
Build AI prompts (chat and rule extraction) so the model only references these paths.
Data lives in rules_data_elements.json (project root). The code loads it, caches it, and exposes functions that filter by rule type and format for prompts.
Types
DataElementProperty (one allowed path under an element):
{
  path: string;           // e.g. "order.order_value"
  type: string;           // "number", "string", "boolean"
  description: string;
  commonValues?: string[];  // e.g. ["CA", "NY"] for state
  allowedValues?: string[]; // e.g. ["doordash", "uber"]
  note?: string;
}
{  path: string;           // e.g. "order.order_value"  type: string;           // "number", "string", "boolean"  description: string;  commonValues?: string[];  // e.g. ["CA", "NY"] for state  allowedValues?: string[]; // e.g. ["doordash", "uber"]  note?: string;}
DataElement (one “group” of fields, e.g. order, addresses, quotes):
{
  name: string;              // e.g. "order", "order.dropoff_address"
  type: string;             // "object", "array", "number"
  description: string;
  availableIn: RuleType[];   // ["provider_selection", "quote_evaluation"] or just one
  properties?: DataElementProperty[];  // the actual paths
  nullable?: boolean;
  note?: string;
}
{  name: string;              // e.g. "order", "order.dropoff_address"  type: string;             // "object", "array", "number"  description: string;  availableIn: RuleType[];   // ["provider_selection", "quote_evaluation"] or just one  properties?: DataElementProperty[];  // the actual paths  nullable?: boolean;  note?: string;}
So the allowed paths are either:
Each DataElementProperty.path when properties exists, or
The element’s name when there are no properties (e.g. a top-level field).
Functions in ruleDataElements.ts
1. loadRuleDataElements(): DataElement[]
Input: None. Reads process.cwd() + "rules_data_elements.json".
Does:
If already in memory (cached), returns that.
Otherwise fs.readFileSync + JSON.parse, stores in cached, returns array.
Output: Full list of data elements (all groups and their paths).
Used by: getRuleDataElements(), getPathsForRuleType(), getSchemaForSystemPrompt() in the same file.
2. getRuleDataElements(): DataElement[]
Input: None.
Does: return loadRuleDataElements().
Output: Same as loadRuleDataElements().
Used by: Exported for other code; not used elsewhere in this repo.
3. getPathsForRuleType(ruleType: RuleType): Set<string>
Input: ruleType — "provider_selection" or "quote_evaluation".
Does:
Loads all elements.
For each element, skips it if ruleType is not in availableIn.
If the element has properties, adds each p.path to a Set.
If it has no properties, adds el.name to the Set.
Returns that Set.
Output: Set of allowed paths for that rule type, e.g.
"order.order_value", "order.dropoff_address.administrative_area", "provider_quotes[].fee", "distance_km", etc.
Used by: validateRule() in ruleEngine.ts to ensure a rule’s JSONLogic only uses these paths.
4. getSchemaForSystemPrompt(): string
Input: None.
Does: return JSON.stringify(loadRuleDataElements(), null, 2).
Output: Pretty-printed JSON string of the full data elements (all paths, types, descriptions).
Used by:
buildSystemPrompt() in chat.ts — injected into the Rulebot system prompt so the model only uses these fields.
buildExtractPrompt() in ruleExtract.ts — injected into the “extract rule as JSON” prompt so the extracted field values are only these paths.
So: data elements = single source of truth for “what fields exist”; validation and AI prompts both derive from it.
What’s in rules_data_elements.json
The file is an array of data elements. Each element is either:
A group with properties — then the allowed paths are the path values inside properties.
A single field — then the path is the element’s name.
Examples from your file:
order — properties with paths like order.task_id, order.hub_id, etc.
order.dropoff_address — paths like order.dropoff_address.administrative_area, order.dropoff_address.postal_code.
order.order_value — one property with path order.order_value (number, cents).
distance_km — one property with path distance_km.
provider_quotes — only in quote_evaluation; paths include provider_quotes[].provider, provider_quotes[].fee, provider_quotes[].quote_id, etc.
order.tip, historical_stats.*, failed_delivery_action, etc.
So the schema layer doesn’t “invent” paths; it reads them from this JSON and uses them for validation and prompts.
Part 2: Validation (validation.ts)
This file uses Zod to define:
Request body shapes for the API (so invalid bodies are rejected with clear errors).
The shape of the “structured rule” that the AI returns (so we can safely convert it to JSONLogic).
You don’t “call” schemas like functions; you call .safeParse(value) on them. That returns either { success: true, data } or { success: false, error }. Routes and ruleExtract use this to validate input and AI output.
Schemas and what they validate
1. ruleTypeSchema
Definition: z.enum(["provider_selection", "quote_evaluation"]).
Validates: A string that must be exactly "provider_selection" or "quote_evaluation".
Used in: createRuleSchema, updateRuleSchema (via ruleType), evaluateRequestSchema, previewRuleSchema, extractedRuleSchema.
Exported type: RuleType.
2. conversationMessageSchema
Definition:
role: "user" | "assistant";
content: non-empty string.
Validates: One message in a conversation (used when we validate or type conversation messages; not used on a single route body in this app, but types the structure).
Exported type: ConversationMessage.
3. createConversationSchema
Definition: { orgId: z.string().min(1, "orgId is required") }.
Validates: Body of POST /api/conversations.
Used in: conversations.ts — createConversationSchema.safeParse(req.body); on success, parsed.data.orgId is used to create the conversation.
Exported type: CreateConversationBody.
4. sendMessageSchema
Definition: { content: z.string().min(1, "Message content is required") }.
Validates: Body of POST /api/conversations/:id/messages.
Used in: conversations.ts; on success, parsed.data.content is the user message sent to chat.
Exported type: SendMessageBody.
5. confirmRuleSchema
Definition:
name: optional non-empty string;
description: optional string.
Validates: Optional confirmation payload (e.g. name/description when confirming a rule from chat). Not heavily used in current routes but available for the API.
Exported type: ConfirmRuleBody.
6. createRuleSchema
Definition:
name: required, non-empty string.
description: optional string.
ruleType: ruleTypeSchema.
condition: z.record(z.unknown()) — any object (the JSONLogic condition).
action: z.record(z.unknown()).default({}).
enabled: z.boolean().default(true).
orgId: required, non-empty string.
Validates: Body of POST /api/rules (create rule). Does not check that condition only uses allowed paths; that’s done by validateRule() in the route using getPathsForRuleType().
Used in: rules.ts create handler; on success, parsed.data is passed to Rule.create() and to validateRule(condition, ruleType).
Exported type: CreateRuleBody.
7. updateRuleSchema
Definition: All optional — name, description, condition, action, enabled (each with the same types as in create where applicable).
Validates: Body of PATCH /api/rules/:id (partial update). Again, if condition is present, the route calls validateRule(updates.condition, rule.ruleType) separately.
Used in: rules.ts PATCH handler; Object.assign(rule, updates) then rule.save().
Exported type: UpdateRuleBody.
8. evaluateRequestSchema
Definition:
orgId: required, non-empty string.
ruleType: ruleTypeSchema.
context: z.record(z.unknown()) — any object (the real order/quote data).
Validates: Body of POST /api/rules/evaluate (runtime evaluation). Ensures we have org, rule type, and a context object.
Used in: rules.ts evaluate handler; on success, orgId, ruleType, and context are passed to Rule.find() and evaluateRule().
Exported type: EvaluateRequestBody.
9. jsonLogicSchema
Definition: z.record(z.unknown()) — any object. Represents “a JSONLogic expression” at the type level.
Exported type: JsonLogic. Not used with .safeParse() in the codebase; mainly for typing.
10. previewRuleSchema
Definition:
ruleType: ruleTypeSchema.
condition: z.record(z.unknown()).
action: optional z.record(z.unknown()).
Validates: Body of POST /api/rules/preview (preview a rule without saving). Route then calls validateRule(condition, ruleType), explainRule(condition), testRuleOnSampleData(condition, ruleType).
Exported type: PreviewRuleBody.
11. extractedRuleConditionSchema (one condition in the structured rule)
Definition:
field: string (the path, e.g. "order.order_value").
operator: ">" | "<" | ">=" | "<=" | "==" | "!=" | "===" | "!==".
value: number | string | boolean.
Validates: One condition in the array that the AI returns when we ask it to “extract rule as JSON”. Ensures we get a valid path, operator, and value.
Used in: extractedRuleSchema (below).
12. extractedRuleSchema (full structured rule from AI)
Definition:
ruleType: ruleTypeSchema.
conditions: array of extractedRuleConditionSchema.
primaryProvider: optional "doordash" | "uber".
fallbackProvider: optional "doordash" | "uber" | null.
confidence: optional number in [0, 1].
name: optional string.
description: optional string.
Validates: The parsed JSON from the rule-extraction OpenAI call. So we only accept a shape that we can safely convert to JSONLogic (via ruleConverter) and then validate with getPathsForRuleType().
Used in: processExtractedContent() in ruleExtract.ts: extractedRuleSchema.safeParse(parsed); on success, validationResult.data is the structured rule passed to convertStructuredRuleToJSONLogic().
Exported type: ExtractedRuleStructured.
So in short:
Data elements = what paths are allowed and what we tell the AI; ruleDataElements functions expose that (load, get paths by rule type, get schema for prompt).
Validation = Zod schemas that define and validate HTTP request bodies and the AI’s structured rule; they don’t implement business logic, they just enforce shape and basic constraints. The “allowed paths” check is done in ruleEngine.validateRule() using getPathsForRuleType().

