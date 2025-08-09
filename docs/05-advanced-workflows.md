# Advanced N8N Workflows

Master complex automation patterns, advanced techniques, and enterprise-grade workflow design. This guide takes your N8N skills from intermediate to expert level.

## 📋 Table of Contents
- [Complex Workflow Patterns](#complex-workflow-patterns)
- [Advanced Data Processing](#advanced-data-processing)
- [Error Handling & Recovery](#error-handling--recovery)
- [Performance Optimization](#performance-optimization)
- [Reusable Components](#reusable-components)
- [Enterprise Patterns](#enterprise-patterns)
- [Advanced Integrations](#advanced-integrations)

## 🏗 Complex Workflow Patterns

### 1. Multi-Stage Data Processing Pipeline

Build sophisticated data transformation workflows that handle complex business logic.

#### Pattern: E-commerce Order Processing System
```
Order Webhook → Validate Order → Check Inventory → Calculate Tax → Process Payment → Update Inventory → Send Confirmations
       ↓              ↓              ↓             ↓              ↓                ↓                  ↓
   Log Entry    Fraud Check    Backorder Queue   Tax Service   Payment Gateway   Inventory DB      Multi-channel
                     ↓              ↓             ↓              ↓                ↓                  ↓
               Risk Assessment  Supplier Alert  Audit Log    Receipt Email    Stock Alert      Email + SMS + Push
```

#### Implementation Strategy
```javascript
// Main validation function
function validateOrder(orderData) {
  const validations = {
    requiredFields: ['customerId', 'items', 'shippingAddress'],
    businessRules: {
      minOrderValue: 1.00,
      maxItemQuantity: 100,
      validPaymentMethods: ['credit_card', 'paypal', 'bank_transfer']
    }
  };
  
  const errors = [];
  
  // Field validation
  validations.requiredFields.forEach(field => {
    if (!orderData[field]) {
      errors.push(`Missing required field: ${field}`);
    }
  });
  
  // Business rule validation
  const totalValue = orderData.items.reduce((sum, item) => 
    sum + (item.price * item.quantity), 0);
    
  if (totalValue < validations.businessRules.minOrderValue) {
    errors.push('Order value below minimum threshold');
  }
  
  return {
    isValid: errors.length === 0,
    errors,
    validatedOrder: errors.length === 0 ? {
      ...orderData,
      totalValue,
      validatedAt: new Date().toISOString()
    } : null
  };
}
```

### 2. Event-Driven Architecture Pattern

Create workflows that respond to and generate events across your system.

#### Pattern: Customer Journey Orchestration
```
Customer Events → Event Router → Journey Engine → Action Dispatcher → Channel Handlers
       ↓              ↓              ↓               ↓                    ↓
   Registration    Determine     Update State    Select Actions    Email/SMS/Push
   Purchase        Journey       Track Progress  Schedule Future   In-app/Web
   Support         Stage         Persist Data    Queue Retries     Social/Ads
   Cancellation    Exit Conditions Score Lead    Handle Failures   Webhook/API
```

#### Implementation: Event Router
```javascript
// Event classification and routing
const eventClassifier = {
  'user.registered': {
    priority: 'high',
    category: 'lifecycle',
    triggers: ['welcome_series', 'onboarding_flow'],
    timeout: 3600 // 1 hour to complete
  },
  'order.completed': {
    priority: 'high', 
    category: 'transaction',
    triggers: ['receipt_email', 'upsell_campaign', 'review_request'],
    timeout: 86400 // 24 hours
  },
  'support.ticket.created': {
    priority: 'critical',
    category: 'service',
    triggers: ['auto_acknowledge', 'escalation_check', 'satisfaction_survey'],
    timeout: 900 // 15 minutes
  }
};

// Process incoming event
const event = $json;
const eventType = event.type;
const classification = eventClassifier[eventType];

if (!classification) {
  return [{
    json: {
      status: 'unhandled',
      event: event,
      reason: 'Unknown event type'
    }
  }];
}

// Generate workflow triggers
const workflows = classification.triggers.map(trigger => ({
  workflow: trigger,
  priority: classification.priority,
  timeout: classification.timeout,
  event: event,
  scheduledFor: new Date().toISOString()
}));

return workflows.map(w => ({ json: w }));
```

### 3. Parallel Processing with Synchronization

Handle multiple operations simultaneously and synchronize results.

#### Pattern: Multi-API Data Enrichment
```javascript
// Parallel API calls for user enrichment
const userId = $json.userId;

// Configure parallel endpoints
const enrichmentAPIs = [
  {
    name: 'userProfile',
    url: `https://api.userservice.com/users/${userId}`,
    timeout: 5000
  },
  {
    name: 'socialData', 
    url: `https://api.social.com/profile/${userId}`,
    timeout: 3000
  },
  {
    name: 'creditScore',
    url: `https://api.credit.com/score/${userId}`,
    timeout: 7000
  },
  {
    name: 'preferences',
    url: `https://api.preferences.com/user/${userId}`,
    timeout: 2000
  }
];

// Return configuration for parallel HTTP requests
return enrichmentAPIs.map(api => ({
  json: {
    apiName: api.name,
    url: api.url,
    timeout: api.timeout,
    userId: userId
  }
}));
```

#### Synchronization Logic
```javascript
// Merge enrichment results
const results = $input.all();
const enrichedProfile = {
  userId: results[0].json.userId,
  timestamp: new Date().toISOString(),
  completeness: 0,
  sources: {}
};

results.forEach(result => {
  const apiName = result.json.apiName;
  
  if (result.json.success) {
    enrichedProfile.sources[apiName] = result.json.data;
    enrichedProfile.completeness += 25; // Each API adds 25%
  } else {
    enrichedProfile.sources[apiName] = {
      error: result.json.error,
      fallback: true
    };
  }
});

// Calculate enrichment score
enrichedProfile.score = enrichedProfile.completeness / 100;
enrichedProfile.quality = enrichedProfile.score > 0.75 ? 'high' : 
                         enrichedProfile.score > 0.5 ? 'medium' : 'low';

return [{ json: enrichedProfile }];
```

## 📊 Advanced Data Processing

### 1. Complex Data Transformations

Handle sophisticated data manipulation scenarios.

#### Multi-Format Data Normalization
```javascript
// Universal data normalizer for different input formats
function normalizeData(input) {
  const normalizers = {
    csv: (data) => {
      const lines = data.split('\n');
      const headers = lines[0].split(',').map(h => h.trim());
      return lines.slice(1).map(line => {
        const values = line.split(',').map(v => v.trim());
        const obj = {};
        headers.forEach((header, index) => {
          obj[header] = values[index];
        });
        return obj;
      });
    },
    
    xml: (data) => {
      // Simplified XML parsing (use proper XML parser in production)
      const items = [];
      const itemMatches = data.match(/<item[^>]*>(.*?)<\/item>/gs);
      if (itemMatches) {
        itemMatches.forEach(item => {
          const obj = {};
          const fieldMatches = item.match(/<([^>]+)>([^<]*)<\/[^>]+>/g);
          if (fieldMatches) {
            fieldMatches.forEach(field => {
              const match = field.match(/<([^>]+)>([^<]*)<\/[^>]+>/);
              if (match) {
                obj[match[1]] = match[2];
              }
            });
          }
          items.push(obj);
        });
      }
      return items;
    },
    
    json: (data) => {
      return Array.isArray(data) ? data : [data];
    },
    
    fixedWidth: (data, schema) => {
      const lines = data.split('\n');
      return lines.map(line => {
        const obj = {};
        let position = 0;
        schema.fields.forEach(field => {
          obj[field.name] = line.substr(position, field.length).trim();
          position += field.length;
        });
        return obj;
      });
    }
  };
  
  const format = input.format;
  const normalizer = normalizers[format];
  
  if (!normalizer) {
    throw new Error(`Unsupported format: ${format}`);
  }
  
  return normalizer(input.data, input.schema);
}
```

### 2. Advanced Data Validation & Cleansing

Implement comprehensive data quality controls.

#### Enterprise Data Validation Framework
```javascript
// Comprehensive validation framework
class DataValidator {
  constructor() {
    this.rules = {
      required: (value) => value !== null && value !== undefined && value !== '',
      email: (value) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
      phone: (value) => /^\+?[\d\s\-\(\)]{10,}$/.test(value),
      url: (value) => /^https?:\/\/.+/.test(value),
      number: (value) => !isNaN(parseFloat(value)),
      integer: (value) => Number.isInteger(Number(value)),
      range: (value, min, max) => {
        const num = Number(value);
        return num >= min && num <= max;
      },
      length: (value, min, max) => {
        const str = String(value);
        return str.length >= min && str.length <= max;
      },
      pattern: (value, regex) => new RegExp(regex).test(value),
      custom: (value, func) => func(value)
    };
  }
  
  validate(data, schema) {
    const errors = [];
    const cleaned = {};
    
    Object.keys(schema).forEach(field => {
      const fieldSchema = schema[field];
      const value = data[field];
      const fieldErrors = [];
      
      // Check each rule
      if (fieldSchema.rules) {
        fieldSchema.rules.forEach(rule => {
          const ruleName = Object.keys(rule)[0];
          const ruleParams = rule[ruleName];
          const validator = this.rules[ruleName];
          
          if (validator && !validator(value, ...ruleParams)) {
            fieldErrors.push({
              field,
              rule: ruleName,
              message: fieldSchema.messages?.[ruleName] || `${field} failed ${ruleName} validation`
            });
          }
        });
      }
      
      // Apply transformations if valid
      if (fieldErrors.length === 0 && fieldSchema.transform) {
        cleaned[field] = this.transform(value, fieldSchema.transform);
      } else {
        cleaned[field] = value;
      }
      
      errors.push(...fieldErrors);
    });
    
    return {
      isValid: errors.length === 0,
      errors,
      data: cleaned
    };
  }
  
  transform(value, transformations) {
    let result = value;
    
    transformations.forEach(transform => {
      switch (transform.type) {
        case 'trim':
          result = String(result).trim();
          break;
        case 'lowercase':
          result = String(result).toLowerCase();
          break;
        case 'uppercase':
          result = String(result).toUpperCase();
          break;
        case 'capitalize':
          result = String(result).charAt(0).toUpperCase() + String(result).slice(1).toLowerCase();
          break;
        case 'remove':
          result = String(result).replace(new RegExp(transform.pattern, 'g'), '');
          break;
        case 'replace':
          result = String(result).replace(new RegExp(transform.from, 'g'), transform.to);
          break;
        case 'format_phone':
          result = String(result).replace(/\D/g, '');
          if (result.length === 10) result = `+1${result}`;
          break;
        case 'format_currency':
          result = parseFloat(result).toFixed(2);
          break;
      }
    });
    
    return result;
  }
}

// Usage example
const validator = new DataValidator();
const schema = {
  name: {
    rules: [
      { required: [] },
      { length: [2, 50] }
    ],
    transform: [
      { type: 'trim' },
      { type: 'capitalize' }
    ],
    messages: {
      required: 'Name is required',
      length: 'Name must be between 2 and 50 characters'
    }
  },
  email: {
    rules: [
      { required: [] },
      { email: [] }
    ],
    transform: [
      { type: 'trim' },
      { type: 'lowercase' }
    ]
  },
  phone: {
    rules: [
      { phone: [] }
    ],
    transform: [
      { type: 'format_phone' }
    ]
  }
};

return [{ 
  json: validator.validate($json, schema) 
}];
```

## ⚠️ Error Handling & Recovery

### 1. Comprehensive Error Handling Strategy

Build resilient workflows that gracefully handle failures.

#### Multi-Level Error Handling
```javascript
// Global error handling framework
class WorkflowErrorHandler {
  constructor() {
    this.errorTypes = {
      validation: { retryable: false, severity: 'low' },
      network: { retryable: true, severity: 'medium', maxRetries: 3 },
      authentication: { retryable: false, severity: 'high' },
      rateLimit: { retryable: true, severity: 'medium', backoff: 'exponential' },
      server: { retryable: true, severity: 'high', maxRetries: 2 },
      business: { retryable: false, severity: 'medium' }
    };
  }
  
  handleError(error, context) {
    const errorType = this.classifyError(error);
    const config = this.errorTypes[errorType];
    
    const errorInfo = {
      type: errorType,
      message: error.message,
      context: context,
      timestamp: new Date().toISOString(),
      severity: config.severity,
      retryable: config.retryable,
      attempts: context.attempts || 0
    };
    
    // Determine next action
    if (config.retryable && errorInfo.attempts < (config.maxRetries || 3)) {
      return this.scheduleRetry(errorInfo, config);
    } else {
      return this.escalateError(errorInfo);
    }
  }
  
  classifyError(error) {
    const message = error.message.toLowerCase();
    
    if (message.includes('validation') || message.includes('invalid')) {
      return 'validation';
    }
    if (message.includes('network') || message.includes('timeout')) {
      return 'network';
    }
    if (message.includes('unauthorized') || message.includes('forbidden')) {
      return 'authentication';
    }
    if (message.includes('rate limit') || message.includes('too many requests')) {
      return 'rateLimit';
    }
    if (message.includes('500') || message.includes('502') || message.includes('503')) {
      return 'server';
    }
    
    return 'business';
  }
  
  scheduleRetry(errorInfo, config) {
    let delay = 1000; // 1 second base delay
    
    if (config.backoff === 'exponential') {
      delay = Math.pow(2, errorInfo.attempts) * 1000;
    } else if (config.backoff === 'linear') {
      delay = (errorInfo.attempts + 1) * 1000;
    }
    
    return {
      action: 'retry',
      delay: delay,
      errorInfo: {
        ...errorInfo,
        nextRetryAt: new Date(Date.now() + delay).toISOString()
      }
    };
  }
  
  escalateError(errorInfo) {
    const escalationLevel = {
      low: 'log',
      medium: 'alert',
      high: 'incident'
    }[errorInfo.severity];
    
    return {
      action: 'escalate',
      level: escalationLevel,
      errorInfo: errorInfo,
      notifications: this.getNotificationChannels(errorInfo.severity)
    };
  }
  
  getNotificationChannels(severity) {
    const channels = {
      low: ['log'],
      medium: ['log', 'email'],
      high: ['log', 'email', 'slack', 'sms']
    };
    
    return channels[severity] || ['log'];
  }
}
```

### 2. Circuit Breaker Pattern

Prevent cascade failures and provide graceful degradation.

#### Circuit Breaker Implementation
```javascript
// Circuit breaker for external service calls
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000, monitoringPeriod = 120000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.monitoringPeriod = monitoringPeriod;
    this.failureCount = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
  }
  
  async call(serviceFunction, fallbackFunction = null) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        return this.executeFallback(fallbackFunction, 'Circuit breaker is OPEN');
      }
    }
    
    try {
      const result = await serviceFunction();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      return this.executeFallback(fallbackFunction, error.message);
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
  
  executeFallback(fallbackFunction, error) {
    if (fallbackFunction) {
      return fallbackFunction(error);
    }
    
    throw new Error(`Service unavailable: ${error}`);
  }
  
  getStatus() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}

// Usage in N8N workflow
const breaker = new CircuitBreaker(3, 30000); // 3 failures, 30s timeout

const result = await breaker.call(
  // Primary service call
  async () => {
    const response = await $http.request({
      method: 'GET',
      url: 'https://api.external-service.com/data',
      timeout: 5000
    });
    return response.data;
  },
  // Fallback function
  (error) => {
    return {
      data: 'fallback_data',
      source: 'cache',
      error: error,
      timestamp: new Date().toISOString()
    };
  }
);

return [{ json: result }];
```

## 🚀 Performance Optimization

### 1. Batch Processing Strategies

Handle large datasets efficiently.

#### Smart Batching Algorithm
```javascript
// Intelligent batch processor
class BatchProcessor {
  constructor(options = {}) {
    this.maxBatchSize = options.maxBatchSize || 100;
    this.maxProcessingTime = options.maxProcessingTime || 30000; // 30 seconds
    this.concurrentBatches = options.concurrentBatches || 3;
    this.adaptiveSizing = options.adaptiveSizing || true;
  }
  
  async process(items, processor) {
    const batches = this.createOptimalBatches(items);
    const results = [];
    
    // Process batches concurrently
    for (let i = 0; i < batches.length; i += this.concurrentBatches) {
      const currentBatches = batches.slice(i, i + this.concurrentBatches);
      
      const batchPromises = currentBatches.map((batch, index) => 
        this.processBatch(batch, processor, i + index)
      );
      
      const batchResults = await Promise.allSettled(batchPromises);
      
      batchResults.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          results.push(...result.value);
        } else {
          console.error(`Batch ${i + index} failed:`, result.reason);
          // Handle failed batch - could retry or use fallback
        }
      });
    }
    
    return results;
  }
  
  createOptimalBatches(items) {
    let batchSize = this.maxBatchSize;
    
    // Adaptive sizing based on item complexity
    if (this.adaptiveSizing) {
      const sampleItem = items[0];
      const estimatedComplexity = this.estimateComplexity(sampleItem);
      batchSize = Math.max(10, Math.floor(this.maxBatchSize / estimatedComplexity));
    }
    
    const batches = [];
    for (let i = 0; i < items.length; i += batchSize) {
      batches.push(items.slice(i, i + batchSize));
    }
    
    return batches;
  }
  
  estimateComplexity(item) {
    // Simple complexity estimation based on object size
    const jsonSize = JSON.stringify(item).length;
    if (jsonSize > 10000) return 4; // Large objects
    if (jsonSize > 1000) return 2;  // Medium objects
    return 1; // Small objects
  }
  
  async processBatch(batch, processor, batchIndex) {
    const startTime = Date.now();
    
    try {
      const results = await processor(batch, batchIndex);
      
      const processingTime = Date.now() - startTime;
      console.log(`Batch ${batchIndex} processed ${batch.length} items in ${processingTime}ms`);
      
      return results;
    } catch (error) {
      console.error(`Batch ${batchIndex} failed:`, error);
      throw error;
    }
  }
}
```

### 2. Caching Strategies

Implement intelligent caching for better performance.

#### Multi-Level Cache System
```javascript
// Advanced caching system
class WorkflowCache {
  constructor() {
    this.memoryCache = new Map();
    this.memoryCacheSize = 1000;
    this.cacheTTL = {
      'api_response': 300000,    // 5 minutes
      'user_data': 900000,      // 15 minutes
      'configuration': 3600000,  // 1 hour
      'static_data': 86400000   // 24 hours
    };
  }
  
