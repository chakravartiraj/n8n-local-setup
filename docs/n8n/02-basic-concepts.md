# N8N Basic Concepts

Now that you've created your first workflow, let's dive deeper into the fundamental concepts that make N8N powerful and flexible.

## 📋 Table of Contents
- [Core Architecture](#core-architecture)
- [Nodes Deep Dive](#nodes-deep-dive)
- [Data Flow & Structure](#data-flow--structure)
- [Triggers & Execution](#triggers--execution)
- [Expressions & Variables](#expressions--variables)
- [Credentials & Security](#credentials--security)
- [Best Practices](#best-practices)

## 🏗 Core Architecture

### The N8N Mental Model
Think of N8N as a **digital assembly line** where:
- **Nodes** are workstations
- **Data** is the product moving through
- **Connections** are conveyor belts
- **Expressions** are quality control instructions

```
Input → Process → Transform → Output → Action
  ↓        ↓         ↓        ↓       ↓
Trigger   Filter    Set     Format  Execute
```

### Execution Philosophy
N8N follows these principles:
1. **Sequential Execution**: Nodes run in order
2. **Data Preservation**: Original data persists unless modified
3. **Error Propagation**: Failures stop execution by default
4. **Stateless Operations**: Each execution is independent

## 🧩 Nodes Deep Dive

### Node Categories

#### 1. **Trigger Nodes** 🎯
Start your workflows automatically
```
Webhook ────────→ (starts when HTTP request received)
Schedule ───────→ (starts at specific times)
Email Trigger ──→ (starts when email received)
File Trigger ───→ (starts when file changes)
```

#### 2. **Action Nodes** ⚡
Perform operations and integrations
```
HTTP Request ───→ (call APIs)
Email ──────────→ (send emails)
Database ───────→ (query/insert data)
File System ────→ (read/write files)
```

#### 3. **Logic Nodes** 🧠
Control workflow behavior
```
IF ─────────────→ (conditional branching)
Switch ─────────→ (multiple conditions)
Merge ──────────→ (combine data streams)
Wait ───────────→ (pause execution)
```

#### 4. **Data Nodes** 📊
Transform and manipulate data
```
Set ────────────→ (modify/add fields)
Function ───────→ (custom JavaScript)
JSON ───────────→ (parse/stringify JSON)
XML ────────────→ (work with XML data)
```

### Node Anatomy
Every node has these components:

```
┌─────────────────────────────────────┐
│  📱 Node Icon & Name                │
├─────────────────────────────────────┤
│  ⚙️  Configuration Panel            │
│     • Required Parameters           │
│     • Optional Settings             │
│     • Advanced Options              │
├─────────────────────────────────────┤
│  🔌 Input Connection Points         │
│  🔌 Output Connection Points        │
├─────────────────────────────────────┤
│  📊 Execution Status                │
│     • Success ✅                    │
│     • Error ❌                      │
│     • Running ⏳                    │
└─────────────────────────────────────┘
```

## 📊 Data Flow & Structure

### Understanding JSON Data
N8N primarily works with JSON (JavaScript Object Notation):

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30,
  "interests": ["automation", "productivity"],
  "address": {
    "street": "123 Main St",
    "city": "Automation City"
  }
}
```

### Data Types in N8N

#### 1. **String** 📝
```json
"Hello World"
"user@example.com"
"2024-01-15"
```

#### 2. **Number** 🔢
```json
42
3.14159
-100
```

#### 3. **Boolean** ✅❌
```json
true
false
```

#### 4. **Array** 📋
```json
["apple", "banana", "cherry"]
[1, 2, 3, 4, 5]
[{"name": "John"}, {"name": "Jane"}]
```

#### 5. **Object** 📦
```json
{
  "user": {
    "id": 123,
    "profile": {
      "name": "John",
      "verified": true
    }
  }
}
```

### Data Flow Patterns

#### 1. **Linear Flow** (Most Common)
```
A → B → C → D
```
Each node processes data sequentially.

#### 2. **Branching Flow**
```
A → B → C
    ↓
    D → E
```
Data splits into multiple paths.

#### 3. **Merging Flow**
```
A → B ↘
      → D → E
C ────↗
```
Multiple data streams combine.

#### 4. **Parallel Processing**
```
    ┌→ B ┐
A ──┼→ C ┼→ F
    └→ D ┘
```
Multiple operations on same data.

## ⚡ Triggers & Execution

### Trigger Types

#### 1. **Manual Triggers**
- **When**: You click "Execute Workflow"
- **Use Case**: Testing and one-time operations
- **Example**: Data migration scripts

#### 2. **Webhook Triggers**
- **When**: HTTP request received
- **Use Case**: Real-time integrations
- **Example**: Form submissions, API callbacks

```http
POST http://localhost:5678/webhook/my-webhook
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

#### 3. **Schedule Triggers** (Cron)
- **When**: Specific times/intervals
- **Use Case**: Regular maintenance tasks
- **Examples**:
  ```
  0 9 * * *        (Daily at 9 AM)
  0 0 * * 1        (Every Monday)
  */15 * * * *     (Every 15 minutes)
  0 0 1 * *        (First day of month)
  ```

#### 4. **Polling Triggers**
- **When**: Checking for changes regularly
- **Use Case**: Monitor external systems
- **Example**: New emails, file changes

### Execution Modes

#### 1. **Manual Execution**
- Triggered by user action
- Useful for testing
- Shows detailed execution log

#### 2. **Automatic Execution**
- Triggered by events/schedule
- Runs in background
- Logged for monitoring

#### 3. **Error Execution**
- Automatically retries failed workflows
- Configurable retry logic
- Sends error notifications

## 🔧 Expressions & Variables

### Expression Basics
Expressions allow dynamic data manipulation using `{{ }}` syntax:

```javascript
{{ $json.fieldName }}          // Access field from current node
{{ $node["Node Name"].json }}  // Access data from specific node
{{ $now }}                     // Current timestamp
{{ $workflow.name }}           // Workflow name
```

### Common Expression Patterns

#### 1. **Accessing Data**
```javascript
// Current node data
{{ $json.email }}
{{ $json.user.name }}
{{ $json.items[0].price }}

// Previous node data
{{ $node["HTTP Request"].json.result }}

// All items from previous node
{{ $items }}
```

#### 2. **String Operations**
```javascript
// Concatenation
{{ $json.firstName + " " + $json.lastName }}

// Template literals
{{ `Hello ${$json.name}!` }}

// String methods
{{ $json.email.toLowerCase() }}
{{ $json.name.trim() }}
{{ $json.text.slice(0, 100) }}
```

#### 3. **Number Operations**
```javascript
// Basic math
{{ $json.price * 1.1 }}        // Add 10%
{{ $json.quantity + 5 }}       // Add 5
{{ Math.round($json.value) }}   // Round number

// Formatting
{{ $json.price.toFixed(2) }}    // 2 decimal places
```

#### 4. **Date Operations**
```javascript
// Current date/time
{{ $now }}
{{ new Date().toISOString() }}

// Date formatting
{{ DateTime.now().toFormat('yyyy-MM-dd') }}
{{ DateTime.now().plus({days: 7}).toISO() }}

// Parse dates
{{ DateTime.fromISO($json.dateString) }}
```

#### 5. **Conditional Logic**
```javascript
// Ternary operator
{{ $json.status === 'active' ? 'Enabled' : 'Disabled' }}

// Null coalescing
{{ $json.name || 'Unknown' }}

// Complex conditions
{{ $json.score > 80 ? 'Pass' : $json.score > 60 ? 'Review' : 'Fail' }}
```

### Variable Types

#### 1. **$json** - Current Item Data
```javascript
{{ $json.fieldName }}
{{ $json.nested.property }}
```

#### 2. **$items** - All Items Array
```javascript
{{ $items.length }}                    // Count of items
{{ $items[0].json.fieldName }}        // First item
{{ $items.map(item => item.json.name) }} // Extract field from all
```

#### 3. **$node** - Specific Node Data
```javascript
{{ $node["Node Name"].json }}
{{ $node["HTTP Request"].json.result }}
```

#### 4. **$workflow** - Workflow Context
```javascript
{{ $workflow.id }}
{{ $workflow.name }}
{{ $workflow.active }}
```

#### 5. **$execution** - Execution Context
```javascript
{{ $execution.id }}
{{ $execution.mode }}
{{ $execution.resumeUrl }}
```

## 🔐 Credentials & Security

### Credential Types

#### 1. **API Keys**
```
Service: OpenAI
Type: API Key
Key: sk-xxxxxxxxxxxxxxxxxx
```

#### 2. **OAuth2**
```
Service: Google
Type: OAuth2
Scopes: https://www.googleapis.com/auth/drive
```

#### 3. **Basic Auth**
```
Username: admin
Password: secretpassword
```

#### 4. **Custom Headers**
```
Header: Authorization
Value: Bearer your-token-here
```

### Security Best Practices

#### 1. **Credential Management**
- ✅ Use credential system, never hardcode secrets
- ✅ Regularly rotate API keys
- ✅ Use minimal required permissions
- ❌ Never share credentials in screenshots

#### 2. **Data Protection**
- ✅ Enable HTTPS for webhooks
- ✅ Validate input data
- ✅ Sanitize sensitive data in logs
- ❌ Log passwords or API keys

#### 3. **Access Control**
- ✅ Use environment-specific credentials
- ✅ Implement proper user permissions
- ✅ Monitor credential usage
- ❌ Share production credentials

## 🎯 Best Practices

### Workflow Design

#### 1. **Start Simple, Iterate**
```
Phase 1: A → B → C (basic functionality)
Phase 2: A → B → C → D (add features)
Phase 3: A → B ⇉ C → D (add error handling)
              ↓
              E (error recovery)
```

#### 2. **Use Descriptive Names**
```
❌ Bad:  "HTTP Request", "Set", "IF"
✅ Good: "Fetch User Data", "Format Response", "Check Status"
```

#### 3. **Add Documentation**
- Use node notes for complex logic
- Document workflow purpose
- Explain unusual expressions
- Note external dependencies

#### 4. **Handle Errors Gracefully**
```
Main Flow: A → B → C → D
           ↓   ↓   ↓   ↓
Error:     E → F → G → H (error handling)
```

### Performance Optimization

#### 1. **Efficient Data Processing**
```javascript
// ✅ Good: Process in batches
{{ $items.slice(0, 100) }}

// ❌ Bad: Process one by one in loop
```

#### 2. **Minimize API Calls**
```
❌ Bad:  Call API for each item individually
✅ Good: Batch API calls when possible
```

#### 3. **Use Appropriate Triggers**
```
❌ Bad:  Poll every minute for rare events
✅ Good: Use webhooks for real-time events
```

### Debugging Strategies

#### 1. **Use Manual Execution**
- Test with real data
- Check each node output
- Verify expressions work correctly

#### 2. **Add Debug Nodes**
```
A → Debug → B → Debug → C
```
Use "No Operation" nodes to inspect data flow.

#### 3. **Check Execution History**
- Review failed executions
- Analyze error messages
- Look for patterns in failures

## 📝 Practical Exercises

### Exercise 1: Data Transformation Master
**Goal**: Master data manipulation
**Scenario**: Transform user registration data
**Input**:
```json
{
  "firstName": "john",
  "lastName": "DOE",
  "email": "JOHN.DOE@EXAMPLE.COM",
  "birthDate": "1990-05-15"
}
```

**Tasks**:
1. Normalize names (proper case)
2. Lowercase email
3. Calculate age from birthDate
4. Create full name field
5. Generate username (first.last)

**Expected Output**:
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "birthDate": "1990-05-15",
  "age": 34,
  "fullName": "John Doe",
  "username": "john.doe"
}
```

### Exercise 2: Conditional Logic Builder
**Goal**: Master IF/Switch nodes
**Scenario**: Order processing system
**Tasks**:
1. Check order amount (>$100 = VIP, <$100 = Standard)
2. Check shipping country (US = domestic, other = international)
3. Calculate shipping cost based on conditions
4. Route to appropriate fulfillment system

### Exercise 3: API Integration Practice
**Goal**: Master HTTP Request node
**Tasks**:
1. Fetch weather data for multiple cities
2. Transform response data
3. Calculate average temperature
4. Send summary email

## ✅ Mastery Checklist

Before moving to advanced topics, ensure you can:

### Data Handling
- [ ] Access nested JSON properties
- [ ] Use expressions for string manipulation
- [ ] Work with arrays and objects
- [ ] Handle different data types
- [ ] Debug data flow issues

### Node Operations
- [ ] Configure all major node types
- [ ] Chain nodes effectively
- [ ] Use conditional logic (IF/Switch)
- [ ] Implement error handling
- [ ] Optimize node performance

### Expressions
- [ ] Write complex expressions
- [ ] Use built-in functions
- [ ] Access node data dynamically
- [ ] Handle date/time operations
- [ ] Create conditional expressions

### Security
- [ ] Set up credentials properly
- [ ] Follow security best practices
- [ ] Protect sensitive data
- [ ] Use appropriate authentication

**Score: 15/20 or higher?** You're ready for [First Workflow Tutorial](./03-first-workflow.md)!

## 🚀 Next Steps

You now understand the fundamental concepts of N8N! Here's what comes next:

1. **Practice**: Build workflows using these concepts
2. **Experiment**: Try different node combinations
3. **Learn**: Explore the [First Workflow Tutorial](./03-first-workflow.md)
4. **Challenge**: Attempt the practical exercises above

**Remember**: Understanding concepts is just the beginning. The real learning happens when you start building! 🛠️

---

**Ready to build?** Continue with [First Workflow Tutorial →](./03-first-workflow.md)
