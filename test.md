# Complete Function-by-Function Guide with Testing
## Every Function Explained + How to Test with Postman

---

# 📚 TABLE OF CONTENTS

1. [Schema Functions](#schema-functions)
2. [Route Handlers](#route-handlers)
3. [Service Functions - Chat](#service-chat)
4. [Service Functions - Rule Extract](#service-rule-extract)
5. [Service Functions - Rule Converter](#service-rule-converter)
6. [Service Functions - Rule Engine](#service-rule-engine)
7. [Model Methods](#model-methods)
8. [Complete Testing Guide](#complete-testing-guide)

---

# SCHEMA FUNCTIONS
## Location: `src/schema/validation.ts`

These functions load and work with the "allowed fields poster" (rules_data_elements.json)

---

## 1️⃣ `loadRuleDataElements()`

**What it does:** Reads the JSON file ONE TIME and caches it in memory

**Input:** None (it finds the file itself)

**Output:** Array of data elements (the full "poster")

**Code Flow:**
```javascript
function loadRuleDataElements() {
  if (cached) return cached;  // Already loaded? Return it
  
  const filePath = path.join(process.cwd(), 'rules_data_elements.json');
  const fileContent = fs.readFileSync(filePath, 'utf-8');
  cached = JSON.parse(fileContent);
  
  return cached;
}
```

**Example Output:**
```json
[
  {
    "name": "order",
    "type": "object",
    "description": "Order information",
    "availableIn": ["provider_selection", "quote_evaluation"],
    "properties": [
      { "path": "order.task_id", "type": "string" },
      { "path": "order.order_value", "type": "number" }
    ]
  },
  {
    "name": "distance_km",
    "type": "number",
    "description": "Distance in kilometers",
    "availableIn": ["provider_selection", "quote_evaluation"]
  }
]
```

**How to test:** You can't test this directly with Postman (it's internal), but you can see its effect in other endpoints

**Called by:**
- `getRuleDataElements()`
- `getPathsForRuleType()`
- `getSchemaForSystemPrompt()`

---

## 2️⃣ `getRuleDataElements()`

**What it does:** Returns the full list of data elements

**Input:** None

**Output:** Same as `loadRuleDataElements()`

**Code Flow:**
```javascript
function getRuleDataElements() {
  return loadRuleDataElements();  // Just a wrapper
}
```

**Purpose:** Public API for getting the schema (not used in current code, but available for future features)

---

## 3️⃣ `getPathsForRuleType(ruleType)`

**What it does:** Gets ONLY the allowed field paths for a specific rule type

**Input:**
- `ruleType`: "provider_selection" OR "quote_evaluation"

**Output:** Set of allowed path strings

**Code Flow:**
```javascript
function getPathsForRuleType(ruleType: "provider_selection" | "quote_evaluation") {
  const elements = loadRuleDataElements();
  const allowedPaths = new Set<string>();
  
  for (const element of elements) {
    // Skip if this element is not available for this rule type
    if (!element.availableIn.includes(ruleType)) continue;
    
    // If element has properties (like order.order_value)
    if (element.properties) {
      for (const prop of element.properties) {
        allowedPaths.add(prop.path);
      }
    } else {
      // Simple element (like distance_km)
      allowedPaths.add(element.name);
    }
  }
  
  return allowedPaths;
}
```

**Example Input #1:**
```javascript
getPathsForRuleType("provider_selection")
```

**Example Output #1:**
```javascript
Set {
  "order.task_id",
  "order.order_value",
  "order.tip",
  "order.dropoff_address.administrative_area",
  "order.dropoff_address.locality",
  "distance_km",
  "failed_delivery_action"
  // ... 56 total paths
}
```

**Example Input #2:**
```javascript
getPathsForRuleType("quote_evaluation")
```

**Example Output #2:**
```javascript
Set {
  "order.task_id",
  "order.order_value",
  // ... all 56 from provider_selection PLUS:
  "provider_quotes",
  "provider_quotes[].provider",
  "provider_quotes[].fee",
  "provider_quotes[].quote_id"
  // ... 64 total paths
}
```

**How to test:** Can't test directly, but it affects validation - see "Testing Rule Validation" section

**Called by:**
- `validateRule()` - to check if a rule uses only allowed fields

---

## 4️⃣ `getSchemaForSystemPrompt()`

**What it does:** Converts the schema to a JSON string for the AI

**Input:** None

**Output:** Formatted JSON string (the "poster" as text for AI to read)

**Code Flow:**
```javascript
function getSchemaForSystemPrompt() {
  const elements = loadRuleDataElements();
  return JSON.stringify(elements, null, 2);  // Pretty-print with 2 spaces
}
```

**Example Output:**
```json
"[
  {
    \"name\": \"order\",
    \"type\": \"object\",
    \"description\": \"Order information\",
    \"availableIn\": [\"provider_selection\", \"quote_evaluation\"],
    \"properties\": [
      { \"path\": \"order.order_value\", \"type\": \"number\" }
    ]
  }
]"
```

**How to test:** This gets included in AI prompts - you can see it by looking at chat responses

**Called by:**
- `buildSystemPrompt()` (in chat.ts)
- `buildExtractPrompt()` (in ruleExtract.ts)

---

# ROUTE HANDLERS
## Location: `src/routes/conversations.ts` and `src/routes/rules.ts`

These are the actual API endpoints you call from Postman

---

## 5️⃣ POST `/api/conversations` - Create Conversation

**What it does:** Starts a new chat session

**Input (req.body):**
```json
{
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Validation:**
1. Checks `orgId` is present and non-empty (createConversationSchema)
2. Checks `orgId` is a valid MongoDB ObjectId

**Process:**
```javascript
// 1. Validate request
const parsed = createConversationSchema.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ error: "..." });

// 2. Check ObjectId format
if (!mongoose.Types.ObjectId.isValid(orgId)) {
  return res.status(400).json({ error: "Invalid orgId format" });
}

// 3. Create conversation in MongoDB
const conversation = await Conversation.create({
  orgId: new mongoose.Types.ObjectId(orgId),
  messages: [],
  status: "drafting"
});

// 4. Return
res.status(201).json(conversation);
```

**Success Output (201):**
```json
{
  "_id": "65b2c3d4e5f6g7h8i9j0k1l2",
  "orgId": "507f1f77bcf86cd799439011",
  "messages": [],
  "status": "drafting",
  "createdAt": "2026-02-04T10:00:00.000Z",
  "updatedAt": "2026-02-04T10:00:00.000Z"
}
```

**Error Output (400):**
```json
{
  "error": "orgId is required"
}
```
OR
```json
{
  "error": "Invalid orgId format"
}
```

### 🧪 POSTMAN TEST:

**Request:**
```
POST http://localhost:3001/api/conversations
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Expected Result:** Status 201, returns conversation with `_id`

**Save to variable:**
```javascript
// In Postman Tests tab:
let response = pm.response.json();
pm.environment.set("conversationId", response._id);
```

**Test Script:**
```javascript
pm.test("Status is 201", function() {
  pm.response.to.have.status(201);
});

pm.test("Response has conversation ID", function() {
  let json = pm.response.json();
  pm.expect(json).to.have.property("_id");
});

pm.test("Messages array is empty", function() {
  let json = pm.response.json();
  pm.expect(json.messages).to.be.an("array").that.is.empty;
});
```

---

## 6️⃣ POST `/api/conversations/:id/messages` - Send Message

**What it does:** Adds a user message and gets AI response

**Input (URL param):**
```
:id = "65b2c3d4e5f6g7h8i9j0k1l2"
```

**Input (req.body):**
```json
{
  "content": "I want orders over $150 to go to DoorDash"
}
```

**Validation:**
1. Check `id` is valid ObjectId
2. Check `content` is non-empty (sendMessageSchema)
3. Check conversation exists in DB

**Process:**
```javascript
// 1. Validate ID
if (!mongoose.Types.ObjectId.isValid(id)) {
  return res.status(400).json({ error: "Invalid conversation ID" });
}

// 2. Find conversation
const conversation = await Conversation.findById(id);
if (!conversation) {
  return res.status(404).json({ error: "Conversation not found" });
}

// 3. Validate message content
const parsed = sendMessageSchema.safeParse(req.body);
if (!parsed.success) return res.status(400).json({ ... });

// 4. Add user message
const userMessage = { role: "user", content: parsed.data.content };
conversation.messages.push(userMessage);

// 5. Get AI response
const assistantContent = await getChatResponse(conversation.messages);

// 6. Add AI message
const assistantMessage = { role: "assistant", content: assistantContent };
conversation.messages.push(assistantMessage);

// 7. Save
await conversation.save();

// 8. Return both messages
res.status(200).json({ message: assistantMessage, conversation });
```

**Success Output (200):**
```json
{
  "message": {
    "role": "assistant",
    "content": "Got it! So orders with order_value > 15000 cents ($150) should go to DoorDash. What about orders under $150?"
  },
  "conversation": {
    "_id": "65b2c3d4e5f6g7h8i9j0k1l2",
    "messages": [
      { "role": "user", "content": "I want orders over $150 to go to DoorDash" },
      { "role": "assistant", "content": "Got it! So orders with..." }
    ],
    "status": "drafting"
  }
}
```

**Calls internally:**
- `getChatResponse()` - see function #13

### 🧪 POSTMAN TEST:

**Request:**
```
POST http://localhost:3001/api/conversations/{{conversationId}}/messages
Content-Type: application/json

{
  "content": "I want orders over $150 to go to DoorDash"
}
```

**Test Script:**
```javascript
pm.test("Status is 200", function() {
  pm.response.to.have.status(200);
});

pm.test("Response has assistant message", function() {
  let json = pm.response.json();
  pm.expect(json.message.role).to.equal("assistant");
  pm.expect(json.message.content).to.be.a("string").that.is.not.empty;
});

pm.test("Conversation has 2 messages", function() {
  let json = pm.response.json();
  pm.expect(json.conversation.messages).to.have.lengthOf(2);
});
```

---

## 7️⃣ GET `/api/conversations/:id` - Get Conversation

**What it does:** Retrieves a conversation with all messages

**Input (URL param):**
```
:id = "65b2c3d4e5f6g7h8i9j0k1l2"
```

**Process:**
```javascript
// 1. Validate ID
if (!mongoose.Types.ObjectId.isValid(id)) {
  return res.status(400).json({ error: "Invalid conversation ID" });
}

// 2. Find conversation
const conversation = await Conversation.findById(id);
if (!conversation) {
  return res.status(404).json({ error: "Conversation not found" });
}

// 3. Return
res.status(200).json(conversation);
```

**Success Output (200):**
```json
{
  "_id": "65b2c3d4e5f6g7h8i9j0k1l2",
  "orgId": "507f1f77bcf86cd799439011",
  "messages": [
    { "role": "user", "content": "I want orders over $150 to go to DoorDash" },
    { "role": "assistant", "content": "Got it! ..." }
  ],
  "status": "drafting",
  "createdAt": "2026-02-04T10:00:00.000Z"
}
```

### 🧪 POSTMAN TEST:

**Request:**
```
GET http://localhost:3001/api/conversations/{{conversationId}}
```

**Test Script:**
```javascript
pm.test("Status is 200", function() {
  pm.response.to.have.status(200);
});

pm.test("Has messages array", function() {
  let json = pm.response.json();
  pm.expect(json.messages).to.be.an("array");
});
```

---

## 8️⃣ POST `/api/conversations/:id/generate-rule` - Generate Rule

**What it does:** Analyzes conversation and creates a rule preview

**Input (URL param):**
```
:id = "65b2c3d4e5f6g7h8i9j0k1l2"
```

**Input (body):** NONE (reads from conversation)

**Process:**
```javascript
// 1. Validate ID and find conversation
const conversation = await Conversation.findById(id);

// 2. Check has messages
if (!conversation.messages || conversation.messages.length === 0) {
  return res.status(400).json({ error: "Conversation has no messages" });
}

// 3. Extract rule from conversation
const extractResult = await extractRuleFromMessages(conversation.messages);

// 4. If extraction failed
if (!extractResult.ok) {
  return res.status(200).json({
    valid: false,
    errors: [extractResult.error]
  });
}

// 5. Validate the rule
const validation = validateRule(
  extractResult.rule.condition,
  extractResult.rule.ruleType
);

// 6. Explain the rule
const explanation = explainRule(extractResult.rule.condition);

// 7. Test on sample data
const sampleResults = testRuleOnSampleData(
  extractResult.rule.condition,
  extractResult.rule.ruleType
);

// 8. Return preview
res.status(200).json({
  valid: validation.valid,
  errors: validation.errors,
  explanation,
  sampleResults,
  rule: extractResult.rule
});
```

**Success Output (200):**
```json
{
  "valid": true,
  "errors": [],
  "explanation": "If order.order_value > 15000 → doordash, otherwise → uber",
  "sampleResults": [
    "Order with value $200.00 → doordash",
    "Order with value $50.00 → uber",
    "Order with value $150.00 → uber"
  ],
  "rule": {
    "ruleType": "provider_selection",
    "condition": {
      "if": [
        { ">": [{ "var": "order.order_value" }, 15000] },
        "doordash",
        "uber"
      ]
    },
    "action": { "provider": "doordash" },
    "name": "Expensive orders to DoorDash",
    "description": "Routes orders over $150 to DoorDash"
  }
}
```

**Error Output (validation failed):**
```json
{
  "valid": false,
  "errors": ["Field 'order.fake_field' is not available for provider_selection"],
  "explanation": "If order.fake_field > 15000 → doordash, otherwise → uber",
  "sampleResults": [],
  "rule": { ... }
}
```

**Calls internally:**
- `extractRuleFromMessages()` - see function #14
- `validateRule()` - see function #19
- `explainRule()` - see function #21
- `testRuleOnSampleData()` - see function #22

### 🧪 POSTMAN TEST:

**Request:**
```
POST http://localhost:3001/api/conversations/{{conversationId}}/generate-rule
```

**Test Script:**
```javascript
pm.test("Status is 200", function() {
  pm.response.to.have.status(200);
});

pm.test("Has valid field", function() {
  let json = pm.response.json();
  pm.expect(json).to.have.property("valid");
});

pm.test("If valid, has rule condition", function() {
  let json = pm.response.json();
  if (json.valid === true) {
    pm.expect(json.rule).to.have.property("condition");
    pm.expect(json.rule).to.have.property("ruleType");
  }
});

pm.test("Has explanation", function() {
  let json = pm.response.json();
  pm.expect(json.explanation).to.be.a("string");
});

// Save rule for next request
if (pm.response.json().valid) {
  pm.environment.set("generatedRule", JSON.stringify(pm.response.json().rule));
}
```

---

## 9️⃣ POST `/api/rules/preview` - Preview Rule (without saving)

**What it does:** Tests a rule without saving it to database

**Input (req.body):**
```json
{
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.order_value" }, 15000] },
      "doordash",
      "uber"
    ]
  }
}
```

**Process:**
```javascript
// 1. Validate schema
const parsed = previewRuleSchema.safeParse(req.body);

// 2. Validate rule logic
const validation = validateRule(parsed.data.condition, parsed.data.ruleType);

// 3. Explain
const explanation = explainRule(parsed.data.condition);

// 4. Test on samples
const sampleResults = testRuleOnSampleData(
  parsed.data.condition,
  parsed.data.ruleType
);

// 5. Return
res.status(200).json({
  valid: validation.valid,
  errors: validation.errors,
  explanation,
  sampleResults,
  condition: parsed.data.condition
});
```

**Success Output (200):**
```json
{
  "valid": true,
  "errors": [],
  "explanation": "If order.order_value > 15000 → doordash, otherwise → uber",
  "sampleResults": [
    "Order with value $200.00 → doordash",
    "Order with value $50.00 → uber",
    "Order with value $150.00 → uber"
  ],
  "condition": { "if": [...] }
}
```

### 🧪 POSTMAN TEST:

**Request:**
```
POST http://localhost:3001/api/rules/preview
Content-Type: application/json

{
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.order_value" }, 15000] },
      "doordash",
      "uber"
    ]
  }
}
```

**Test invalid field:**
```json
{
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.customer_age" }, 18] },
      "doordash",
      "uber"
    ]
  }
}
```

**Expected:** `valid: false`, error about "customer_age" not available

---

## 🔟 POST `/api/rules` - Create Rule

**What it does:** Saves a rule to the database

**Input (req.body):**
```json
{
  "name": "Expensive orders to DoorDash",
  "description": "Routes orders over $150 to DoorDash",
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.order_value" }, 15000] },
      "doordash",
      "uber"
    ]
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Validation:**
1. Schema validation (createRuleSchema)
2. Rule logic validation (validateRule)

**Process:**
```javascript
// 1. Parse and validate schema
const parsed = createRuleSchema.safeParse(req.body);

// 2. Check orgId format
if (!mongoose.Types.ObjectId.isValid(parsed.data.orgId)) {
  return res.status(400).json({ error: "Invalid orgId" });
}

// 3. Validate rule logic
const validation = validateRule(
  parsed.data.condition,
  parsed.data.ruleType
);

if (!validation.valid) {
  return res.status(400).json({
    error: "Rule validation failed",
    details: validation.errors
  });
}

// 4. Create rule in DB
const rule = await Rule.create({
  ...parsed.data,
  orgId: new mongoose.Types.ObjectId(parsed.data.orgId)
});

// 5. Return formatted response
res.status(201).json(toRuleResponse(rule));
```

**Success Output (201):**
```json
{
  "id": "65c3d4e5f6g7h8i9j0k1l2m3",
  "name": "Expensive orders to DoorDash",
  "description": "Routes orders over $150 to DoorDash",
  "ruleType": "provider_selection",
  "condition": { "if": [...] },
  "action": {},
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011",
  "createdAt": "2026-02-04T10:30:00.000Z",
  "updatedAt": "2026-02-04T10:30:00.000Z"
}
```

**Error Output (400):**
```json
{
  "error": "Rule validation failed",
  "details": ["Field 'order.fake_field' is not available for provider_selection"]
}
```

### 🧪 POSTMAN TEST:

**Request:**
```
POST http://localhost:3001/api/rules
Content-Type: application/json

{
  "name": "Expensive orders to DoorDash",
  "description": "Routes orders over $150 to DoorDash",
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.order_value" }, 15000] },
      "doordash",
      "uber"
    ]
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Test Script:**
```javascript
pm.test("Status is 201", function() {
  pm.response.to.have.status(201);
});

pm.test("Response has rule ID", function() {
  let json = pm.response.json();
  pm.expect(json).to.have.property("id");
  pm.environment.set("ruleId", json.id);
});

pm.test("Rule is enabled", function() {
  let json = pm.response.json();
  pm.expect(json.enabled).to.equal(true);
});
```

---

## 1️⃣1️⃣ GET `/api/rules` - List Rules

**What it does:** Gets all rules for an organization

**Input (query params):**
```
?orgId=507f1f77bcf86cd799439011
?enabled=true  (optional)
```

**Process:**
```javascript
// 1. Check orgId
if (!orgId) {
  return res.status(400).json({ error: "orgId is required" });
}

if (!mongoose.Types.ObjectId.isValid(orgId)) {
  return res.status(400).json({ error: "Invalid orgId" });
}

// 2. Build filter
const filter = { orgId: new mongoose.Types.ObjectId(orgId) };
if (enabled !== undefined) {
  filter.enabled = enabled === "true";
}

// 3. Find rules
const rules = await Rule.find(filter).sort({ createdAt: -1 });

// 4. Format and return
res.status(200).json(rules.map(toRuleResponse));
```

**Success Output (200):**
```json
[
  {
    "id": "65c3d4e5f6g7h8i9j0k1l2m3",
    "name": "Expensive orders to DoorDash",
    "ruleType": "provider_selection",
    "enabled": true,
    "createdAt": "2026-02-04T10:30:00.000Z"
  },
  {
    "id": "65d4e5f6g7h8i9j0k1l2m3n4",
    "name": "California orders to Uber",
    "ruleType": "provider_selection",
    "enabled": false,
    "createdAt": "2026-02-03T09:15:00.000Z"
  }
]
```

### 🧪 POSTMAN TEST:

**Request:**
```
GET http://localhost:3001/api/rules?orgId=507f1f77bcf86cd799439011
```

**Request (only enabled):**
```
GET http://localhost:3001/api/rules?orgId=507f1f77bcf86cd799439011&enabled=true
```

**Test Script:**
```javascript
pm.test("Status is 200", function() {
  pm.response.to.have.status(200);
});

pm.test("Response is an array", function() {
  let json = pm.response.json();
  pm.expect(json).to.be.an("array");
});

pm.test("Each rule has required fields", function() {
  let json = pm.response.json();
  json.forEach(rule => {
    pm.expect(rule).to.have.property("id");
    pm.expect(rule).to.have.property("name");
    pm.expect(rule).to.have.property("ruleType");
  });
});
```

---

## 1️⃣2️⃣ POST `/api/rules/evaluate` - Evaluate Rules

**What it does:** Runs enabled rules on real order data to make a decision

**Input (req.body):**
```json
{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "task_id": "task_12345",
      "order_value": 25000,
      "tip": 500,
      "dropoff_address": {
        "administrative_area": "CA",
        "locality": "San Francisco"
      }
    },
    "distance_km": 8.5
  }
}
```

**Process:**
```javascript
// 1. Validate request
const parsed = evaluateRequestSchema.safeParse(req.body);

// 2. Find enabled rules
const rules = await Rule.find({
  orgId: new mongoose.Types.ObjectId(parsed.data.orgId),
  enabled: true,
  ruleType: parsed.data.ruleType
}).sort({ createdAt: 1 });  // Oldest first

// 3. Try each rule
let matchedRule = null;
let result = null;

for (const rule of rules) {
  result = evaluateRule(rule.condition, parsed.data.context);
  if (result) {
    matchedRule = rule;
    break;  // First match wins
  }
}

// 4. If no match
if (!matchedRule) {
  return res.status(200).json({
    result: null,
    matchedRule: null,
    reason: "No rule matched the provided context"
  });
}

// 5. Explain why it matched
const explanation = explainRule(matchedRule.condition);

// 6. Return result
res.status(200).json({
  result,
  matchedRule: toRuleResponse(matchedRule),
  reason: explanation,
  selectedProvider: result  // For provider_selection
});
```

**Success Output (200) - Match Found:**
```json
{
  "result": "doordash",
  "matchedRule": {
    "id": "65c3d4e5f6g7h8i9j0k1l2m3",
    "name": "Expensive orders to DoorDash",
    "ruleType": "provider_selection"
  },
  "reason": "If order.order_value > 15000 → doordash, otherwise → uber",
  "selectedProvider": "doordash"
}
```

**Success Output (200) - No Match:**
```json
{
  "result": null,
  "matchedRule": null,
  "reason": "No rule matched the provided context"
}
```

**Calls internally:**
- `evaluateRule()` - see function #23

### 🧪 POSTMAN TEST:

**Request (High Value - Should Match):**
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "order_value": 25000,
      "tip": 500
    },
    "distance_km": 8.5
  }
}
```

**Expected:** `result: "doordash"`, `selectedProvider: "doordash"`

**Request (Low Value - Should Match Fallback):**
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "order_value": 5000,
      "tip": 100
    },
    "distance_km": 2.0
  }
}
```

**Expected:** `result: "uber"`, `selectedProvider: "uber"`

**Test Script:**
```javascript
pm.test("Status is 200", function() {
  pm.response.to.have.status(200);
});

pm.test("Has result field", function() {
  let json = pm.response.json();
  pm.expect(json).to.have.property("result");
});

pm.test("If result exists, has reason", function() {
  let json = pm.response.json();
  if (json.result !== null) {
    pm.expect(json).to.have.property("reason");
    pm.expect(json.reason).to.be.a("string");
  }
});

pm.test("Result is valid provider or null", function() {
  let json = pm.response.json();
  if (json.result !== null) {
    pm.expect(["doordash", "uber"]).to.include(json.result);
  }
});
```

---

# SERVICE CHAT
## Location: `src/services/chat.ts`

---

## 1️⃣3️⃣ `getChatResponse(messages)`

**What it does:** Sends conversation to OpenAI and gets Rulebot's response

**Input:**
```javascript
[
  { role: "user", content: "I want orders over $150 to go to DoorDash" }
]
```

**Process:**
```javascript
async function getChatResponse(messages) {
  // 1. Build system prompt
  const systemPrompt = buildSystemPrompt();
  
  // 2. Call OpenAI
  const completion = await client.chat.completions.create({
    model: "gpt-4",
    messages: [
      { role: "system", content: systemPrompt },
      ...messages
    ],
    temperature: 0.7
  });
  
  // 3. Extract response
  return completion.choices[0].message.content;
}
```

**Output:**
```javascript
"Got it! So orders with order_value > 15000 cents ($150) should go to DoorDash. What about orders under $150?"
```

**Calls internally:**
- `buildSystemPrompt()` - see function #14

**Called by:**
- POST `/api/conversations/:id/messages` route handler

**How to test:** Use the send message endpoint - you'll see the AI response

---

## 1️⃣4️⃣ `buildSystemPrompt()`

**What it does:** Creates instructions for the AI

**Input:** None

**Process:**
```javascript
function buildSystemPrompt() {
  const schema = getSchemaForSystemPrompt();
  
  return `You are Rulebot, an expert in delivery routing rules.

AVAILABLE DATA FIELDS (use ONLY these paths):
${schema}

IMPORTANT RULES:
- All monetary values (order_value, tip, fee) are in CENTS
- $1 = 100 cents, $150 = 15000 cents
- Ask clarifying questions ONE AT A TIME
- Be conversational and helpful
- Only suggest fields that are in the available list above

Your job is to help users define routing rules by asking questions.`;
}
```

**Output:** A long string with instructions + the full schema

**Example Output:**
```
You are Rulebot, an expert in delivery routing rules.

AVAILABLE DATA FIELDS (use ONLY these paths):
[
  {
    "name": "order",
    "properties": [
      { "path": "order.order_value", "type": "number" }
    ]
  }
]

IMPORTANT RULES:
- All monetary values are in CENTS
...
```

**Calls internally:**
- `getSchemaForSystemPrompt()` - see function #4

**Called by:**
- `getChatResponse()` - function #13

---

# SERVICE RULE EXTRACT
## Location: `src/services/ruleExtract.ts`

---

## 1️⃣5️⃣ `extractRuleFromMessages(messages)`

**What it does:** Main function - analyzes chat and extracts a structured rule

**Input:**
```javascript
[
  { role: "user", content: "Orders over $150 to DoorDash" },
  { role: "assistant", content: "What about under $150?" },
  { role: "user", content: "Under $150 to Uber" }
]
```

**Process:**
```javascript
async function extractRuleFromMessages(messages) {
  // 1. Build extraction prompt
  const systemPrompt = buildExtractPrompt();
  
  // 2. Call OpenAI
  const completion = await client.chat.completions.create({
    model: "gpt-4",
    messages: [
      { role: "system", content: systemPrompt },
      ...messages,
      {
        role: "user",
        content: "Based on our conversation, generate the rule in JSON format."
      }
    ],
    temperature: 0
  });
  
  // 3. Process the response
  const content = completion.choices[0].message.content;
  return processExtractedContent(content);
}
```

**Output (success):**
```javascript
{
  ok: true,
  rule: {
    ruleType: "provider_selection",
    condition: {
      "if": [
        { ">": [{ "var": "order.order_value" }, 15000] },
        "doordash",
        "uber"
      ]
    },
    action: { provider: "doordash" },
    name: "Expensive orders to DoorDash",
    description: "Routes orders over $150 to DoorDash"
  }
}
```

**Output (failure):**
```javascript
{
  ok: false,
  error: "Could not extract valid rule from conversation"
}
```

**Calls internally:**
- `buildExtractPrompt()` - function #16
- `processExtractedContent()` - function #18

**Called by:**
- POST `/api/conversations/:id/generate-rule` route handler

---

## 1️⃣6️⃣ `buildExtractPrompt()`

**What it does:** Creates instructions for rule extraction

**Input:** None

**Process:**
```javascript
function buildExtractPrompt() {
  const schema = getSchemaForSystemPrompt();
  
  return `You are a rule extraction expert.

Analyze the conversation and return ONLY a JSON object with this structure:
{
  "ruleType": "provider_selection" or "quote_evaluation",
  "conditions": [
    { "field": "order.order_value", "operator": ">", "value": 15000 }
  ],
  "primaryProvider": "doordash" or "uber",
  "fallbackProvider": "uber" or "doordash" or null,
  "confidence": 0.0 to 1.0,
  "name": "Rule name",
  "description": "What the rule does"
}

AVAILABLE FIELDS (use ONLY these):
${schema}

CRITICAL:
- All money is in CENTS ($150 = 15000)
- Return ONLY JSON, no markdown, no explanation
- Use ONLY fields from the available list`;
}
```

**Output:** Long string with extraction instructions

**Calls internally:**
- `getSchemaForSystemPrompt()` - function #4

**Called by:**
- `extractRuleFromMessages()` - function #15

---

## 1️⃣7️⃣ `parseJsonFromResponse(content)`

**What it does:** Converts AI's text response to a JavaScript object

**Input:**
```
```json
{
  "ruleType": "provider_selection",
  "conditions": [...]
}
```
```
OR just:
```
{
  "ruleType": "provider_selection",
  "conditions": [...]
}
```

**Process:**
```javascript
function parseJsonFromResponse(content) {
  try {
    // Remove markdown code fences if present
    let cleaned = content.trim();
    
    if (cleaned.startsWith("```json")) {
      cleaned = cleaned.replace(/^```json\s*/i, "");
      cleaned = cleaned.replace(/```\s*$/, "");
    } else if (cleaned.startsWith("```")) {
      cleaned = cleaned.replace(/^```\s*/, "");
      cleaned = cleaned.replace(/```\s*$/, "");
    }
    
    // Parse JSON
    return JSON.parse(cleaned.trim());
  } catch (error) {
    return null;
  }
}
```

**Output (success):**
```javascript
{
  ruleType: "provider_selection",
  conditions: [
    { field: "order.order_value", operator: ">", value: 15000 }
  ],
  primaryProvider: "doordash",
  fallbackProvider: "uber"
}
```

**Output (failure):**
```javascript
null
```

**Called by:**
- `processExtractedContent()` - function #18

---

## 1️⃣8️⃣ `processExtractedContent(content)`

**What it does:** Validates AI's extracted rule and converts to JSONLogic

**Input:** Raw AI response text (string)

**Process:**
```javascript
function processExtractedContent(content) {
  // 1. Parse JSON
  const parsed = parseJsonFromResponse(content);
  
  if (!parsed) {
    return { ok: false, error: "Could not parse JSON from response" };
  }
  
  // 2. Check for error response from AI
  if (parsed.error) {
    return { ok: false, error: parsed.error };
  }
  
  // 3. Validate structure with Zod
  const validationResult = extractedRuleSchema.safeParse(parsed);
  
  if (!validationResult.success) {
    return {
      ok: false,
      error: "Invalid rule structure: " + validationResult.error.message
    };
  }
  
  const structured = validationResult.data;
  
  // 4. Convert to JSONLogic
  const condition = convertStructuredRuleToJSONLogic(structured);
  
  // 5. Build final rule object
  const rule = {
    ruleType: structured.ruleType,
    condition,
    action: { provider: structured.primaryProvider },
    name: structured.name || `Rule for ${structured.ruleType}`,
    description: structured.description
  };
  
  return { ok: true, rule };
}
```

**Output (success):**
```javascript
{
  ok: true,
  rule: {
    ruleType: "provider_selection",
    condition: { "if": [...] },  // JSONLogic format
    action: { provider: "doordash" },
    name: "Expensive orders to DoorDash",
    description: "..."
  }
}
```

**Output (failure):**
```javascript
{
  ok: false,
  error: "Invalid rule structure: conditions must be an array"
}
```

**Calls internally:**
- `parseJsonFromResponse()` - function #17
- `convertStructuredRuleToJSONLogic()` - function #19

**Called by:**
- `extractRuleFromMessages()` - function #15

---

# SERVICE RULE CONVERTER
## Location: `src/services/ruleConverter.ts`

---

## 1️⃣9️⃣ `convertStructuredRuleToJSONLogic(structured)`

**What it does:** Converts human-friendly rule format to JSONLogic

**Input:**
```javascript
{
  ruleType: "provider_selection",
  conditions: [
    { field: "order.order_value", operator: ">", value: 15000 }
  ],
  primaryProvider: "doordash",
  fallbackProvider: "uber"
}
```

**Process:**
```javascript
function convertStructuredRuleToJSONLogic(structured) {
  // For single condition with both providers
  if (structured.conditions.length === 1 && 
      structured.primaryProvider && 
      structured.fallbackProvider) {
    
    const condition = structured.conditions[0];
    const test = buildJSONLogicCondition(condition);
    
    return {
      "if": [
        test,
        structured.primaryProvider,
        structured.fallbackProvider
      ]
    };
  }
  
  // More complex cases would go here...
}
```

**Output:**
```javascript
{
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}
```

**Calls internally:**
- `buildJSONLogicCondition()` - function #20

**Called by:**
- `processExtractedContent()` - function #18

---

## 2️⃣0️⃣ `buildJSONLogicCondition(condition)`

**What it does:** Converts a single condition to JSONLogic format

**Input:**
```javascript
{ field: "order.order_value", operator: ">", value: 15000 }
```

**Process:**
```javascript
function buildJSONLogicCondition(condition) {
  const { field, operator, value } = condition;
  
  return {
    [operator]: [
      { "var": field },
      value
    ]
  };
}
```

**Output:**
```javascript
{ ">": [{ "var": "order.order_value" }, 15000] }
```

**More Examples:**

**Input:** `{ field: "distance_km", operator: "<", value: 10 }`  
**Output:** `{ "<": [{ "var": "distance_km" }, 10] }`

**Input:** `{ field: "order.dropoff_address.administrative_area", operator: "==", value: "CA" }`  
**Output:** `{ "==": [{ "var": "order.dropoff_address.administrative_area" }, "CA"] }`

**Called by:**
- `convertStructuredRuleToJSONLogic()` - function #19

---

# SERVICE RULE ENGINE
## Location: `src/services/ruleEngine.ts`

---

## 2️⃣1️⃣ `validateRule(condition, ruleType)`

**What it does:** Checks if a rule uses only allowed fields

**Input:**
```javascript
condition = {
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}
ruleType = "provider_selection"
```

**Process:**
```javascript
function validateRule(condition, ruleType) {
  // 1. Get allowed paths for this rule type
  const allowedPaths = getPathsForRuleType(ruleType);
  
  // 2. Collect all paths used in the rule
  const usedPaths = new Set();
  collectVarPaths(condition, usedPaths);
  
  // 3. Check each used path
  const errors = [];
  for (const path of usedPaths) {
    const normalized = normalizeArrayIndices(path);
    if (!allowedPaths.has(normalized)) {
      errors.push(
        `Field '${path}' is not available for ${ruleType}`
      );
    }
  }
  
  return {
    valid: errors.length === 0,
    errors
  };
}
```

**Output (valid):**
```javascript
{
  valid: true,
  errors: []
}
```

**Output (invalid):**
```javascript
{
  valid: false,
  errors: [
    "Field 'order.customer_age' is not available for provider_selection"
  ]
}
```

**Example with invalid field:**

**Input:**
```javascript
condition = {
  "if": [
    { ">": [{ "var": "order.customer_age" }, 18] },
    "doordash",
    "uber"
  ]
}
ruleType = "provider_selection"
```

**Output:**
```javascript
{
  valid: false,
  errors: ["Field 'order.customer_age' is not available for provider_selection"]
}
```

**Calls internally:**
- `getPathsForRuleType()` - function #3
- `collectVarPaths()` - function #24
- `normalizeArrayIndices()` - function #25

**Called by:**
- POST `/api/conversations/:id/generate-rule`
- POST `/api/rules/preview`
- POST `/api/rules`

---

## 2️⃣2️⃣ `explainRule(condition)`

**What it does:** Converts JSONLogic to readable English

**Input:**
```javascript
{
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}
```

**Process:**
```javascript
function explainRule(condition) {
  return explainNode(condition);
}

