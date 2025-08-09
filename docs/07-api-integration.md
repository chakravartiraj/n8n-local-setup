# API Integration Guide

Master the art of connecting N8N with external APIs and services.

## Table of Contents
- [Understanding APIs](#understanding-apis)
- [HTTP Request Node](#http-request-node)
- [Authentication Methods](#authentication-methods)
- [Request Types & Methods](#request-types--methods)
- [Response Handling](#response-handling)
- [Error Handling](#error-handling)
- [Rate Limiting](#rate-limiting)
- [Real-world Examples](#real-world-examples)
- [Best Practices](#best-practices)

## Understanding APIs

### What is an API?
Application Programming Interface (API) allows different software applications to communicate with each other.

### Types of APIs
- **REST APIs**: Most common, uses HTTP methods
- **GraphQL**: Query language for APIs
- **SOAP**: XML-based protocol
- **WebSocket**: Real-time communication

### API Components
```
URL: https://api.example.com/v1/users
Method: GET, POST, PUT, DELETE
Headers: Authorization, Content-Type
Body: JSON data for POST/PUT requests
```

## HTTP Request Node

### Basic Configuration
```json
{
  "url": "https://api.example.com/v1/users",
  "method": "GET",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer {{$credentials.apiToken}}"
  }
}
```

### Dynamic URLs
```javascript
// Using expressions for dynamic URLs
"https://api.example.com/v1/users/{{$json.userId}}/orders"

// Building complex URLs
"https://api.example.com/search?q={{$json.query}}&limit={{$json.limit || 10}}"
```

### Query Parameters
```json
{
  "url": "https://api.example.com/v1/products",
  "qs": {
    "category": "{{$json.category}}",
    "page": "{{$json.page}}",
    "per_page": 20,
    "sort": "created_at",
    "order": "desc"
  }
}
```

## Authentication Methods

### API Key Authentication
```json
{
  "headers": {
    "X-API-Key": "{{$credentials.apiKey}}"
  }
}
```

### Bearer Token
```json
{
  "headers": {
    "Authorization": "Bearer {{$credentials.accessToken}}"
  }
}
```

### Basic Authentication
```json
{
  "auth": {
    "user": "{{$credentials.username}}",
    "password": "{{$credentials.password}}"
  }
}
```

### OAuth 2.0 Flow
```javascript
// Step 1: Get access token
const tokenResponse = await fetch('https://api.example.com/oauth/token', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: new URLSearchParams({
    grant_type: 'client_credentials',
    client_id: $credentials.clientId,
    client_secret: $credentials.clientSecret
  })
});

const tokenData = await tokenResponse.json();
const accessToken = tokenData.access_token;

// Step 2: Use token for API calls
return [{
  json: {
    accessToken: accessToken,
    expiresIn: tokenData.expires_in
  }
}];
```

## Request Types & Methods

### GET Requests
```javascript
// Simple GET request
{
  "method": "GET",
  "url": "https://jsonplaceholder.typicode.com/users",
  "headers": {
    "Accept": "application/json"
  }
}

// GET with query parameters
{
  "method": "GET", 
  "url": "https://api.github.com/search/repositories",
  "qs": {
    "q": "n8n",
    "sort": "stars",
    "order": "desc"
  }
}
```

### POST Requests
```javascript
// Create new resource
{
  "method": "POST",
  "url": "https://api.example.com/v1/users",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "name": "{{$json.name}}",
    "email": "{{$json.email}}",
    "role": "user"
  }
}
```

### PUT Requests
```javascript
// Update existing resource
{
  "method": "PUT",
  "url": "https://api.example.com/v1/users/{{$json.userId}}",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "name": "{{$json.name}}",
    "email": "{{$json.email}}",
    "updatedAt": "{{new Date().toISOString()}}"
  }
}
```

### DELETE Requests
```javascript
// Delete resource
{
  "method": "DELETE",
  "url": "https://api.example.com/v1/users/{{$json.userId}}",
  "headers": {
    "Authorization": "Bearer {{$credentials.token}}"
  }
}
```

### PATCH Requests
```javascript
// Partial update
{
  "method": "PATCH",
  "url": "https://api.example.com/v1/users/{{$json.userId}}",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "status": "{{$json.newStatus}}"
  }
}
```

## Response Handling

### Processing JSON Responses
```javascript
// Extract specific data from response
const response = $json;

return [{
  json: {
    userId: response.data.id,
    userName: response.data.attributes.name,
    userEmail: response.data.attributes.email,
    totalRecords: response.meta.total,
    currentPage: response.meta.page
  }
}];
```

### Handling Arrays
```javascript
// Process array responses
const users = $json.data;

const processedUsers = users.map(user => ({
  json: {
    id: user.id,
    name: user.name,
    email: user.email,
    isActive: user.status === 'active',
    joinedDate: new Date(user.created_at).toISOString()
  }
}));

return processedUsers;
```

### Pagination Handling
```javascript
// Handle paginated responses
const allResults = [];
let currentPage = 1;
let hasMore = true;

while (hasMore) {
  const response = await fetch(`https://api.example.com/users?page=${currentPage}&per_page=100`, {
    headers: {
      'Authorization': `Bearer ${$credentials.token}`
    }
  });
  
  const data = await response.json();
  allResults.push(...data.results);
  
  hasMore = data.has_more;
  currentPage++;
  
  // Safety break
  if (currentPage > 50) break;
}

return allResults.map(item => ({ json: item }));
```

## Error Handling

### HTTP Status Codes
```javascript
// Check response status
const response = $json;
const statusCode = $httpCode;

if (statusCode >= 200 && statusCode < 300) {
  // Success
  return [{ json: response }];
} else if (statusCode === 401) {
  // Unauthorized
  throw new Error('Authentication failed');
} else if (statusCode === 429) {
  // Rate limited
  throw new Error('Rate limit exceeded');
} else {
  // Other errors
  throw new Error(`API error: ${statusCode} - ${response.message}`);
}
```

### Retry Logic
```javascript
// Implement retry with exponential backoff
async function apiCallWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      
      if (response.ok) {
        return await response.json();
      }
      
      if (response.status === 429) {
        // Rate limited - wait and retry
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Wait before retry
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Usage
try {
  const result = await apiCallWithRetry(
    'https://api.example.com/data',
    {
      method: 'GET',
      headers: { 'Authorization': `Bearer ${$credentials.token}` }
    }
  );
  
  return [{ json: result }];
} catch (error) {
  return [{
    json: {
      error: true,
      message: error.message,
      timestamp: new Date().toISOString()
    }
  }];
}
```

## Rate Limiting

### Respecting Rate Limits
```javascript
// Check rate limit headers
const headers = $response.headers;
const remaining = parseInt(headers['x-ratelimit-remaining'] || '0');
const resetTime = parseInt(headers['x-ratelimit-reset'] || '0');

if (remaining <= 5) {
  const waitTime = (resetTime * 1000) - Date.now();
  if (waitTime > 0) {
    console.log(`Rate limit nearly exceeded. Waiting ${waitTime}ms`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
  }
}

return [{ json: $json }];
```

### Implementing Rate Limiting
```javascript
// Simple rate limiter
const CALLS_PER_MINUTE = 60;
const calls = [];
const now = Date.now();

// Remove calls older than 1 minute
const recentCalls = calls.filter(time => now - time < 60000);

if (recentCalls.length >= CALLS_PER_MINUTE) {
  const oldestCall = Math.min(...recentCalls);
  const waitTime = 60000 - (now - oldestCall);
  
  throw new Error(`Rate limit exceeded. Wait ${Math.ceil(waitTime / 1000)} seconds`);
}

// Add current call
calls.push(now);

// Continue with API call
return [{ json: { canProceed: true } }];
```

## Real-world Examples

### CRM Integration (Salesforce-style)
```javascript
// Create lead in CRM
const leadData = {
  method: 'POST',
  url: 'https://api.salesforce.com/services/data/v52.0/sobjects/Lead/',
  headers: {
    'Authorization': `Bearer ${$credentials.accessToken}`,
    'Content-Type': 'application/json'
  },
  body: {
    FirstName: $json.firstName,
    LastName: $json.lastName,
    Email: $json.email,
    Company: $json.company,
    LeadSource: 'Website',
    Status: 'Open - Not Contacted'
  }
};

// Update contact if lead creation fails
if ($httpCode === 400 && $json[0].errorCode === 'DUPLICATES_DETECTED') {
  // Handle duplicate - update existing contact
  const contactId = $json[0].duplicateResult.matchResults[0].matchRecords[0].record.Id;
  
  return [{
    json: {
      method: 'PATCH',
      url: `https://api.salesforce.com/services/data/v52.0/sobjects/Contact/${contactId}`,
      headers: leadData.headers,
      body: {
        ...leadData.body,
        LastModifiedDate: new Date().toISOString()
      }
    }
  }];
}

return [{ json: $json }];
```

### E-commerce Integration (Shopify-style)
```javascript
// Get orders and process
const orders = await fetch('https://mystore.myshopify.com/admin/api/2023-01/orders.json', {
  headers: {
    'X-Shopify-Access-Token': $credentials.accessToken
  }
});

const orderData = await orders.json();

const processedOrders = orderData.orders.map(order => {
  const customerData = {
    id: order.customer.id,
    email: order.customer.email,
    name: `${order.customer.first_name} ${order.customer.last_name}`,
    totalOrders: order.customer.orders_count,
    totalSpent: parseFloat(order.customer.total_spent)
  };

  const orderSummary = {
    id: order.id,
    orderNumber: order.order_number,
    totalPrice: parseFloat(order.total_price),
    currency: order.currency,
    createdAt: order.created_at,
    status: order.financial_status,
    itemCount: order.line_items.length
  };

  return {
    json: {
      customer: customerData,
      order: orderSummary,
      lineItems: order.line_items
    }
  };
});

return processedOrders;
```

### Social Media Integration (Twitter-style)
```javascript
// Post tweet with media
const tweetData = {
  method: 'POST',
  url: 'https://api.twitter.com/2/tweets',
  headers: {
    'Authorization': `Bearer ${$credentials.bearerToken}`,
    'Content-Type': 'application/json'
  },
  body: {
    text: $json.tweetText
  }
};

// Add media if present
if ($json.mediaIds && $json.mediaIds.length > 0) {
  tweetData.body.media = {
    media_ids: $json.mediaIds
  };
}

// Add thread if this is a reply
if ($json.replyToTweetId) {
  tweetData.body.reply = {
    in_reply_to_tweet_id: $json.replyToTweetId
  };
}

return [{ json: tweetData }];
```

### Payment Processing (Stripe-style)
```javascript
// Create payment intent
const paymentIntent = {
  method: 'POST',
  url: 'https://api.stripe.com/v1/payment_intents',
  headers: {
    'Authorization': `Bearer ${$credentials.secretKey}`,
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: new URLSearchParams({
    amount: Math.round($json.amount * 100), // Convert to cents
    currency: $json.currency || 'usd',
    customer: $json.customerId,
    description: $json.description,
    'metadata[order_id]': $json.orderId,
    'metadata[customer_email]': $json.customerEmail
  }).toString()
};

// Handle response
const response = $json;

if (response.status === 'requires_action') {
  return [{
    json: {
      requiresAction: true,
      clientSecret: response.client_secret,
      nextAction: response.next_action
    }
  }];
} else if (response.status === 'succeeded') {
  return [{
    json: {
      success: true,
      paymentIntentId: response.id,
      amount: response.amount,
      currency: response.currency
    }
  }];
}

return [{ json: response }];
```

## Best Practices

### API Security
```javascript
// Secure credential handling
const credentials = {
  apiKey: $credentials.apiKey, // Never log or expose
  baseUrl: 'https://api.example.com'
};

// Validate SSL certificates
const options = {
  rejectUnauthorized: true, // Always validate SSL
  timeout: 30000 // Set reasonable timeout
};
```

### Request Optimization
```javascript
// Batch API calls
const batchRequests = [];
const BATCH_SIZE = 10;

for (let i = 0; i < $input.all().length; i += BATCH_SIZE) {
  const batch = $input.all().slice(i, i + BATCH_SIZE);
  
  const batchRequest = {
    method: 'POST',
    url: 'https://api.example.com/batch',
    body: {
      requests: batch.map(item => ({
        method: 'GET',
        url: `/users/${item.json.userId}`
      }))
    }
  };
  
  batchRequests.push(batchRequest);
}

return batchRequests.map(req => ({ json: req }));
```

### Response Caching
```javascript
// Simple caching mechanism
const cache = new Map();
const cacheKey = `${$json.endpoint}_${$json.params}`;
const cacheExpiry = 5 * 60 * 1000; // 5 minutes

// Check cache
if (cache.has(cacheKey)) {
  const cached = cache.get(cacheKey);
  if (Date.now() - cached.timestamp < cacheExpiry) {
    return [{ json: cached.data }];
  }
}

// Make API call
const response = $json;

// Cache response
cache.set(cacheKey, {
  data: response,
  timestamp: Date.now()
});

return [{ json: response }];
```

### Monitoring & Logging
```javascript
// API call monitoring
const startTime = Date.now();

try {
  const response = await fetch($json.url, $json.options);
  const endTime = Date.now();
  
  // Log successful call
  console.log(`API Call: ${$json.method} ${$json.url} - ${response.status} (${endTime - startTime}ms)`);
  
  return [{
    json: {
      data: await response.json(),
      meta: {
        status: response.status,
        responseTime: endTime - startTime,
        timestamp: new Date().toISOString()
      }
    }
  }];
} catch (error) {
  const endTime = Date.now();
  
  // Log failed call
  console.error(`API Call Failed: ${$json.method} ${$json.url} - ${error.message} (${endTime - startTime}ms)`);
  
  throw error;
}
```

## Practice Exercises

### Exercise 1: GitHub API Integration
Create a workflow that:
1. Fetches your GitHub repositories
2. Gets commit information for each repo
3. Calculates activity metrics
4. Sends a summary report

### Exercise 2: Multi-API Data Aggregation
Build a workflow that combines data from:
1. Weather API for location data
2. News API for local news
3. Social media API for trending topics
4. Creates a personalized daily digest

### Exercise 3: E-commerce Sync
Create a system that:
1. Syncs products between platforms
2. Updates inventory levels
3. Handles price changes
4. Manages product variants

## Next Steps

- Learn about [Webhooks & Triggers](./08-webhooks-triggers.md)
- Explore [Custom Functions & Expressions](./09-custom-functions.md)
- Master [Error Handling & Debugging](./10-error-handling.md)

---

Master API integration and unlock the full potential of automation by connecting any service!