  generateKey(operation, params) {
    const sortedParams = Object.keys(params)
      .sort()
      .reduce((result, key) => {
        result[key] = params[key];
        return result;
      }, {});
      
    return `${operation}:${JSON.stringify(sortedParams)}`;
  }
  
  get(key, cacheType = 'api_response') {
    const cached = this.memoryCache.get(key);
    
    if (!cached) return null;
    
    // Check TTL
    const ttl = this.cacheTTL[cacheType];
    if (Date.now() - cached.timestamp > ttl) {
      this.memoryCache.delete(key);
      return null;
    }
    
    return cached.data;
  }
  
  set(key, data, cacheType = 'api_response') {
    // Implement LRU eviction if cache is full
    if (this.memoryCache.size >= this.memoryCacheSize) {
      const firstKey = this.memoryCache.keys().next().value;
      this.memoryCache.delete(firstKey);
    }
    
    this.memoryCache.set(key, {
      data: data,
      timestamp: Date.now(),
      type: cacheType
    });
  }
  
  invalidate(pattern) {
    const keysToDelete = [];
    
    for (const key of this.memoryCache.keys()) {
      if (key.includes(pattern)) {
        keysToDelete.push(key);
      }
    }
    
    keysToDelete.forEach(key => this.memoryCache.delete(key));
  }
  