function explainNode(node) {
  if (!node || typeof node !== "object") {
    return String(node);
  }
  
  // Handle "if" statements
  if (node.if) {
    const [test, thenValue, elseValue] = node.if;
    const testExplained = explainNode(test);
    const thenExplained = explainNode(thenValue);
    const elseExplained = explainNode(elseValue);
    
    return `If ${testExplained} → ${thenExplained}, otherwise → ${elseExplained}`;
  }
  
  // Handle operators
  if (node[">"])  return `${explainNode(node[">"][0])} > ${explainNode(node[">"][1])}`;
  if (node["<"])  return `${explainNode(node["<"][0])} < ${explainNode(node["<"][1])}`;
  if (node["=="]) return `${explainNode(node["=="][0])} == ${explainNode(node["=="][1])}`;
  
  // Handle var
  if (node.var) return node.var;
  
  return JSON.stringify(node);
}
```

**Output:**
```javascript
"If order.order_value > 15000 → doordash, otherwise → uber"
```

**More Examples:**

**Input:**
```javascript
{ "==": [{ "var": "order.dropoff_address.administrative_area" }, "CA"] }
```
**Output:**
```javascript
"order.dropoff_address.administrative_area == CA"
```

**Input:**
```javascript
{
  "if": [
    { "<": [{ "var": "distance_km" }, 10] },
    "uber",
    "doordash"
  ]
}
```
**Output:**
```javascript
"If distance_km < 10 → uber, otherwise → doordash"
```

**Called by:**
- POST `/api/conversations/:id/generate-rule`
- POST `/api/rules/preview`
- POST `/api/rules/evaluate`

---

## 2️⃣3️⃣ `testRuleOnSampleData(condition, ruleType)`

**What it does:** Runs the rule on 3 fake orders to show examples

**Input:**
```javascript
condition = {
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}
ruleType = "provider_selection"
```

**Process:**
```javascript
function testRuleOnSampleData(condition, ruleType) {
  // 1. Build sample contexts
  const contexts = buildSampleContexts(ruleType);
  
  // 2. Test rule on each context
  const results = [];
  for (const context of contexts) {
    const result = jsonLogic.apply(condition, context);
    const orderValue = context.order?.order_value || 0;
    const formatted = `Order with value $${(orderValue / 100).toFixed(2)} → ${result}`;
    results.push(formatted);
  }
  
  return results;
}
```

**Output:**
```javascript
[
  "Order with value $200.00 → doordash",
  "Order with value $50.00 → uber",
  "Order with value $150.00 → uber"
]
```

**Calls internally:**
- `buildSampleContexts()` - function #26

**Called by:**
- POST `/api/conversations/:id/generate-rule`
- POST `/api/rules/preview`

---

## 2️⃣4️⃣ `evaluateRule(condition, context)`

**What it does:** Runs a rule on REAL order data

**Input:**
```javascript
condition = {
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}

