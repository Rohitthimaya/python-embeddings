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