  getStats() {
    const stats = {
      totalEntries: this.memoryCache.size,
      byType: {},
      oldestEntry: null,
      newestEntry: null
    };
    
    let oldestTime = Date.now();
    let newestTime = 0;
    
    for (const [key, value] of this.memoryCache.entries()) {
      const type = value.type;
      stats.byType[type] = (stats.byType[type] || 0) + 1;
      
      if (value.timestamp < oldestTime) {
        oldestTime = value.timestamp;
        stats.oldestEntry = key;
      }
      
      if (value.timestamp > newestTime) {
        newestTime = value.timestamp;
        stats.newestEntry = key;
      }
    }
    
    return stats;
  }
}

// Usage in workflow
const cache = new WorkflowCache();
const cacheKey = cache.generateKey('user_profile', { userId: $json.userId });

// Try cache first
let userData = cache.get(cacheKey, 'user_data');

if (!userData) {
  // Fetch from API if not cached
  userData = await fetchUserProfile($json.userId);
  cache.set(cacheKey, userData, 'user_data');
}

return [{ json: userData }];
```

## 🔧 Reusable Components

### 1. Workflow Templates & Sub-workflows

Create modular, reusable workflow components.

#### Template Pattern: Data Validation Service
```javascript
// Reusable validation template
const ValidationTemplate = {
  name: 'Universal Data Validator',
  description: 'Configurable validation service for any data type',
  
  parameters: {
    validationRules: {
      type: 'object',
      required: true,
      description: 'Validation configuration object'
    },
    inputData: {
      type: 'object', 
      required: true,
      description: 'Data to validate'
    },
    strict: {
      type: 'boolean',
      default: false,
      description: 'Stop on first error or collect all errors'
    }
  },
  
  workflow: [
    {
      node: 'Function',
      name: 'Execute Validation',
      code: `
        const { validationRules, inputData, strict } = $json;
        const validator = new DataValidator();
        return [{ json: validator.validate(inputData, validationRules, strict) }];
      `
    },
    {
      node: 'IF',
      name: 'Check Result',
      condition: '{{ $json.isValid }}'
    },
    {
      node: 'Set',
      name: 'Success Response',
      path: 'true',
      fields: {
        status: 'valid',
        data: '{{ $json.data }}',
        timestamp: '{{ $now }}'
      }
    },
    {
      node: 'Set', 
      name: 'Error Response',
      path: 'false',
      fields: {
        status: 'invalid',
        errors: '{{ $json.errors }}',
        timestamp: '{{ $now }}'
      }
    }
  ]
};
```

### 2. Function Libraries

Build reusable function collections.

#### Utility Function Library
```javascript
// Comprehensive utility library
const N8NUtils = {
  // Data manipulation utilities
  data: {
    deepMerge: (target, source) => {
      const output = Object.assign({}, target);
      if (N8NUtils.validation.isObject(target) && N8NUtils.validation.isObject(source)) {
        Object.keys(source).forEach(key => {
          if (N8NUtils.validation.isObject(source[key])) {
            if (!(key in target))
              Object.assign(output, { [key]: source[key] });
            else
              output[key] = N8NUtils.data.deepMerge(target[key], source[key]);
          } else {
            Object.assign(output, { [key]: source[key] });
          }
        });
      }
      return output;
    },
    
    flatten: (obj, prefix = '') => {
      const flattened = {};
      Object.keys(obj).forEach(key => {
        const value = obj[key];
        const newKey = prefix ? `${prefix}.${key}` : key;
        
        if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
          Object.assign(flattened, N8NUtils.data.flatten(value, newKey));
        } else {
          flattened[newKey] = value;
        }
      });
      return flattened;
    },
    
    groupBy: (array, key) => {
      return array.reduce((groups, item) => {
        const group = item[key];
        groups[group] = groups[group] || [];
        groups[group].push(item);
        return groups;
      }, {});
    },
    
    deduplicate: (array, key) => {
      const seen = new Set();
      return array.filter(item => {
        const value = key ? item[key] : item;
        if (seen.has(value)) {
          return false;
        }
        seen.add(value);
        return true;
      });
    }
  },
  
  // Validation utilities
  validation: {
    isObject: (obj) => obj !== null && typeof obj === 'object' && !Array.isArray(obj),
    isEmail: (email) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email),
    isUrl: (url) => /^https?:\/\/.+/.test(url),
    isPhoneNumber: (phone) => /^\+?[\d\s\-\(\)]{10,}$/.test(phone),
    isJSON: (str) => {
      try {
        JSON.parse(str);
        return true;
      } catch {
        return false;
      }
    }
  },
  
  // Date utilities  
  date: {
    addBusinessDays: (date, days) => {
      const result = new Date(date);
      let addedDays = 0;
      
      while (addedDays < days) {
        result.setDate(result.getDate() + 1);
        if (result.getDay() !== 0 && result.getDay() !== 6) {
          addedDays++;
        }
      }
      
      return result;
    },
    
    getBusinessDaysCount: (startDate, endDate) => {
      const start = new Date(startDate);
      const end = new Date(endDate);
      let count = 0;
      
      while (start <= end) {
        if (start.getDay() !== 0 && start.getDay() !== 6) {
          count++;
        }
        start.setDate(start.getDate() + 1);
      }
      
      return count;
    },
    
    formatRelative: (date) => {
      const now = new Date();
      const target = new Date(date);
      const diffMs = target - now;
      const diffDays = Math.floor(diffMs / (1000 * 60 * 60 * 24));
      
      if (diffDays === 0) return 'Today';
      if (diffDays === 1) return 'Tomorrow';
      if (diffDays === -1) return 'Yesterday';
      if (diffDays > 0) return `In ${diffDays} days`;
      return `${Math.abs(diffDays)} days ago`;
    }
  },
  
  // String utilities
  string: {
    slugify: (text) => {
      return text
        .toLowerCase()
        .replace(/[^\w\s-]/g, '')
        .replace(/[\s_-]+/g, '-')
        .replace(/^-+|-+$/g, '');
    },
    
    truncate: (text, length, suffix = '...') => {
      if (text.length <= length) return text;
      return text.substring(0, length) + suffix;
    },
    
    template: (template, vars) => {
      return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
        return vars[key] || match;
      });
    }
  },
  
  // API utilities
  api: {
    retry: async (fn, retries = 3, delay = 1000) => {
      try {
        return await fn();
      } catch (error) {
        if (retries > 0) {
          await new Promise(resolve => setTimeout(resolve, delay));
          return N8NUtils.api.retry(fn, retries - 1, delay * 2);
        }
        throw error;
      }
    },
    
    batchRequests: async (requests, batchSize = 5, delay = 100) => {
      const results = [];
      
      for (let i = 0; i < requests.length; i += batchSize) {
        const batch = requests.slice(i, i + batchSize);
        const batchResults = await Promise.allSettled(batch);
        results.push(...batchResults);
        
        if (i + batchSize < requests.length) {
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
      
      return results;
    }
  }
};

// Export for use in workflows
return [{ json: N8NUtils }];
```

## ✅ Advanced Mastery Checklist

Before moving to expert level, ensure you can:

### Complex Patterns
- [ ] Design multi-stage processing pipelines
- [ ] Implement event-driven architectures
- [ ] Handle parallel processing with synchronization
- [ ] Create sophisticated data transformation workflows

### Data Processing
- [ ] Build universal data normalizers
- [ ] Implement comprehensive validation frameworks
- [ ] Handle multiple data formats efficiently
- [ ] Create data quality monitoring systems

### Error Handling
- [ ] Implement multi-level error handling strategies
- [ ] Use circuit breaker patterns
- [ ] Create custom retry logic with backoff
- [ ] Design graceful degradation systems

### Performance
- [ ] Implement intelligent batching algorithms
- [ ] Build multi-level caching systems
- [ ] Optimize workflow execution paths
- [ ] Monitor and tune performance metrics

### Reusability
- [ ] Create workflow templates and patterns
- [ ] Build comprehensive utility libraries
- [ ] Design modular, composable components
- [ ] Implement configuration-driven workflows

**Score: 18/25 or higher?** You're ready for [Enterprise Patterns](./13-enterprise-patterns.md)!

## 🚀 Next Steps

Congratulations! You've mastered advanced N8N workflows. Continue your journey:

1. **Practice**: Implement these patterns in real projects
2. **Experiment**: Combine patterns for complex solutions
3. **Optimize**: Focus on performance and reliability
4. **Learn**: Explore [Enterprise Patterns](./13-enterprise-patterns.md)

**Pro Tip**: Advanced workflows require careful planning. Always start with architecture design before implementation! 🏗️

---

**Ready for enterprise scale?** Continue with [Enterprise Patterns →](./13-enterprise-patterns.md)