context = {
  "order": {
    "task_id": "task_12345",
    "order_value": 25000,
    "tip": 500
  },
  "distance_km": 8.5
}
```

**Process:**
```javascript
function evaluateRule(condition, context) {
  return jsonLogic.apply(condition, context);
}
```

**How JSONLogic works:**
```
1. See "if" statement
2. Evaluate test: { ">": [{ "var": "order.order_value" }, 15000] }
3. Get "order.order_value" from context → 25000
4. Check: 25000 > 15000? → TRUE
5. Return first branch: "doordash"
```

**Output:**
```javascript
"doordash"
```

**Example with FALSE condition:**

**Input:**
```javascript
context = {
  "order": { "order_value": 8000 },
  "distance_km": 3.0
}
```

**Process:**
```
1. Get order.order_value → 8000
2. Check: 8000 > 15000? → FALSE
3. Return else branch: "uber"
```

**Output:**
```javascript
"uber"
```

**Called by:**
- POST `/api/rules/evaluate`

---

## 2️⃣5️⃣ `collectVarPaths(node, out)`

**What it does:** Walks through JSONLogic and finds all `{ "var": "..." }` references

**Input:**
```javascript
node = {
  "if": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    "doordash",
    "uber"
  ]
}
out = new Set()
```

**Process (recursively):**
```javascript
function collectVarPaths(node, out) {
  if (!node || typeof node !== "object") return;
  
  // If this is a "var" node, add it
  if (node.var) {
    out.add(node.var);
    return;
  }
  
  // Otherwise, recurse into all values
  if (Array.isArray(node)) {
    for (const item of node) {
      collectVarPaths(item, out);
    }
  } else {
    for (const value of Object.values(node)) {
      collectVarPaths(value, out);
    }
  }
}
```

**After execution, `out` contains:**
```javascript
Set { "order.order_value" }
```

**More complex example:**

**Input:**
```javascript
node = {
  "and": [
    { ">": [{ "var": "order.order_value" }, 15000] },
    { "==": [{ "var": "order.dropoff_address.administrative_area" }, "CA"] }
  ]
}
```

**Output (Set contents):**
```javascript
Set {
  "order.order_value",
  "order.dropoff_address.administrative_area"
}
```

**Called by:**
- `validateRule()` - function #21

---

## 2️⃣6️⃣ `normalizeArrayIndices(path)`

**What it does:** Converts `provider_quotes[0].fee` to `provider_quotes[].fee`

**Why:** The allowed list uses `[]` for arrays, but rules might use `[0]` or `[1]`

**Input Examples:**

| Input | Output |
|-------|--------|
| `"provider_quotes[0].fee"` | `"provider_quotes[].fee"` |
| `"provider_quotes[].provider"` | `"provider_quotes[].provider"` |
| `"order.order_value"` | `"order.order_value"` |

**Process:**
```javascript
function normalizeArrayIndices(path) {
  return path.replace(/\[\d+\]/g, '[]');
}
```

**Regular expression explanation:**
- `\[` - literal `[`
- `\d+` - one or more digits
- `\]` - literal `]`
- `g` - global (replace all occurrences)

**Called by:**
- `validateRule()` - function #21

---

## 2️⃣7️⃣ `buildSampleContexts(ruleType)`

**What it does:** Creates 3 fake orders for testing

**Input:** `"provider_selection"` or `"quote_evaluation"`

**Process:**
```javascript
function buildSampleContexts(ruleType) {
  const baseContexts = [
    // High value order
    {
      order: {
        task_id: "sample_1",
        order_value: 20000,  // $200
        tip: 1000,
        dropoff_address: {
          administrative_area: "CA",
          locality: "San Francisco"
        }
      },
      distance_km: 15.5
    },
    // Low value order
    {
      order: {
        task_id: "sample_2",
        order_value: 5000,  // $50
        tip: 500,
        dropoff_address: {
          administrative_area: "NY",
          locality: "New York"
        }
      },
      distance_km: 3.2
    },
    // Medium value order
    {
      order: {
        task_id: "sample_3",
        order_value: 15000,  // $150 exactly
        tip: 750,
        dropoff_address: {
          administrative_area: "TX",
          locality: "Austin"
        }
      },
      distance_km: 8.0
    }
  ];
  
  // If quote_evaluation, add quotes to each
  if (ruleType === "quote_evaluation") {
    return baseContexts.map(ctx => ({
      ...ctx,
      provider_quotes: [
        {
          provider: "uber",
          quote_id: "quote_uber_1",
          fee: 850,
          currency: "USD"
        },
        {
          provider: "doordash",
          quote_id: "quote_dd_1",
          fee: 975,
          currency: "USD"
        }
      ]
    }));
  }
  
  return baseContexts;
}
```

**Output (provider_selection):**
```javascript
[
  {
    order: { order_value: 20000, ... },
    distance_km: 15.5
  },
  {
    order: { order_value: 5000, ... },
    distance_km: 3.2
  },
  {
    order: { order_value: 15000, ... },
    distance_km: 8.0
  }
]
```

**Output (quote_evaluation):**
```javascript
[
  {
    order: { order_value: 20000, ... },
    distance_km: 15.5,
    provider_quotes: [
      { provider: "uber", fee: 850 },
      { provider: "doordash", fee: 975 }
    ]
  },
  // ... 2 more with quotes
]
```

**Called by:**
- `testRuleOnSampleData()` - function #23

---

# MODEL METHODS
## Location: `src/models/`

---

## 2️⃣8️⃣ `Conversation.create(data)`

**What it does:** Creates a new conversation document in MongoDB

**Input:**
```javascript
{
  orgId: new mongoose.Types.ObjectId("507f1f77bcf86cd799439011"),
  messages: [],
  status: "drafting"
}
```

**Output:**
```javascript
{
  _id: new ObjectId("65b2c3d4e5f6g7h8i9j0k1l2"),
  orgId: ObjectId("507f1f77bcf86cd799439011"),
  messages: [],
  status: "drafting",
  createdAt: new Date("2026-02-04T10:00:00.000Z"),
  updatedAt: new Date("2026-02-04T10:00:00.000Z")
}
```

**Called by:**
- POST `/api/conversations` route handler

---

## 2️⃣9️⃣ `Conversation.findById(id)`

**What it does:** Finds a conversation by ID

**Input:**
```javascript
"65b2c3d4e5f6g7h8i9j0k1l2"
```

**Output (found):**
```javascript
{
  _id: ObjectId("65b2c3d4e5f6g7h8i9j0k1l2"),
  orgId: ObjectId("507f1f77bcf86cd799439011"),
  messages: [
    { role: "user", content: "..." },
    { role: "assistant", content: "..." }
  ],
  status: "drafting",
  createdAt: Date,
  updatedAt: Date
}
```

**Output (not found):**
```javascript
null
```

**Called by:**
- POST `/api/conversations/:id/messages`
- GET `/api/conversations/:id`
- POST `/api/conversations/:id/generate-rule`

---

## 3️⃣0️⃣ `conversation.save()`

**What it does:** Saves changes to an existing conversation

**Input:** Modified conversation document

**Example:**
```javascript
const conversation = await Conversation.findById(id);
conversation.messages.push({ role: "user", content: "Hello" });
await conversation.save();
```

**Output:** Updated document with new `updatedAt` timestamp

**Called by:**
- POST `/api/conversations/:id/messages`

---

## 3️⃣1️⃣ `Rule.create(data)`

**What it does:** Creates a new rule in MongoDB

**Input:**
```javascript
{
  name: "Expensive orders to DoorDash",
  description: "Routes orders over $150 to DoorDash",
  ruleType: "provider_selection",
  condition: { "if": [...] },
  action: {},
  enabled: true,
  orgId: new ObjectId("507f1f77bcf86cd799439011")
}
```

**Output:**
```javascript
{
  _id: new ObjectId("65c3d4e5f6g7h8i9j0k1l2m3"),
  name: "Expensive orders to DoorDash",
  ruleType: "provider_selection",
  condition: { "if": [...] },
  enabled: true,
  orgId: ObjectId("507f1f77bcf86cd799439011"),
  createdAt: Date,
  updatedAt: Date
}
```

**Called by:**
- POST `/api/rules` route handler

---

## 3️⃣2️⃣ `Rule.find(filter)`

**What it does:** Finds all rules matching a filter

**Input:**
```javascript
{
  orgId: new ObjectId("507f1f77bcf86cd799439011"),
  enabled: true,
  ruleType: "provider_selection"
}
```

**Output:**
```javascript
[
  {
    _id: ObjectId("65c3d4e5f6g7h8i9j0k1l2m3"),
    name: "Expensive orders to DoorDash",
    enabled: true,
    ruleType: "provider_selection",
    condition: { ... }
  },
  {
    _id: ObjectId("65d4e5f6g7h8i9j0k1l2m3n4"),
    name: "California orders",
    enabled: true,
    ruleType: "provider_selection",
    condition: { ... }
  }
]
```

**Called by:**
- GET `/api/rules` route handler
- POST `/api/rules/evaluate` route handler

---

## 3️⃣3️⃣ `Rule.findById(id)`

**What it does:** Finds a single rule by ID

**Input:**
```javascript
"65c3d4e5f6g7h8i9j0k1l2m3"
```

**Output (found):**
```javascript
{
  _id: ObjectId("65c3d4e5f6g7h8i9j0k1l2m3"),
  name: "Expensive orders to DoorDash",
  ruleType: "provider_selection",
  condition: { "if": [...] },
  enabled: true,
  orgId: ObjectId("507f1f77bcf86cd799439011")
}
```

**Output (not found):**
```javascript
null
```

**Called by:**
- GET `/api/rules/:id`
- PATCH `/api/rules/:id`
- DELETE `/api/rules/:id`

---

# COMPLETE TESTING GUIDE

## 🎯 Test Scenario 1: Complete Flow from Chat to Working Rule

### Step 1: Create Conversation
```
POST http://localhost:3001/api/conversations
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Expected:** Status 201  
**Save:** `response._id` as `{{conversationId}}`

