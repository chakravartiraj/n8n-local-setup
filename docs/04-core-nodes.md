# Core Nodes Guide

Master the essential building blocks of N8N automation. This comprehensive guide covers the most important nodes you'll use in 90% of your workflows.

## 📋 Table of Contents
- [Trigger Nodes](#trigger-nodes)
- [Action Nodes](#action-nodes)
- [Logic & Control Nodes](#logic--control-nodes)
- [Data Transformation Nodes](#data-transformation-nodes)
- [Integration Nodes](#integration-nodes)
- [Utility Nodes](#utility-nodes)
- [Node Combinations](#node-combinations)

## 🎯 Trigger Nodes

Triggers start your workflows. Choose the right trigger for your automation needs.

### 1. Webhook Trigger 🌐

**Use Case**: Real-time integrations, API endpoints, form submissions
**Best For**: Instant response workflows

#### Configuration
```yaml
HTTP Method: POST/GET/PUT/DELETE
Path: your-webhook-path
Authentication: None/Basic/Header
Response Mode: Immediate/After Execution
Response Data: JSON/Text
```

#### Common Patterns
```javascript
// Form submission webhook
{
  "name": "{{ $json.name }}",
  "email": "{{ $json.email }}",
  "message": "{{ $json.message }}"
}

// API callback webhook
{
  "event": "{{ $json.event }}",
  "data": "{{ $json.data }}",
  "timestamp": "{{ $json.timestamp }}"
}
```

#### Pro Tips
- ✅ Use descriptive webhook paths: `/order-created` vs `/webhook`
- ✅ Validate incoming data immediately
- ✅ Return meaningful responses
- ⚠️ Always handle malformed requests

### 2. Schedule Trigger ⏰

**Use Case**: Regular maintenance, reports, monitoring
**Best For**: Time-based automation

#### Cron Expression Examples
```bash
# Every minute
* * * * *

# Every hour at 15 minutes past
15 * * * *

# Daily at 9:30 AM
30 9 * * *

# Every Monday at 8 AM
0 8 * * 1

# First day of every month at midnight
0 0 1 * *

# Every 15 minutes during business hours (9-17)
*/15 9-17 * * 1-5
```

#### Common Use Cases
- **Daily Reports**: `0 9 * * *` (9 AM daily)
- **Weekly Backups**: `0 2 * * 0` (2 AM Sundays)
- **Hourly Monitoring**: `0 * * * *` (Top of every hour)
- **End of Month**: `0 0 1 * *` (Midnight, 1st of month)

#### Pro Tips
- ✅ Use timezone-aware scheduling
- ✅ Add buffer time for long-running workflows
- ✅ Test with frequent schedules first (every minute)
- ⚠️ Consider server load during peak hours

### 3. Email Trigger 📧

**Use Case**: Email-based automation, support tickets
**Best For**: Email processing workflows

#### Configuration
```yaml
IMAP Server: imap.gmail.com
Port: 993
Security: SSL/TLS
Check Interval: 60 seconds
Email Format: Simple/Full
```

#### Email Processing Patterns
```javascript
// Extract attachments
{{ $json.attachments }}

// Parse subject line
{{ $json.subject.match(/TICKET-(\d+)/)[1] }}

// Get sender info
{{ $json.from.address }}
{{ $json.from.name }}

// Email body content
{{ $json.text }}
{{ $json.html }}
```

### 4. File Trigger 📁

**Use Case**: File processing, data imports
**Best For**: File-based automation

#### Watch Patterns
```javascript
// Watch for new files
Watch Event: add

// Watch for modifications
Watch Event: change

// Watch specific file types
Path: /uploads/*.csv
Path: /images/*.{jpg,png,gif}

// Recursive watching
Include Subdirectories: true
```

## ⚡ Action Nodes

Action nodes perform operations and interact with external services.

### 1. HTTP Request 🌐

**The Swiss Army knife of N8N** - connects to any API

#### Common Configurations

##### GET Request
```yaml
Method: GET
URL: https://api.example.com/users/{{ $json.userId }}
Headers:
  Authorization: Bearer {{ $credentials.token }}
  Content-Type: application/json
```

##### POST Request
```yaml
Method: POST
URL: https://api.example.com/users
Body Parameters:
  name: "{{ $json.name }}"
  email: "{{ $json.email }}"
  department: "{{ $json.department }}"
```

##### Authentication Examples
```yaml
# API Key in Header
Headers:
  X-API-Key: "{{ $credentials.apiKey }}"

# Bearer Token
Headers:
  Authorization: "Bearer {{ $credentials.token }}"

# Basic Auth
Authentication: Basic Auth
Username: "{{ $credentials.username }}"
Password: "{{ $credentials.password }}"
```

#### Response Handling
```javascript
// Check response status
{{ $response.statusCode === 200 }}

// Extract specific data
{{ $response.body.data.users }}

// Handle errors
{{ $response.statusCode >= 400 ? 'Error' : 'Success' }}
```

### 2. Email Node 📧

**Use Case**: Automated notifications, reports, alerts
**Best For**: Communication workflows

#### Email Templates
```html
<!-- Welcome Email -->
<html>
<body>
  <h2>Welcome {{ $json.name }}!</h2>
  <p>Thank you for joining {{ $json.company }}.</p>
  <ul>
    <li>Account ID: {{ $json.accountId }}</li>
    <li>Registration Date: {{ $json.date }}</li>
  </ul>
</body>
</html>
```

```html
<!-- Report Email -->
<html>
<body>
  <h2>Daily Report - {{ DateTime.now().toFormat('yyyy-MM-dd') }}</h2>
  <table border="1">
    <tr>
      <th>Metric</th>
      <th>Value</th>
    </tr>
    <tr>
      <td>Total Orders</td>
      <td>{{ $json.totalOrders }}</td>
    </tr>
    <tr>
      <td>Revenue</td>
      <td>${{ $json.revenue.toFixed(2) }}</td>
    </tr>
  </table>
</body>
</html>
```

#### Dynamic Recipients
```javascript
// Multiple recipients
{{ $json.managers.map(m => m.email).join(',') }}

// Conditional recipients
{{ $json.priority === 'high' ? 'manager@company.com' : 'team@company.com' }}

// Department-based routing
{{ $json.department === 'sales' ? 'sales@company.com' : 'support@company.com' }}
```

### 3. Database Nodes 🗄️

Connect to MySQL, PostgreSQL, MongoDB, and more.

#### MySQL/PostgreSQL Examples
```sql
-- Insert new record
INSERT INTO customers (name, email, company, created_at)
VALUES (
  '{{ $json.name }}',
  '{{ $json.email }}',
  '{{ $json.company }}',
  NOW()
);

-- Update existing record
UPDATE orders 
SET status = '{{ $json.status }}',
    updated_at = NOW()
WHERE order_id = {{ $json.orderId }};

-- Complex query with joins
SELECT 
  c.name,
  c.email,
  COUNT(o.id) as order_count,
  SUM(o.total) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at >= '{{ $json.startDate }}'
GROUP BY c.id
HAVING order_count > 0;
```

#### MongoDB Examples
```javascript
// Find documents
db.customers.find({
  "email": "{{ $json.email }}",
  "status": "active"
})

// Insert document
db.orders.insertOne({
  "customerId": "{{ $json.customerId }}",
  "items": {{ $json.items }},
  "total": {{ $json.total }},
  "createdAt": new Date()
})

// Update document
db.customers.updateOne(
  { "_id": ObjectId("{{ $json.customerId }}") },
  { 
    "$set": { 
      "lastLogin": new Date(),
      "status": "{{ $json.status }}"
    }
  }
)
```

## 🧠 Logic & Control Nodes

Control the flow and logic of your workflows.

### 1. IF Node 🔀

**Use Case**: Conditional branching, decision making
**Best For**: Simple true/false decisions

#### Common Conditions
```javascript
// Value comparisons
{{ $json.amount > 1000 }}
{{ $json.status === 'approved' }}
{{ $json.priority !== 'low' }}

// String operations
{{ $json.email.includes('@company.com') }}
{{ $json.name.length > 0 }}
{{ $json.description.toLowerCase().includes('urgent') }}

// Array operations
{{ $json.tags.includes('vip') }}
{{ $json.items.length > 5 }}

// Date comparisons
{{ DateTime.fromISO($json.created_at) > DateTime.now().minus({days: 7}) }}

// Complex conditions
{{ $json.amount > 1000 && $json.customer_type === 'enterprise' }}
{{ $json.region === 'US' || $json.region === 'CA' }}
```

### 2. Switch Node 🔄

**Use Case**: Multiple conditions, routing logic
**Best For**: Complex decision trees

#### Configuration Examples
```javascript
// Route by customer tier
Mode: Expression
Value: {{ $json.customer_tier }}
Routes:
  - bronze: bronze customers
  - silver: silver customers  
  - gold: gold customers
  - default: all others

// Route by amount ranges
Mode: Expression
Value: {{ $json.amount }}
Routes:
  - "< 100": small orders
  - "100 <= value < 1000": medium orders
  - ">= 1000": large orders

// Route by region
Mode: Expression
Value: {{ $json.country }}
Routes:
  - "US,CA,MX": north_america
  - "GB,DE,FR,IT": europe
  - "AU,NZ": oceania
  - default: rest_of_world
```

### 3. Merge Node 🔗

**Use Case**: Combining data streams, synchronization
**Best For**: Parallel processing convergence

#### Merge Modes
```yaml
# Wait for all inputs
Mode: Wait
Wait for All: true

# Merge as they arrive
Mode: Append
Wait for All: false

# Keep only specific inputs
Mode: Choose Branch
Output: First/Last/All
```

#### Common Patterns
```javascript
// Combine user data from multiple sources
// Input 1: Basic info from webhook
// Input 2: Enriched data from API
// Output: Complete user profile

// Parallel processing results
// Input 1: Email validation result
// Input 2: Phone validation result  
// Input 3: Address validation result
// Output: Complete validation summary
```

### 4. Wait Node ⏱️

**Use Case**: Rate limiting, delays, timing control
**Best For**: Workflow timing management

#### Wait Types
```yaml
# Fixed delay
Mode: Time
Amount: 30
Unit: seconds

# Wait until specific time
Mode: Until
Time: 09:00
Timezone: America/New_York

# Dynamic delay
Mode: Expression
Value: {{ $json.priority === 'low' ? 3600 : 60 }}
```

## 📊 Data Transformation Nodes

Transform and manipulate data flowing through your workflows.

### 1. Set Node ✏️

**The most versatile node** - modify, add, and transform data

#### Common Patterns
```javascript
// Add calculated fields
fullName: {{ $json.firstName + ' ' + $json.lastName }}
slug: {{ $json.title.toLowerCase().replace(/\s+/g, '-') }}
expiryDate: {{ DateTime.now().plus({days: 30}).toISO() }}

// Format data
formattedPrice: ${{ $json.price.toFixed(2) }}
displayDate: {{ DateTime.fromISO($json.date).toFormat('MMM dd, yyyy') }}
percentage: {{ ($json.value * 100).toFixed(1) }}%

// Clean and validate
email: {{ $json.email.toLowerCase().trim() }}
phone: {{ $json.phone.replace(/\D/g, '') }}
status: {{ $json.status || 'pending' }}

// Transform arrays
tagList: {{ $json.tags.join(', ') }}
firstItem: {{ $json.items[0] }}
itemCount: {{ $json.items.length }}

// Create objects
address: {{
  {
    street: $json.street,
    city: $json.city,
    state: $json.state,
    zip: $json.zip
  }
}}
```

### 2. Function Node 🔧

**Use Case**: Complex JavaScript logic, custom transformations
**Best For**: Advanced data processing

#### Common Functions
```javascript
// Data validation
const items = $input.all();
const validItems = [];
const errors = [];

for (const item of items) {
  const data = item.json;
  
  if (!data.email || !data.email.includes('@')) {
    errors.push(`Invalid email: ${data.email}`);
    continue;
  }
  
  if (!data.amount || data.amount < 0) {
    errors.push(`Invalid amount: ${data.amount}`);
    continue;
  }
  
  validItems.push({
    json: {
      ...data,
      validated: true,
      validatedAt: new Date().toISOString()
    }
  });
}

return validItems;
```

```javascript
// Data aggregation
const items = $input.all();
const summary = {
  totalItems: items.length,
  totalAmount: 0,
  averageAmount: 0,
  categories: {},
  topCategory: ''
};

items.forEach(item => {
  const data = item.json;
  summary.totalAmount += data.amount;
  
  if (!summary.categories[data.category]) {
    summary.categories[data.category] = 0;
  }
  summary.categories[data.category]++;
});

summary.averageAmount = summary.totalAmount / summary.totalItems;
summary.topCategory = Object.keys(summary.categories)
  .reduce((a, b) => summary.categories[a] > summary.categories[b] ? a : b);

return [{ json: summary }];
```

### 3. JSON Node 📄

**Use Case**: JSON parsing and stringification
**Best For**: API response processing

#### Operations
```javascript
// Parse JSON string
Mode: Parse
JSON String: {{ $json.rawJsonString }}

// Stringify object
Mode: Stringify
JSON Object: {{ $json.dataObject }}

// Flatten nested JSON
Mode: Flatten
Separator: .
```

### 4. Date & Time Node 📅

**Use Case**: Date calculations, formatting, timezone conversion
**Best For**: Time-based logic

#### Common Operations
```javascript
// Current date/time
{{ DateTime.now().toISO() }}
{{ DateTime.now().toFormat('yyyy-MM-dd HH:mm:ss') }}

// Date arithmetic
{{ DateTime.now().plus({days: 7}).toISO() }}
{{ DateTime.now().minus({hours: 2}).toISO() }}

// Parse and format
{{ DateTime.fromISO($json.dateString).toFormat('MMM dd, yyyy') }}
{{ DateTime.fromFormat($json.date, 'MM/dd/yyyy').toISO() }}

// Timezone conversion
{{ DateTime.now().setZone('America/New_York').toFormat('HH:mm') }}
{{ DateTime.fromISO($json.utcDate).setZone('Europe/London').toISO() }}

// Date comparisons
{{ DateTime.fromISO($json.date1) > DateTime.fromISO($json.date2) }}
{{ DateTime.now().diff(DateTime.fromISO($json.startDate), 'days').days }}
```

## 🔗 Integration Nodes

Connect with popular services and platforms.

### 1. Google Workspace 📧

#### Gmail
```javascript
// Send email
To: {{ $json.recipient }}
Subject: {{ $json.subject }}
Message: {{ $json.body }}
Attachments: {{ $json.files }}

// Search emails
Query: from:{{ $json.sender }} subject:{{ $json.keyword }}
Max Results: 50
```

#### Google Sheets
```javascript
// Append row
Range: Sheet1!A:Z
Values: [
  "{{ $json.name }}",
  "{{ $json.email }}",
  "{{ $json.date }}"
]

// Read range
Range: Sheet1!A1:Z100
Value Render Option: FORMATTED_VALUE
```

### 2. Slack 💬

```javascript
// Send message
Channel: #general
Text: 🚨 Alert: {{ $json.message }}
Username: N8N Bot

// Send rich message
Blocks: [
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*{{ $json.title }}*\n{{ $json.description }}"
    }
  }
]
```

### 3. Microsoft 365 📊

#### Excel Online
```javascript
// Add row to table
Table ID: Table1
Values: {
  "Name": "{{ $json.name }}",
  "Email": "{{ $json.email }}",
  "Status": "{{ $json.status }}"
}
```

#### Teams
```javascript
// Send message
Team ID: {{ $json.teamId }}
Channel ID: {{ $json.channelId }}
Message: {{ $json.message }}
```

## 🛠 Utility Nodes

Helper nodes for common operations.

### 1. No Operation (Debug) 🔍

**Use Case**: Debugging, data inspection
**Best For**: Development and troubleshooting

```javascript
// Add anywhere in workflow to inspect data
// Outputs exactly what it receives
// Perfect for debugging data flow
```

### 2. Error Trigger ⚠️

**Use Case**: Error handling, failure recovery
**Best For**: Robust error management

```javascript
// Triggers when other workflows fail
Workflow: {{ $json.workflow.name }}
Error: {{ $json.error.message }}
Node: {{ $json.node.name }}
```

### 3. Split In Batches 📦

**Use Case**: Processing large datasets
**Best For**: Performance optimization

```javascript
// Process 100 items at a time
Batch Size: 100
Reset: true

// Access batch info
{{ $json.batchSize }}
{{ $json.batchIndex }}
{{ $json.isLastBatch }}
```

## 🎯 Node Combinations

Powerful patterns using multiple nodes together.

### 1. Data Validation Pipeline
```
Webhook → Function (validate) → IF → Set (clean) → Database
                                 ↓
                              Error Response
```

### 2. Multi-Service Notification
```
Trigger → Set (format) → Email
                      → Slack  
                      → Teams
                      → SMS
```

### 3. Data Enrichment Chain
```
HTTP Request → Set → HTTP Request → Set → HTTP Request → Final Output
(Get User)   (Add)  (Get Company)  (Add)  (Get Score)
```

### 4. Conditional Processing
```
Webhook → Switch → Email (VIP customers)
                → SMS (Regular customers)
                → No Action (Inactive)
```

### 5. Error Recovery Pattern
```
Main Flow: A → B → C
           ↓   ↓   ↓
Error:     Wait → Retry → Fallback
```

## ✅ Mastery Checklist

Before moving to advanced workflows, ensure you can:

### Trigger Mastery
- [ ] Configure webhooks with proper security
- [ ] Set up complex cron schedules
- [ ] Handle email triggers with filters
- [ ] Use file triggers for data processing

### Action Mastery  
- [ ] Make authenticated HTTP requests
- [ ] Send formatted emails with templates
- [ ] Perform database operations (CRUD)
- [ ] Handle API responses and errors

### Logic Mastery
- [ ] Create complex conditional logic
- [ ] Use Switch for multi-way branching
- [ ] Merge parallel data streams
- [ ] Implement proper timing controls

### Data Mastery
- [ ] Transform data with Set node
- [ ] Write custom JavaScript functions
- [ ] Parse and manipulate JSON
- [ ] Handle dates and times correctly

### Integration Mastery
- [ ] Connect to major platforms (Google, Microsoft, Slack)
- [ ] Handle authentication properly
- [ ] Manage rate limits and quotas
- [ ] Process different data formats

**Score: 18/25 or higher?** You're ready for [Advanced Workflows](./05-advanced-workflows.md)!

## 🚀 Next Steps

You now have solid mastery of N8N's core nodes! Here's how to continue:

1. **Practice**: Build workflows using different node combinations
2. **Experiment**: Try integrations with your actual tools
3. **Optimize**: Focus on performance and error handling
4. **Learn**: Move on to [Advanced Workflows](./05-advanced-workflows.md)

**Remember**: The power of N8N comes from combining these nodes creatively. The best automations often use simple nodes in clever ways! 🎯

---

**Ready for complexity?** Continue with [Advanced Workflows →](./05-advanced-workflows.md)
