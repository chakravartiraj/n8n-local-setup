# Webhooks & Triggers in N8N

Master the art of creating reactive automation workflows with webhooks and triggers.

## Table of Contents
- [Understanding Triggers](#understanding-triggers)
- [Webhook Fundamentals](#webhook-fundamentals)
- [Setting Up Webhooks](#setting-up-webhooks)
- [Webhook Security](#webhook-security)
- [Event-Driven Workflows](#event-driven-workflows)
- [Advanced Trigger Patterns](#advanced-trigger-patterns)
- [Real-world Examples](#real-world-examples)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

## Understanding Triggers

### What are Triggers?
Triggers are the starting points of workflows that respond to events or conditions.

### Types of Triggers
- **Manual Trigger**: Execute workflows manually
- **Webhook Trigger**: HTTP requests trigger workflows
- **Schedule Trigger**: Time-based execution (cron jobs)
- **Email Trigger**: Email reception triggers
- **File Trigger**: File system changes
- **App Triggers**: Third-party service events

### Trigger vs Polling
```
Trigger (Push):
Service → Webhook → N8N → Action

Polling (Pull):
N8N → Check Service → Action (if change detected)
```

## Webhook Fundamentals

### What is a Webhook?
A webhook is an HTTP callback - a way for an application to provide real-time information to another application.

### Webhook Flow
```
1. Event occurs in external service
2. Service sends HTTP POST to your webhook URL
3. N8N receives and processes the request
4. Workflow executes based on the data
```

### Webhook URL Structure
```
https://your-n8n-instance.com/webhook/webhook-path
https://your-n8n-instance.com/webhook-test/webhook-path (for testing)
```

## Setting Up Webhooks

### Basic Webhook Node Configuration
```json
{
  "httpMethod": "POST",
  "path": "my-webhook",
  "responseMode": "responseNode",
  "options": {
    "noResponseBody": false
  }
}
```

### Dynamic Webhook Paths
```javascript
// Use expressions for dynamic paths
"user-{{$json.userId}}/events"
"orders/{{$json.storeId}}/status-update"
```

### Multiple HTTP Methods
```json
{
  "httpMethod": "ANY",
  "path": "api/endpoint",
  "options": {
    "allowedMethods": ["GET", "POST", "PUT", "DELETE"]
  }
}
```

### Processing Webhook Data
```javascript
// Access webhook data
const webhookData = $json;
const headers = $headers;
const queryParams = $query;
const httpMethod = $method;

// Process the incoming data
return [{
  json: {
    receivedAt: new Date().toISOString(),
    method: httpMethod,
    data: webhookData,
    userAgent: headers['user-agent'],
    sourceIP: headers['x-forwarded-for'] || headers['x-real-ip']
  }
}];
```

## Webhook Security

### Signature Verification
```javascript
// Verify GitHub webhook signature
const crypto = require('crypto');

const payload = JSON.stringify($json);
const signature = $headers['x-hub-signature-256'];
const secret = $credentials.webhookSecret;

const expectedSignature = 'sha256=' + crypto
  .createHmac('sha256', secret)
  .update(payload)
  .digest('hex');

if (signature !== expectedSignature) {
  throw new Error('Invalid webhook signature');
}

return [{ json: { verified: true, data: $json } }];
```

### API Key Validation
```javascript
// Validate API key from headers
const apiKey = $headers['x-api-key'];
const validKeys = $credentials.validApiKeys.split(',');

if (!apiKey || !validKeys.includes(apiKey)) {
  // Return 401 Unauthorized
  return [{
    json: { error: 'Unauthorized' },
    binary: {},
    httpCode: 401
  }];
}

// Process valid request
return [{ json: $json }];
```

### IP Whitelist
```javascript
// Check if request comes from allowed IPs
const allowedIPs = [
  '192.30.252.0/22',
  '185.199.108.0/22',
  '140.82.112.0/20'
];

const requestIP = $headers['x-forwarded-for'] || $headers['x-real-ip'];

function isIPInRange(ip, range) {
  // IP range checking logic
  // (Implementation depends on your needs)
  return true; // Simplified
}

const isAllowed = allowedIPs.some(range => isIPInRange(requestIP, range));

if (!isAllowed) {
  return [{
    json: { error: 'Forbidden' },
    httpCode: 403
  }];
}

return [{ json: $json }];
```

### Rate Limiting
```javascript
// Simple rate limiting
const rateLimit = new Map();
const clientIP = $headers['x-forwarded-for'] || 'unknown';
const now = Date.now();
const windowMs = 60000; // 1 minute
const maxRequests = 100;

// Clean old entries
for (const [ip, data] of rateLimit.entries()) {
  if (now - data.firstRequest > windowMs) {
    rateLimit.delete(ip);
  }
}

// Check current IP
if (!rateLimit.has(clientIP)) {
  rateLimit.set(clientIP, { firstRequest: now, count: 1 });
} else {
  const data = rateLimit.get(clientIP);
  data.count++;
  
  if (data.count > maxRequests) {
    return [{
      json: { error: 'Rate limit exceeded' },
      httpCode: 429
    }];
  }
}

return [{ json: $json }];
```

## Event-Driven Workflows

### GitHub Webhook Integration
```javascript
// Process GitHub webhook events
const event = $headers['x-github-event'];
const payload = $json;

switch (event) {
  case 'push':
    return [{
      json: {
        type: 'code_push',
        repository: payload.repository.full_name,
        pusher: payload.pusher.name,
        commits: payload.commits.length,
        branch: payload.ref.replace('refs/heads/', '')
      }
    }];
    
  case 'pull_request':
    return [{
      json: {
        type: 'pull_request',
        action: payload.action,
        repository: payload.repository.full_name,
        author: payload.pull_request.user.login,
        title: payload.pull_request.title,
        number: payload.pull_request.number
      }
    }];
    
  case 'issues':
    return [{
      json: {
        type: 'issue',
        action: payload.action,
        repository: payload.repository.full_name,
        author: payload.issue.user.login,
        title: payload.issue.title,
        number: payload.issue.number
      }
    }];
    
  default:
    return [{
      json: {
        type: 'unknown',
        event: event,
        ignored: true
      }
    }];
}
```

### Stripe Webhook Processing
```javascript
// Handle Stripe webhook events
const event = $json;

switch (event.type) {
  case 'payment_intent.succeeded':
    const paymentIntent = event.data.object;
    return [{
      json: {
        type: 'payment_success',
        paymentId: paymentIntent.id,
        amount: paymentIntent.amount / 100,
        currency: paymentIntent.currency,
        customer: paymentIntent.customer,
        metadata: paymentIntent.metadata
      }
    }];
    
  case 'payment_intent.payment_failed':
    const failedPayment = event.data.object;
    return [{
      json: {
        type: 'payment_failed',
        paymentId: failedPayment.id,
        amount: failedPayment.amount / 100,
        currency: failedPayment.currency,
        error: failedPayment.last_payment_error?.message,
        customer: failedPayment.customer
      }
    }];
    
  case 'customer.subscription.created':
  case 'customer.subscription.updated':
  case 'customer.subscription.deleted':
    const subscription = event.data.object;
    return [{
      json: {
        type: 'subscription_change',
        action: event.type.split('.').pop(),
        subscriptionId: subscription.id,
        customer: subscription.customer,
        status: subscription.status,
        planId: subscription.items.data[0]?.price.id
      }
    }];
    
  default:
    return [{
      json: {
        type: 'unhandled_event',
        eventType: event.type,
        eventId: event.id
      }
    }];
}
```

### Custom Application Webhooks
```javascript
// Process custom application events
const eventType = $json.event_type;
const eventData = $json.data;
const timestamp = $json.timestamp;

// Validate event structure
if (!eventType || !eventData || !timestamp) {
  return [{
    json: { error: 'Invalid event structure' },
    httpCode: 400
  }];
}

// Route based on event type
switch (eventType) {
  case 'user.created':
    return [{
      json: {
        action: 'send_welcome_email',
        user: {
          id: eventData.user_id,
          email: eventData.email,
          name: eventData.name
        },
        template: 'welcome_new_user'
      }
    }];
    
  case 'order.completed':
    return [{
      json: {
        action: 'process_order',
        order: {
          id: eventData.order_id,
          customer_id: eventData.customer_id,
          total: eventData.total_amount,
          items: eventData.items
        },
        notifications: ['email', 'sms']
      }
    }];
    
  case 'subscription.expired':
    return [{
      json: {
        action: 'handle_expiration',
        subscription: {
          id: eventData.subscription_id,
          customer_id: eventData.customer_id,
          expired_at: eventData.expired_at
        },
        grace_period_days: 7
      }
    }];
    
  default:
    return [{
      json: {
        action: 'log_unknown_event',
        event_type: eventType,
        received_at: new Date().toISOString()
      }
    }];
}
```

## Advanced Trigger Patterns

### Conditional Webhook Processing
```javascript
// Process webhooks based on conditions
const data = $json;
const conditions = {
  isHighPriority: data.priority === 'high',
  isBusinessHours: () => {
    const now = new Date();
    const hour = now.getHours();
    const day = now.getDay();
    return day >= 1 && day <= 5 && hour >= 9 && hour <= 17;
  },
  isVIPCustomer: data.customer_tier === 'vip'
};

// Determine processing path
if (conditions.isHighPriority && conditions.isVIPCustomer) {
  return [{
    json: {
      ...data,
      processing_path: 'immediate_vip',
      escalate: true,
      notify: ['sms', 'email', 'slack']
    }
  }];
} else if (conditions.isHighPriority) {
  return [{
    json: {
      ...data,
      processing_path: 'immediate_standard',
      escalate: false,
      notify: ['email', 'slack']
    }
  }];
} else if (conditions.isBusinessHours()) {
  return [{
    json: {
      ...data,
      processing_path: 'standard_business_hours',
      escalate: false,
      notify: ['email']
    }
  }];
} else {
  return [{
    json: {
      ...data,
      processing_path: 'queue_for_business_hours',
      schedule_for: '9:00 AM next business day',
      notify: []
    }
  }];
}
```

### Webhook Aggregation
```javascript
// Aggregate multiple events before processing
const aggregationKey = `${$json.user_id}_${$json.event_type}`;
const aggregationWindow = 5 * 60 * 1000; // 5 minutes
const storage = new Map(); // In real implementation, use persistent storage

// Get or create aggregation bucket
if (!storage.has(aggregationKey)) {
  storage.set(aggregationKey, {
    events: [],
    firstEventTime: Date.now(),
    lastEventTime: Date.now()
  });
}

const bucket = storage.get(aggregationKey);
bucket.events.push($json);
bucket.lastEventTime = Date.now();

// Check if aggregation window is complete
const windowExpired = Date.now() - bucket.firstEventTime > aggregationWindow;
const eventThresholdMet = bucket.events.length >= 10;

if (windowExpired || eventThresholdMet) {
  // Process aggregated events
  const result = {
    aggregation_key: aggregationKey,
    event_count: bucket.events.length,
    window_duration: bucket.lastEventTime - bucket.firstEventTime,
    first_event_time: new Date(bucket.firstEventTime).toISOString(),
    last_event_time: new Date(bucket.lastEventTime).toISOString(),
    events: bucket.events,
    summary: {
      unique_users: [...new Set(bucket.events.map(e => e.user_id))].length,
      event_types: [...new Set(bucket.events.map(e => e.event_type))]
    }
  };
  
  // Clear the bucket
  storage.delete(aggregationKey);
  
  return [{ json: result }];
} else {
  // Return empty to wait for more events
  return [];
}
```

### Webhook Transformation Pipeline
```javascript
// Multi-stage webhook processing pipeline
const stages = {
  validate: (data) => {
    const required = ['id', 'type', 'timestamp'];
    const missing = required.filter(field => !data[field]);
    
    if (missing.length > 0) {
      throw new Error(`Missing required fields: ${missing.join(', ')}`);
    }
    
    return data;
  },
  
  normalize: (data) => {
    return {
      id: String(data.id),
      type: data.type.toLowerCase(),
      timestamp: new Date(data.timestamp).toISOString(),
      payload: data.payload || {},
      metadata: {
        source: data.source || 'unknown',
        version: data.version || '1.0',
        processed_at: new Date().toISOString()
      }
    };
  },
  
  enrich: (data) => {
    // Add contextual information
    data.metadata.processing_region = 'us-east-1';
    data.metadata.workflow_id = 'webhook-processor-v2';
    
    // Add computed fields
    if (data.type === 'user_action') {
      data.computed = {
        is_weekend: new Date().getDay() % 6 === 0,
        hour_of_day: new Date().getHours(),
        is_mobile: data.payload.user_agent?.includes('Mobile') || false
      };
    }
    
    return data;
  },
  
  route: (data) => {
    const routes = {
      'user_action': 'user-analytics-queue',
      'order_event': 'order-processing-queue',
      'system_event': 'monitoring-queue'
    };
    
    data.routing = {
      destination: routes[data.type] || 'default-queue',
      priority: data.payload.priority || 'normal',
      retry_policy: 'exponential_backoff'
    };
    
    return data;
  }
};

// Execute pipeline
try {
  let processedData = $json;
  
  for (const [stageName, stageFunction] of Object.entries(stages)) {
    processedData = stageFunction(processedData);
    processedData.metadata.pipeline_stages = processedData.metadata.pipeline_stages || [];
    processedData.metadata.pipeline_stages.push({
      stage: stageName,
      completed_at: new Date().toISOString()
    });
  }
  
  return [{ json: processedData }];
} catch (error) {
  return [{
    json: {
      error: true,
      message: error.message,
      original_data: $json,
      failed_at: new Date().toISOString()
    }
  }];
}
```

## Real-world Examples

### E-commerce Order Processing
```javascript
// Complete e-commerce webhook handler
const orderEvent = $json;

// Validate order structure
const requiredFields = ['order_id', 'customer_id', 'status', 'items', 'total'];
const missingFields = requiredFields.filter(field => !orderEvent[field]);

if (missingFields.length > 0) {
  return [{
    json: { 
      error: `Missing required fields: ${missingFields.join(', ')}`,
      httpCode: 400
    }
  }];
}

// Process based on order status
switch (orderEvent.status) {
  case 'created':
    return [{
      json: {
        actions: [
          {
            type: 'inventory_reserve',
            items: orderEvent.items.map(item => ({
              sku: item.sku,
              quantity: item.quantity
            }))
          },
          {
            type: 'payment_authorize',
            amount: orderEvent.total,
            payment_method: orderEvent.payment_method
          },
          {
            type: 'send_confirmation_email',
            customer_id: orderEvent.customer_id,
            order_id: orderEvent.order_id
          }
        ]
      }
    }];
    
  case 'paid':
    return [{
      json: {
        actions: [
          {
            type: 'inventory_commit',
            order_id: orderEvent.order_id
          },
          {
            type: 'create_shipment',
            order_id: orderEvent.order_id,
            priority: orderEvent.customer_tier === 'premium' ? 'express' : 'standard'
          },
          {
            type: 'update_customer_analytics',
            customer_id: orderEvent.customer_id,
            purchase_amount: orderEvent.total
          }
        ]
      }
    }];
    
  case 'shipped':
    return [{
      json: {
        actions: [
          {
            type: 'send_tracking_email',
            customer_id: orderEvent.customer_id,
            tracking_number: orderEvent.tracking_number
          },
          {
            type: 'schedule_delivery_notification',
            estimated_delivery: orderEvent.estimated_delivery
          }
        ]
      }
    }];
    
  case 'cancelled':
    return [{
      json: {
        actions: [
          {
            type: 'inventory_release',
            order_id: orderEvent.order_id
          },
          {
            type: 'process_refund',
            order_id: orderEvent.order_id,
            amount: orderEvent.total
          },
          {
            type: 'send_cancellation_email',
            customer_id: orderEvent.customer_id
          }
        ]
      }
    }];
    
  default:
    return [{
      json: {
        actions: [
          {
            type: 'log_unknown_status',
            status: orderEvent.status,
            order_id: orderEvent.order_id
          }
        ]
      }
    }];
}
```

### Customer Support Ticket System
```javascript
// Support ticket webhook processor
const ticket = $json;

// Determine urgency and routing
function calculatePriority(ticket) {
  let score = 0;
  
  // Customer tier
  if (ticket.customer_tier === 'enterprise') score += 50;
  else if (ticket.customer_tier === 'pro') score += 30;
  else if (ticket.customer_tier === 'plus') score += 10;
  
  // Issue type
  const urgentKeywords = ['down', 'broken', 'urgent', 'critical', 'security'];
  const description = (ticket.description || '').toLowerCase();
  if (urgentKeywords.some(keyword => description.includes(keyword))) {
    score += 40;
  }
  
  // Customer history
  if (ticket.customer_satisfaction_score < 3) score += 20;
  if (ticket.open_tickets_count > 3) score += 15;
  
  return score;
}

const priorityScore = calculatePriority(ticket);
let priority, assignmentGroup, responseTime;

if (priorityScore >= 80) {
  priority = 'critical';
  assignmentGroup = 'senior-support';
  responseTime = '15 minutes';
} else if (priorityScore >= 50) {
  priority = 'high';
  assignmentGroup = 'standard-support';
  responseTime = '2 hours';
} else if (priorityScore >= 20) {
  priority = 'normal';
  assignmentGroup = 'general-support';
  responseTime = '24 hours';
} else {
  priority = 'low';
  assignmentGroup = 'community-support';
  responseTime = '48 hours';
}

// Auto-assign based on expertise
function findBestAgent(ticket, group) {
  // This would integrate with your staff management system
  const agents = {
    'senior-support': ['alice@company.com', 'bob@company.com'],
    'standard-support': ['charlie@company.com', 'diana@company.com'],
    'general-support': ['eve@company.com', 'frank@company.com'],
    'community-support': ['grace@company.com']
  };
  
  // Simple round-robin assignment
  const groupAgents = agents[group] || agents['general-support'];
  const agentIndex = ticket.id % groupAgents.length;
  return groupAgents[agentIndex];
}

const assignedAgent = findBestAgent(ticket, assignmentGroup);

return [{
  json: {
    ticket_id: ticket.id,
    priority: priority,
    priority_score: priorityScore,
    assignment: {
      group: assignmentGroup,
      agent: assignedAgent,
      assigned_at: new Date().toISOString()
    },
    sla: {
      response_time: responseTime,
      resolution_target: priority === 'critical' ? '4 hours' : 
                        priority === 'high' ? '24 hours' : '5 days'
    },
    notifications: {
      customer: {
        type: 'email',
        template: 'ticket_created',
        immediate: true
      },
      agent: {
        type: priority === 'critical' ? 'sms' : 'email',
        immediate: priority === 'critical'
      },
      manager: {
        type: 'email',
        immediate: priority === 'critical',
        escalate_after: responseTime
      }
    }
  }
}];
```

### IoT Device Monitoring
```javascript
// IoT device telemetry processing
const telemetry = $json;

// Validate telemetry data
if (!telemetry.device_id || !telemetry.timestamp || !telemetry.metrics) {
  return [{
    json: { error: 'Invalid telemetry data' },
    httpCode: 400
  }];
}

// Check for anomalies
function detectAnomalies(metrics, deviceType) {
  const anomalies = [];
  const thresholds = {
    temperature: { min: -10, max: 80, critical: 90 },
    humidity: { min: 0, max: 100 },
    battery: { min: 0, max: 100, critical: 10 },
    signal_strength: { min: -120, max: -30 }
  };
  
  for (const [metric, value] of Object.entries(metrics)) {
    const threshold = thresholds[metric];
    if (!threshold) continue;
    
    if (value < threshold.min || value > threshold.max) {
      anomalies.push({
        metric,
        value,
        threshold,
        severity: threshold.critical && (value <= threshold.critical || value >= threshold.critical) ? 'critical' : 'warning'
      });
    }
  }
  
  return anomalies;
}

const anomalies = detectAnomalies(telemetry.metrics, telemetry.device_type);

// Determine device health status
let healthStatus = 'healthy';
let alertLevel = 'none';

if (anomalies.length > 0) {
  const hasCritical = anomalies.some(a => a.severity === 'critical');
  healthStatus = hasCritical ? 'critical' : 'warning';
  alertLevel = hasCritical ? 'immediate' : 'standard';
}

// Check for patterns
const batteryTrend = telemetry.metrics.battery < 20 ? 'declining' : 'stable';
const connectivityIssues = telemetry.metrics.signal_strength < -100;

return [{
  json: {
    device_id: telemetry.device_id,
    timestamp: telemetry.timestamp,
    health_status: healthStatus,
    metrics: telemetry.metrics,
    analysis: {
      anomalies: anomalies,
      battery_trend: batteryTrend,
      connectivity_issues: connectivityIssues,
      alert_level: alertLevel
    },
    actions: {
      alerts: anomalies.length > 0 ? [
        {
          type: 'device_alert',
          severity: alertLevel,
          recipients: alertLevel === 'immediate' ? ['ops-team@company.com', 'on-call@company.com'] : ['ops-team@company.com']
        }
      ] : [],
      maintenance: batteryTrend === 'declining' ? [
        {
          type: 'schedule_battery_replacement',
          priority: telemetry.metrics.battery < 15 ? 'urgent' : 'normal'
        }
      ] : []
    }
  }
}];
```

## Troubleshooting

### Common Webhook Issues

#### Webhook Not Triggering
```javascript
// Debug webhook reception
const debugInfo = {
  timestamp: new Date().toISOString(),
  method: $method,
  path: $path,
  headers: $headers,
  query: $query,
  body: $json,
  bodyRaw: $bodyRaw
};

console.log('Webhook Debug Info:', JSON.stringify(debugInfo, null, 2));

// Check common issues
const issues = [];

if (!$json || Object.keys($json).length === 0) {
  issues.push('Empty or invalid JSON body');
}

if (!$headers['content-type']?.includes('application/json')) {
  issues.push('Missing or incorrect Content-Type header');
}

if ($method !== 'POST') {
  issues.push(`Unexpected HTTP method: ${$method}`);
}

return [{
  json: {
    debug: debugInfo,
    issues: issues,
    received: true
  }
}];
```

#### Signature Verification Failures
```javascript
// Debug signature verification
const payload = $bodyRaw;
const receivedSignature = $headers['x-signature'];
const secret = $credentials.secret;

// Try different signature formats
const signatures = {
  'sha256': crypto.createHmac('sha256', secret).update(payload).digest('hex'),
  'sha256_prefix': 'sha256=' + crypto.createHmac('sha256', secret).update(payload).digest('hex'),
  'sha1': crypto.createHmac('sha1', secret).update(payload).digest('hex'),
  'sha1_prefix': 'sha1=' + crypto.createHmac('sha1', secret).update(payload).digest('hex')
};

const match = Object.entries(signatures).find(([type, sig]) => sig === receivedSignature);

return [{
  json: {
    received_signature: receivedSignature,
    computed_signatures: signatures,
    match: match ? match[0] : 'none',
    payload_length: payload.length,
    secret_length: secret.length
  }
}];
```

### Response Handling
```javascript
// Proper webhook responses
const success = true; // Your processing result

if (success) {
  // Return 200 OK with optional response data
  return [{
    json: {
      status: 'success',
      message: 'Webhook processed successfully',
      processed_at: new Date().toISOString()
    },
    httpCode: 200
  }];
} else {
  // Return appropriate error code
  return [{
    json: {
      status: 'error',
      message: 'Failed to process webhook',
      error_code: 'PROCESSING_FAILED'
    },
    httpCode: 422
  }];
}
```

## Best Practices

### Security Best Practices
```javascript
// Comprehensive security checks
function validateWebhook(headers, body, credentials) {
  const checks = {
    hasValidSignature: false,
    isFromAllowedIP: false,
    hasValidTimestamp: false,
    isWithinRateLimit: true
  };
  
  // Signature validation
  if (headers['x-signature']) {
    const expectedSig = crypto
      .createHmac('sha256', credentials.secret)
      .update(body)
      .digest('hex');
    checks.hasValidSignature = headers['x-signature'] === `sha256=${expectedSig}`;
  }
  
  // IP validation
  const clientIP = headers['x-forwarded-for'] || headers['x-real-ip'];
  checks.isFromAllowedIP = credentials.allowedIPs.includes(clientIP);
  
  // Timestamp validation (prevent replay attacks)
  const timestamp = headers['x-timestamp'];
  if (timestamp) {
    const now = Math.floor(Date.now() / 1000);
    const requestTime = parseInt(timestamp);
    checks.hasValidTimestamp = Math.abs(now - requestTime) < 300; // 5 minutes
  }
  
  return checks;
}

const validation = validateWebhook($headers, $bodyRaw, $credentials);
const isValid = Object.values(validation).every(check => check === true);

if (!isValid) {
  return [{
    json: { 
      error: 'Validation failed',
      details: validation
    },
    httpCode: 403
  }];
}

// Continue with processing
return [{ json: $json }];
```

### Performance Optimization
```javascript
// Optimize webhook processing
const processingStartTime = Date.now();

try {
  // Quick validation first
  if (!$json.id) {
    throw new Error('Missing required ID field');
  }
  
  // Async processing for heavy operations
  const heavyProcessing = new Promise((resolve) => {
    setTimeout(() => {
      // Simulate heavy processing
      resolve({ processed: true });
    }, 100);
  });
  
  // Don't wait for heavy processing in webhook response
  heavyProcessing.then(result => {
    console.log('Heavy processing completed:', result);
  }).catch(error => {
    console.error('Heavy processing failed:', error);
  });
  
  // Return quick response
  return [{
    json: {
      status: 'accepted',
      id: $json.id,
      processing_time_ms: Date.now() - processingStartTime
    },
    httpCode: 202 // Accepted
  }];
  
} catch (error) {
  return [{
    json: {
      status: 'error',
      message: error.message,
      processing_time_ms: Date.now() - processingStartTime
    },
    httpCode: 400
  }];
}
```

### Monitoring and Logging
```javascript
// Comprehensive webhook monitoring
const webhookMetrics = {
  timestamp: new Date().toISOString(),
  webhook_path: $path,
  http_method: $method,
  source_ip: $headers['x-forwarded-for'] || 'unknown',
  user_agent: $headers['user-agent'] || 'unknown',
  content_length: parseInt($headers['content-length'] || '0'),
  event_type: $json.type || 'unknown',
  processing_time_ms: 0
};

const startTime = Date.now();

try {
  // Your webhook processing logic here
  const result = processWebhookData($json);
  
  webhookMetrics.processing_time_ms = Date.now() - startTime;
  webhookMetrics.status = 'success';
  webhookMetrics.result_count = Array.isArray(result) ? result.length : 1;
  
  // Log metrics (integrate with your monitoring system)
  console.log('Webhook Metrics:', JSON.stringify(webhookMetrics));
  
  return result;
  
} catch (error) {
  webhookMetrics.processing_time_ms = Date.now() - startTime;
  webhookMetrics.status = 'error';
  webhookMetrics.error_message = error.message;
  
  // Log error metrics
  console.error('Webhook Error:', JSON.stringify(webhookMetrics));
  
  throw error;
}

function processWebhookData(data) {
  // Your actual processing logic
  return [{ json: { processed: data } }];
}
```

## Practice Exercises

### Exercise 1: Multi-Service Integration
Create a webhook that receives user registration events and:
1. Validates the user data
2. Creates accounts in multiple systems
3. Sends welcome emails
4. Updates analytics

### Exercise 2: Real-time Notification System
Build a webhook-based notification system that:
1. Receives events from various sources
2. Routes notifications based on user preferences
3. Handles multiple delivery channels (email, SMS, push)
4. Tracks delivery status

### Exercise 3: IoT Data Pipeline
Create a webhook pipeline for IoT devices that:
1. Receives telemetry data
2. Validates and normalizes the data
3. Detects anomalies
4. Triggers alerts and actions

## Next Steps

- Master [Custom Functions & Expressions](./09-custom-functions.md)
- Learn [Error Handling & Debugging](./10-error-handling.md)
- Explore [Performance Optimization](./11-performance-optimization.md)

---

Webhooks are the gateway to reactive automation - master them to build truly responsive systems!