---

### Step 2: First Message
```
POST http://localhost:3001/api/conversations/{{conversationId}}/messages
Content-Type: application/json

{
  "content": "I want expensive orders to go to DoorDash"
}
```

**Expected:** 
- Status 200
- AI asks about threshold (e.g., "What dollar amount is expensive?")

---

### Step 3: Clarify Threshold
```
POST http://localhost:3001/api/conversations/{{conversationId}}/messages
Content-Type: application/json

{
  "content": "Over $150"
}
```

**Expected:**
- Status 200
- AI asks about other orders (e.g., "What about orders under $150?")

---

### Step 4: Complete the Rule
```
POST http://localhost:3001/api/conversations/{{conversationId}}/messages
Content-Type: application/json

{
  "content": "Orders under $150 should go to Uber"
}
```

**Expected:**
- Status 200
- AI confirms understanding

---

### Step 5: Generate Rule
```
POST http://localhost:3001/api/conversations/{{conversationId}}/generate-rule
```

**Expected:**
```json
{
  "valid": true,
  "errors": [],
  "explanation": "If order.order_value > 15000 → doordash, otherwise → uber",
  "sampleResults": [
    "Order with value $200.00 → doordash",
    "Order with value $50.00 → uber",
    "Order with value $150.00 → uber"
  ],
  "rule": {
    "ruleType": "provider_selection",
    "condition": {
      "if": [
        { ">": [{ "var": "order.order_value" }, 15000] },
        "doordash",
        "uber"
      ]
    }
  }
}
```

**Tests to verify:**
- ✅ `valid === true`
- ✅ `errors` is empty array
- ✅ `explanation` contains the logic
- ✅ `sampleResults` shows 3 examples
- ✅ `rule.condition` uses `order.order_value` (which is allowed)
- ✅ Value is `15000` (not `150` - remember cents!)

---

### Step 6: Save Rule
```
POST http://localhost:3001/api/rules
Content-Type: application/json

{
  "name": "Expensive orders to DoorDash",
  "description": "Routes orders over $150 to DoorDash, otherwise Uber",
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.order_value" }, 15000] },
      "doordash",
      "uber"
    ]
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

**Expected:** Status 201, rule saved  
**Save:** `response.id` as `{{ruleId}}`

---

### Step 7: Test - High Value Order
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "task_id": "task_001",
      "order_value": 25000,
      "tip": 2000
    },
    "distance_km": 10.5
  }
}
```

**Expected:**
```json
{
  "result": "doordash",
  "matchedRule": {
    "id": "{{ruleId}}",
    "name": "Expensive orders to DoorDash"
  },
  "reason": "If order.order_value > 15000 → doordash, otherwise → uber",
  "selectedProvider": "doordash"
}
```

**Why:** 25000 > 15000, so rule returns "doordash"

---

### Step 8: Test - Low Value Order
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "task_id": "task_002",
      "order_value": 8000,
      "tip": 500
    },
    "distance_km": 3.2
  }
}
```

**Expected:**
```json
{
  "result": "uber",
  "selectedProvider": "uber"
}
```

**Why:** 8000 NOT > 15000, so rule returns "uber" (else branch)

---

### Step 9: Test - Exactly $150
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "task_id": "task_003",
      "order_value": 15000,
      "tip": 750
    },
    "distance_km": 5.0
  }
}
```

**Expected:**
```json
{
  "result": "uber",
  "selectedProvider": "uber"
}
```

**Why:** 15000 is NOT > 15000 (needs to be strictly greater), so returns "uber"

---

## 🎯 Test Scenario 2: Testing Invalid Fields

### Try using a field NOT on the allowed list

```
POST http://localhost:3001/api/rules/preview
Content-Type: application/json

{
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { ">": [{ "var": "order.customer_age" }, 18] },
      "doordash",
      "uber"
    ]
  }
}
```

**Expected:**
```json
{
  "valid": false,
  "errors": [
    "Field 'order.customer_age' is not available for provider_selection"
  ],
  "explanation": "If order.customer_age > 18 → doordash, otherwise → uber",
  "sampleResults": []
}
```

**Why:** `customer_age` is NOT in `rules_data_elements.json`, so validation fails

---

## 🎯 Test Scenario 3: Geography-Based Rule

### Create California → DoorDash rule

```
POST http://localhost:3001/api/rules
Content-Type: application/json

{
  "name": "California to DoorDash",
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { "==": [{ "var": "order.dropoff_address.administrative_area" }, "CA"] },
      "doordash",
      "uber"
    ]
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

### Test - California Order
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "order_value": 10000,
      "dropoff_address": {
        "administrative_area": "CA",
        "locality": "Los Angeles"
      }
    },
    "distance_km": 12.0
  }
}
```

**Expected:** `result: "doordash"`

### Test - New York Order
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": {
      "order_value": 10000,
      "dropoff_address": {
        "administrative_area": "NY",
        "locality": "New York"
      }
    },
    "distance_km": 8.0
  }
}
```

**Expected:** `result: "uber"`

---

## 🎯 Test Scenario 4: Distance-Based Rule

### Create rule: Short distance → Uber, Long distance → DoorDash

```
POST http://localhost:3001/api/rules
Content-Type: application/json

{
  "name": "Distance based routing",
  "ruleType": "provider_selection",
  "condition": {
    "if": [
      { "<": [{ "var": "distance_km" }, 5] },
      "uber",
      "doordash"
    ]
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

### Test - Short Distance (3km)
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": { "order_value": 10000 },
    "distance_km": 3.0
  }
}
```

**Expected:** `result: "uber"` (3 < 5)

### Test - Long Distance (12km)
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "provider_selection",
  "context": {
    "order": { "order_value": 10000 },
    "distance_km": 12.0
  }
}
```

**Expected:** `result: "doordash"` (12 NOT < 5)

---

## 🎯 Test Scenario 5: Quote Evaluation (Cheapest Quote)

### Create cheapest quote rule

```
POST http://localhost:3001/api/rules
Content-Type: application/json

{
  "name": "Pick cheapest quote",
  "ruleType": "quote_evaluation",
  "condition": {
    "var": "provider_quotes[0].provider"
  },
  "enabled": true,
  "orgId": "507f1f77bcf86cd799439011"
}
```

### Test - Uber is cheaper
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "quote_evaluation",
  "context": {
    "order": { "order_value": 10000 },
    "distance_km": 8.0,
    "provider_quotes": [
      {
        "provider": "uber",
        "quote_id": "quote_uber_123",
        "fee": 850,
        "currency": "USD"
      },
      {
        "provider": "doordash",
        "quote_id": "quote_dd_456",
        "fee": 975,
        "currency": "USD"
      }
    ]
  }
}
```

**Expected:** `result: "uber"`  
**Why:** `provider_quotes[0]` is Uber (cheapest at 850 cents = $8.50)

### Test - DoorDash is cheaper
```
POST http://localhost:3001/api/rules/evaluate
Content-Type: application/json

{
  "orgId": "507f1f77bcf86cd799439011",
  "ruleType": "quote_evaluation",
  "context": {
    "order": { "order_value": 10000 },
    "distance_km": 8.0,
    "provider_quotes": [
      {
        "provider": "doordash",
        "quote_id": "quote_dd_789",
        "fee": 650,
        "currency": "USD"
      },
      {
        "provider": "uber",
        "quote_id": "quote_uber_012",
        "fee": 875,
        "currency": "USD"
      }
    ]
  }
}
```

**Expected:** `result: "doordash"`  
**Why:** `provider_quotes[0]` is DoorDash (cheapest at 650 cents = $6.50)

---

## 🧪 Complete Postman Test Collection

```javascript
// Pre-request Script (Collection level)
pm.environment.set("baseUrl", "http://localhost:3001");
pm.environment.set("orgId", "507f1f77bcf86cd799439011");
```

### Test 1: Create Conversation
```javascript
// Tests tab
pm.test("Status is 201", () => pm.response.to.have.status(201));
pm.test("Has conversation ID", () => {
  const json = pm.response.json();
  pm.expect(json).to.have.property("_id");
  pm.environment.set("conversationId", json._id);
});
```

### Test 2: Send Message
```javascript
// Tests tab
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Has assistant message", () => {
  const json = pm.response.json();
  pm.expect(json.message.role).to.equal("assistant");
});
```

### Test 3: Generate Rule
```javascript
// Tests tab
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Rule is valid", () => {
  const json = pm.response.json();
  pm.expect(json.valid).to.equal(true);
});
pm.test("Has sample results", () => {
  const json = pm.response.json();
  pm.expect(json.sampleResults).to.be.an("array").with.lengthOf(3);
});
```

### Test 4: Save Rule
```javascript
// Tests tab
pm.test("Status is 201", () => pm.response.to.have.status(201));
pm.test("Rule saved with ID", () => {
  const json = pm.response.json();
  pm.expect(json).to.have.property("id");
  pm.environment.set("ruleId", json.id);
});
```

### Test 5: Evaluate High Value
```javascript
// Tests tab
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Selected DoorDash", () => {
  const json = pm.response.json();
  pm.expect(json.result).to.equal("doordash");
});
```

### Test 6: Evaluate Low Value
```javascript
// Tests tab
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Selected Uber", () => {
  const json = pm.response.json();
  pm.expect(json.result).to.equal("uber");
});
```

---

## 📊 Function Call Chain Summary

Here's how functions connect in a real flow:

```
USER SENDS MESSAGE
↓
POST /api/conversations/:id/messages
↓
├─ sendMessageSchema.safeParse()
├─ Conversation.findById()
├─ getChatResponse()
│  ├─ buildSystemPrompt()
│  │  └─ getSchemaForSystemPrompt()
│  │     └─ loadRuleDataElements()
│  └─ OpenAI API call
└─ conversation.save()

USER GENERATES RULE
↓
POST /api/conversations/:id/generate-rule
↓
├─ Conversation.findById()
├─ extractRuleFromMessages()
│  ├─ buildExtractPrompt()
│  │  └─ getSchemaForSystemPrompt()
│  ├─ OpenAI API call
│  └─ processExtractedContent()
│     ├─ parseJsonFromResponse()
│     ├─ extractedRuleSchema.safeParse()
│     └─ convertStructuredRuleToJSONLogic()
│        └─ buildJSONLogicCondition()
├─ validateRule()
│  ├─ getPathsForRuleType()
│  │  └─ loadRuleDataElements()
│  ├─ collectVarPaths()
│  └─ normalizeArrayIndices()
├─ explainRule()
│  └─ explainNode()
└─ testRuleOnSampleData()
   ├─ buildSampleContexts()
   └─ jsonLogic.apply() [3 times]

USER SAVES RULE
↓
POST /api/rules
↓
├─ createRuleSchema.safeParse()
├─ validateRule() [same as above]
└─ Rule.create()

SYSTEM EVALUATES RULE
↓
POST /api/rules/evaluate
↓
├─ evaluateRequestSchema.safeParse()
├─ Rule.find()
├─ Loop: evaluateRule() for each rule
│  └─ jsonLogic.apply()
└─ explainRule() [for winner]
```

---

This guide covers every function with examples and shows you exactly how to test each part! 🚀
