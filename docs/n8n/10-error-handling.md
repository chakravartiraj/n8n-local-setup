# Error Handling & Debugging in N8N

Master the art of building robust, fault-tolerant workflows with comprehensive error handling and debugging techniques.

## Table of Contents
- [Understanding Errors in N8N](#understanding-errors-in-n8n)
- [Error Types and Patterns](#error-types-and-patterns)
- [Built-in Error Handling](#built-in-error-handling)
- [Custom Error Handling](#custom-error-handling)
- [Debugging Techniques](#debugging-techniques)
- [Retry Strategies](#retry-strategies)
- [Monitoring and Alerting](#monitoring-and-alerting)
- [Production Error Handling](#production-error-handling)
- [Best Practices](#best-practices)

## Understanding Errors in N8N

### Error Flow in N8N
```
Node Execution → Error Occurs → Error Handling → Continue/Stop Workflow
```

### When Errors Occur
- Network timeouts
- Invalid API responses
- Missing required data
- Authentication failures
- Rate limit exceeded
- Invalid JSON parsing
- Database connection issues
- External service downtime

### Error Propagation
```javascript
// Errors can be:
// 1. Caught and handled within a node
// 2. Propagated to error handling nodes
// 3. Stop the entire workflow execution
```

## Error Types and Patterns

### Network Errors
```javascript
// Common network error patterns
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  return await response.json();
} catch (error) {
  if (error.code === 'ECONNREFUSED') {
    // Connection refused
    throw new Error('Service unavailable - connection refused');
  } else if (error.code === 'ETIMEDOUT') {
    // Timeout
    throw new Error('Request timeout - service not responding');
  } else if (error.code === 'ENOTFOUND') {
    // DNS resolution failed
    throw new Error('DNS resolution failed - check URL');
  }
  throw error;
}
```

### Data Validation Errors
```javascript
// Data validation error handling
function validateRequiredFields(data, requiredFields) {
  const missingFields = [];
  
  for (const field of requiredFields) {
    if (data[field] === undefined || data[field] === null || data[field] === '') {
      missingFields.push(field);
    }
  }
  
  if (missingFields.length > 0) {
    throw new Error(`Missing required fields: ${missingFields.join(', ')}`);
  }
  
  return true;
}

// Usage in workflow
try {
  validateRequiredFields($json, ['id', 'email', 'name']);
  // Continue processing
} catch (error) {
  return [{
    json: {
      error: true,
      message: error.message,
      data: $json,
      timestamp: new Date().toISOString()
    }
  }];
}
```

### Authentication Errors
```javascript
// Handle authentication errors
async function authenticatedRequest(url, options) {
  try {
    const response = await fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${$credentials.accessToken}`
      }
    });
    
    if (response.status === 401) {
      // Try to refresh token
      const newToken = await refreshAccessToken();
      
      // Retry with new token
      const retryResponse = await fetch(url, {
        ...options,
        headers: {
          ...options.headers,
          'Authorization': `Bearer ${newToken}`
        }
      });
      
      if (retryResponse.status === 401) {
        throw new Error('Authentication failed - credentials may be invalid');
      }
      
      return await retryResponse.json();
    }
    
    if (!response.ok) {
      throw new Error(`Request failed: ${response.status} ${response.statusText}`);
    }
    
    return await response.json();
    
  } catch (error) {
    if (error.message.includes('Authentication failed')) {
      // Handle auth failure
      throw new Error('AUTHENTICATION_FAILED');
    }
    throw error;
  }
}

async function refreshAccessToken() {
  // Token refresh logic
  const response = await fetch('https://api.example.com/oauth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      refresh_token: $credentials.refreshToken
    })
  });
  
  if (!response.ok) {
    throw new Error('Token refresh failed');
  }
  
  const data = await response.json();
  return data.access_token;
}
```

### Rate Limiting Errors
```javascript
// Handle rate limiting gracefully
async function rateLimitedRequest(url, options, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      
      if (response.status === 429) {
        const retryAfter = response.headers.get('retry-after');
        const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : Math.pow(2, attempt) * 1000;
        
        if (attempt === maxRetries) {
          throw new Error(`Rate limit exceeded. Max retries (${maxRetries}) reached`);
        }
        
        console.log(`Rate limited. Waiting ${waitTime}ms before retry ${attempt + 1}/${maxRetries}`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
        continue;
      }
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      
      return await response.json();
      
    } catch (error) {
      if (attempt === maxRetries || !error.message.includes('Rate limit')) {
        throw error;
      }
    }
  }
}
```

## Built-in Error Handling

### Continue on Fail
```javascript
// Node configuration to continue on failure
{
  "continueOnFail": true,
  "onError": "continueRegularOutput"
}

// In Function node
for (const item of $input.all()) {
  try {
    // Process item
    const result = processItem(item.json);
    item.json = result;
  } catch (error) {
    // Add error information to item
    item.json = {
      ...item.json,
      error: {
        message: error.message,
        timestamp: new Date().toISOString(),
        failed: true
      }
    };
  }
}

return $input.all();
```

### Error Trigger Node
```javascript
// Use Error Trigger to catch workflow errors
// Error Trigger node automatically activates when any node in the workflow fails

// In Error Trigger processing
const errorInfo = $json;

return [{
  json: {
    workflowId: errorInfo.workflow.id,
    workflowName: errorInfo.workflow.name,
    executionId: errorInfo.execution.id,
    errorNode: errorInfo.node.name,
    errorMessage: errorInfo.error.message,
    errorTimestamp: errorInfo.error.timestamp,
    inputData: errorInfo.inputData,
    severity: determineSeverity(errorInfo.error.message),
    notificationChannels: getNotificationChannels(errorInfo.error.message)
  }
}];

function determineSeverity(errorMessage) {
  if (errorMessage.includes('AUTHENTICATION_FAILED') || 
      errorMessage.includes('PERMISSION_DENIED')) {
    return 'HIGH';
  } else if (errorMessage.includes('TIMEOUT') || 
             errorMessage.includes('RATE_LIMIT')) {
    return 'MEDIUM';
  }
  return 'LOW';
}

function getNotificationChannels(errorMessage) {
  const channels = ['email'];
  
  if (errorMessage.includes('CRITICAL') || 
      errorMessage.includes('AUTHENTICATION_FAILED')) {
    channels.push('slack', 'sms');
  }
  
  return channels;
}
```

## Custom Error Handling

### Try-Catch Patterns
```javascript
// Comprehensive try-catch pattern
class ErrorHandler {
  static async safeExecute(operation, fallback = null, retries = 0) {
    let lastError = null;
    
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt < retries) {
          // Wait before retry with exponential backoff
          const delay = Math.pow(2, attempt) * 1000;
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        
        // All retries exhausted
        if (fallback !== null) {
          console.warn(`Operation failed after ${retries + 1} attempts. Using fallback.`, error);
          return typeof fallback === 'function' ? fallback(error) : fallback;
        }
        
        throw error;
      }
    }
  }
  
  static createErrorResponse(error, originalData = null) {
    return {
      success: false,
      error: {
        message: error.message,
        type: error.constructor.name,
        timestamp: new Date().toISOString(),
        stack: error.stack
      },
      originalData: originalData,
      retryable: this.isRetryableError(error)
    };
  }
  
  static isRetryableError(error) {
    const retryablePatterns = [
      'ECONNRESET',
      'ETIMEDOUT',
      'ENOTFOUND',
      'Rate limit',
      'timeout',
      '429',
      '500',
      '502',
      '503',
      '504'
    ];
    
    return retryablePatterns.some(pattern => 
      error.message.toLowerCase().includes(pattern.toLowerCase())
    );
  }
}

// Usage in workflow
const results = [];

for (const item of $input.all()) {
  const result = await ErrorHandler.safeExecute(
    async () => {
      // Your operation here
      const processed = await processItem(item.json);
      return {
        success: true,
        data: processed,
        processedAt: new Date().toISOString()
      };
    },
    (error) => ErrorHandler.createErrorResponse(error, item.json),
    2 // 2 retries
  );
  
  results.push({ json: result });
}

return results;
```

### Circuit Breaker Pattern
```javascript
// Circuit breaker for external service calls
class CircuitBreaker {
  constructor(failureThreshold = 5, timeoutDuration = 60000) {
    this.failureCount = 0;
    this.failureThreshold = failureThreshold;
    this.timeoutDuration = timeoutDuration;
    this.lastFailureTime = null;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
  }
  
  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeoutDuration) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN - service unavailable');
      }
    }
    
    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
  
  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}

// Usage
const apiCircuitBreaker = new CircuitBreaker(3, 30000); // 3 failures, 30s timeout

const processedItems = [];

for (const item of $input.all()) {
  try {
    const result = await apiCircuitBreaker.execute(async () => {
      return await callExternalAPI(item.json);
    });
    
    processedItems.push({
      json: {
        ...item.json,
        result: result,
        circuitBreakerState: apiCircuitBreaker.getState()
      }
    });
    
  } catch (error) {
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        circuitBreakerState: apiCircuitBreaker.getState(),
        failed: true
      }
    });
  }
}

return processedItems;

async function callExternalAPI(data) {
  const response = await fetch('https://api.example.com/process', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data)
  });
  
  if (!response.ok) {
    throw new Error(`API call failed: ${response.status}`);
  }
  
  return await response.json();
}
```

### Graceful Degradation
```javascript
// Graceful degradation pattern
class ServiceManager {
  constructor() {
    this.services = {
      primary: 'https://api-primary.example.com',
      secondary: 'https://api-secondary.example.com',
      cache: 'https://cache.example.com',
      fallback: 'local' // Local processing
    };
  }
  
  async getData(id) {
    // Try primary service
    try {
      return await this.callService(this.services.primary, id);
    } catch (error) {
      console.warn('Primary service failed:', error.message);
    }
    
    // Try secondary service
    try {
      return await this.callService(this.services.secondary, id);
    } catch (error) {
      console.warn('Secondary service failed:', error.message);
    }
    
    // Try cache
    try {
      return await this.callService(this.services.cache, id);
    } catch (error) {
      console.warn('Cache service failed:', error.message);
    }
    
    // Fallback to local processing
    return this.localFallback(id);
  }
  
  async callService(serviceUrl, id) {
    const response = await fetch(`${serviceUrl}/data/${id}`, {
      timeout: 5000 // 5 second timeout
    });
    
    if (!response.ok) {
      throw new Error(`Service call failed: ${response.status}`);
    }
    
    return await response.json();
  }
  
  localFallback(id) {
    // Local processing when all services fail
    return {
      id: id,
      data: null,
      source: 'local_fallback',
      message: 'All external services unavailable - using fallback',
      timestamp: new Date().toISOString()
    };
  }
}

// Usage
const serviceManager = new ServiceManager();
const processedItems = [];

for (const item of $input.all()) {
  try {
    const data = await serviceManager.getData(item.json.id);
    
    processedItems.push({
      json: {
        ...item.json,
        enrichedData: data,
        success: true
      }
    });
    
  } catch (error) {
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        success: false
      }
    });
  }
}

return processedItems;
```

## Debugging Techniques

### Logging and Tracing
```javascript
// Comprehensive logging utility
class Logger {
  constructor(context = 'Workflow') {
    this.context = context;
    this.logs = [];
  }
  
  debug(message, data = null) {
    this.log('DEBUG', message, data);
  }
  
  info(message, data = null) {
    this.log('INFO', message, data);
  }
  
  warn(message, data = null) {
    this.log('WARN', message, data);
  }
  
  error(message, error = null, data = null) {
    this.log('ERROR', message, { error: error?.message, stack: error?.stack, data });
  }
  
  log(level, message, data) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: level,
      context: this.context,
      message: message,
      data: data,
      nodePosition: $position || 'unknown'
    };
    
    this.logs.push(logEntry);
    console.log(`[${level}] ${this.context}: ${message}`, data || '');
  }
  
  getLogs() {
    return [...this.logs];
  }
  
  getLogsSince(timestamp) {
    return this.logs.filter(log => new Date(log.timestamp) >= new Date(timestamp));
  }
}

// Usage in workflow
const logger = new Logger('DataProcessor');

const processedItems = [];

for (const item of $input.all()) {
  logger.debug('Processing item', { id: item.json.id, position: $position });
  
  try {
    // Validation
    logger.debug('Validating item data');
    if (!item.json.id) {
      throw new Error('Missing required field: id');
    }
    
    // Processing
    logger.info('Starting data transformation');
    const processed = {
      ...item.json,
      processed: true,
      timestamp: new Date().toISOString()
    };
    
    logger.info('Item processed successfully', { id: item.json.id });
    
    processedItems.push({
      json: {
        ...processed,
        _logs: logger.getLogsSince(new Date(Date.now() - 60000)) // Last minute
      }
    });
    
  } catch (error) {
    logger.error('Failed to process item', error, { item: item.json });
    
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true,
        _logs: logger.getLogs()
      }
    });
  }
}

return processedItems;
```

### Data Inspection
```javascript
// Data inspection and validation utilities
class DataInspector {
  static inspect(data, path = '') {
    const inspection = {
      path: path || 'root',
      type: typeof data,
      value: data,
      size: this.getSize(data),
      properties: {},
      issues: []
    };
    
    // Type-specific inspection
    if (data === null) {
      inspection.issues.push('Value is null');
    } else if (data === undefined) {
      inspection.issues.push('Value is undefined');
    } else if (typeof data === 'string') {
      inspection.properties = {
        length: data.length,
        isEmpty: data.trim().length === 0,
        isEmail: /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data),
        isUrl: this.isValidUrl(data)
      };
    } else if (typeof data === 'number') {
      inspection.properties = {
        isInteger: Number.isInteger(data),
        isFinite: Number.isFinite(data),
        isNaN: Number.isNaN(data)
      };
    } else if (Array.isArray(data)) {
      inspection.properties = {
        length: data.length,
        isEmpty: data.length === 0,
        elementTypes: [...new Set(data.map(item => typeof item))]
      };
    } else if (typeof data === 'object') {
      inspection.properties = {
        keys: Object.keys(data),
        keyCount: Object.keys(data).length,
        hasNullValues: Object.values(data).includes(null),
        hasUndefinedValues: Object.values(data).includes(undefined)
      };
    }
    
    return inspection;
  }
  
  static deepInspect(data, maxDepth = 3, currentDepth = 0, path = '') {
    const result = {
      summary: this.inspect(data, path),
      children: {}
    };
    
    if (currentDepth < maxDepth && typeof data === 'object' && data !== null) {
      if (Array.isArray(data)) {
        // Inspect first few array elements
        for (let i = 0; i < Math.min(data.length, 5); i++) {
          result.children[`[${i}]`] = this.deepInspect(
            data[i], 
            maxDepth, 
            currentDepth + 1, 
            `${path}[${i}]`
          );
        }
      } else {
        // Inspect object properties
        for (const [key, value] of Object.entries(data)) {
          result.children[key] = this.deepInspect(
            value, 
            maxDepth, 
            currentDepth + 1, 
            path ? `${path}.${key}` : key
          );
        }
      }
    }
    
    return result;
  }
  
  static getSize(data) {
    if (data === null || data === undefined) return 0;
    if (typeof data === 'string') return data.length;
    if (Array.isArray(data)) return data.length;
    if (typeof data === 'object') return Object.keys(data).length;
    return 1;
  }
  
  static isValidUrl(string) {
    try {
      new URL(string);
      return true;
    } catch {
      return false;
    }
  }
  
  static findIssues(data) {
    const issues = [];
    const inspection = this.deepInspect(data);
    
    function collectIssues(node, path = '') {
      if (node.summary.issues.length > 0) {
        issues.push({
          path: path,
          issues: node.summary.issues
        });
      }
      
      for (const [key, child] of Object.entries(node.children)) {
        collectIssues(child, path ? `${path}.${key}` : key);
      }
    }
    
    collectIssues(inspection);
    return issues;
  }
}

// Usage for debugging
const debugResults = [];

for (const item of $input.all()) {
  const inspection = DataInspector.deepInspect(item.json);
  const issues = DataInspector.findIssues(item.json);
  
  debugResults.push({
    json: {
      originalData: item.json,
      inspection: inspection,
      issues: issues,
      position: $position,
      debuggedAt: new Date().toISOString()
    }
  });
}

return debugResults;
```

### Performance Monitoring
```javascript
// Performance monitoring for debugging
class PerformanceMonitor {
  constructor() {
    this.metrics = new Map();
    this.startTime = Date.now();
  }
  
  start(label) {
    this.metrics.set(label, {
      startTime: Date.now(),
      endTime: null,
      duration: null,
      memory: this.getMemoryUsage()
    });
  }
  
  end(label) {
    const metric = this.metrics.get(label);
    if (metric) {
      metric.endTime = Date.now();
      metric.duration = metric.endTime - metric.startTime;
      metric.memoryEnd = this.getMemoryUsage();
      metric.memoryDelta = metric.memoryEnd - metric.memory;
    }
  }
  
  getMemoryUsage() {
    // Simplified memory usage (not available in N8N context)
    return Date.now() % 10000; // Placeholder
  }
  
  getMetrics() {
    const metrics = {};
    for (const [label, data] of this.metrics.entries()) {
      metrics[label] = { ...data };
    }
    return metrics;
  }
  
  getSummary() {
    const totalDuration = Date.now() - this.startTime;
    const completedMetrics = Array.from(this.metrics.values())
      .filter(m => m.duration !== null);
    
    return {
      totalExecutionTime: totalDuration,
      operationsCount: this.metrics.size,
      completedOperations: completedMetrics.length,
      averageOperationTime: completedMetrics.length > 0 
        ? completedMetrics.reduce((sum, m) => sum + m.duration, 0) / completedMetrics.length 
        : 0,
      slowestOperation: completedMetrics.length > 0
        ? Math.max(...completedMetrics.map(m => m.duration))
        : 0
    };
  }
}

// Usage for performance debugging
const monitor = new PerformanceMonitor();
const results = [];

monitor.start('total_processing');

for (const [index, item] of $input.all().entries()) {
  monitor.start(`item_${index}`);
  
  try {
    monitor.start('validation');
    // Validation logic
    validateData(item.json);
    monitor.end('validation');
    
    monitor.start('transformation');
    // Transformation logic
    const transformed = transformData(item.json);
    monitor.end('transformation');
    
    monitor.start('external_api');
    // External API call
    const enriched = await enrichData(transformed);
    monitor.end('external_api');
    
    results.push({
      json: {
        ...enriched,
        _performance: {
          itemIndex: index,
          metrics: monitor.getMetrics()
        }
      }
    });
    
  } catch (error) {
    results.push({
      json: {
        ...item.json,
        error: error.message,
        _performance: {
          itemIndex: index,
          metrics: monitor.getMetrics(),
          failed: true
        }
      }
    });
  }
  
  monitor.end(`item_${index}`);
}

monitor.end('total_processing');

// Add performance summary to first result
if (results.length > 0) {
  results[0].json._performanceSummary = monitor.getSummary();
}

return results;

function validateData(data) {
  if (!data.id) throw new Error('Missing ID');
  return true;
}

function transformData(data) {
  return { ...data, transformed: true };
}

async function enrichData(data) {
  // Simulate async operation
  await new Promise(resolve => setTimeout(resolve, 100));
  return { ...data, enriched: true };
}
```

## Retry Strategies

### Exponential Backoff
```javascript
// Exponential backoff retry strategy
async function exponentialBackoffRetry(operation, maxRetries = 5, baseDelay = 1000) {
  let lastError;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      
      if (attempt === maxRetries - 1) {
        throw new Error(`Operation failed after ${maxRetries} attempts. Last error: ${error.message}`);
      }
      
      // Calculate delay with exponential backoff and jitter
      const delay = baseDelay * Math.pow(2, attempt);
      const jitter = Math.random() * 0.1 * delay; // Add 10% jitter
      const totalDelay = delay + jitter;
      
      console.log(`Attempt ${attempt + 1} failed. Retrying in ${Math.round(totalDelay)}ms...`);
      await new Promise(resolve => setTimeout(resolve, totalDelay));
    }
  }
}

// Usage
const processedItems = [];

for (const item of $input.all()) {
  try {
    const result = await exponentialBackoffRetry(
      async () => {
        // Your operation that might fail
        const response = await fetch('https://api.example.com/data', {
          method: 'POST',
          body: JSON.stringify(item.json)
        });
        
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        return await response.json();
      },
      3, // 3 retries
      1000 // 1 second base delay
    );
    
    processedItems.push({
      json: {
        ...item.json,
        result: result,
        success: true
      }
    });
    
  } catch (error) {
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        success: false,
        retries_exhausted: true
      }
    });
  }
}

return processedItems;
```

### Linear Backoff
```javascript
// Linear backoff retry strategy
async function linearBackoffRetry(operation, maxRetries = 3, delay = 2000) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries - 1) {
        throw error;
      }
      
      console.log(`Attempt ${attempt + 1} failed. Retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

### Conditional Retry
```javascript
// Conditional retry based on error type
async function conditionalRetry(operation, shouldRetry, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries - 1 || !shouldRetry(error)) {
        throw error;
      }
      
      const delay = Math.pow(2, attempt) * 1000;
      console.log(`Retryable error on attempt ${attempt + 1}. Retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// Define retry conditions
function isRetryableError(error) {
  const retryableErrors = [
    'ECONNRESET',
    'ETIMEDOUT',
    'ENOTFOUND',
    'Rate limit exceeded',
    'Service temporarily unavailable'
  ];
  
  const retryableStatusCodes = [429, 500, 502, 503, 504];
  
  // Check error message
  const messageRetryable = retryableErrors.some(msg => 
    error.message.toLowerCase().includes(msg.toLowerCase())
  );
  
  // Check HTTP status codes
  const statusRetryable = retryableStatusCodes.some(code => 
    error.message.includes(code.toString())
  );
  
  return messageRetryable || statusRetryable;
}

// Usage
for (const item of $input.all()) {
  try {
    const result = await conditionalRetry(
      async () => {
        return await callExternalAPI(item.json);
      },
      isRetryableError,
      3
    );
    
    // Process successful result
  } catch (error) {
    // Handle final failure
  }
}
```

## Monitoring and Alerting

### Error Metrics Collection
```javascript
// Error metrics collection
class ErrorMetrics {
  constructor() {
    this.errors = [];
    this.startTime = Date.now();
  }
  
  recordError(error, context = {}) {
    this.errors.push({
      timestamp: new Date().toISOString(),
      message: error.message,
      type: error.constructor.name,
      context: context,
      stack: error.stack
    });
  }
  
  getMetrics() {
    const now = Date.now();
    const duration = now - this.startTime;
    
    const errorTypes = this.errors.reduce((acc, error) => {
      acc[error.type] = (acc[error.type] || 0) + 1;
      return acc;
    }, {});
    
    const recentErrors = this.errors.filter(error => 
      new Date(error.timestamp).getTime() > now - 300000 // Last 5 minutes
    );
    
    return {
      totalErrors: this.errors.length,
      errorRate: this.errors.length / (duration / 1000), // errors per second
      errorTypes: errorTypes,
      recentErrors: recentErrors.length,
      firstError: this.errors.length > 0 ? this.errors[0].timestamp : null,
      lastError: this.errors.length > 0 ? this.errors[this.errors.length - 1].timestamp : null
    };
  }
  
  shouldAlert() {
    const metrics = this.getMetrics();
    
    // Alert conditions
    if (metrics.totalErrors > 10) return { reason: 'High error count', severity: 'HIGH' };
    if (metrics.errorRate > 0.1) return { reason: 'High error rate', severity: 'MEDIUM' };
    if (metrics.recentErrors > 5) return { reason: 'Recent error spike', severity: 'MEDIUM' };
    
    return null;
  }
}

// Usage in workflow
const errorMetrics = new ErrorMetrics();
const results = [];

for (const item of $input.all()) {
  try {
    const processed = await processItem(item.json);
    results.push({ json: processed });
  } catch (error) {
    errorMetrics.recordError(error, { 
      itemId: item.json.id, 
      position: $position 
    });
    
    results.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true
      }
    });
  }
}

// Check if alert should be sent
const alertCondition = errorMetrics.shouldAlert();
if (alertCondition) {
  results.push({
    json: {
      alert: true,
      severity: alertCondition.severity,
      reason: alertCondition.reason,
      metrics: errorMetrics.getMetrics(),
      timestamp: new Date().toISOString()
    }
  });
}

return results;

async function processItem(data) {
  // Your processing logic
  return { ...data, processed: true };
}
```

### Health Check Implementation
```javascript
// Health check system
class HealthChecker {
  constructor() {
    this.checks = new Map();
  }
  
  addCheck(name, checkFunction, timeout = 5000) {
    this.checks.set(name, { checkFunction, timeout });
  }
  
  async runAllChecks() {
    const results = {};
    
    for (const [name, { checkFunction, timeout }] of this.checks.entries()) {
      try {
        const result = await Promise.race([
          checkFunction(),
          new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Health check timeout')), timeout)
          )
        ]);
        
        results[name] = {
          status: 'healthy',
          response: result,
          timestamp: new Date().toISOString()
        };
      } catch (error) {
        results[name] = {
          status: 'unhealthy',
          error: error.message,
          timestamp: new Date().toISOString()
        };
      }
    }
    
    return results;
  }
  
  getOverallHealth(results) {
    const unhealthyServices = Object.values(results)
      .filter(result => result.status === 'unhealthy');
    
    if (unhealthyServices.length === 0) {
      return { status: 'healthy', message: 'All services operational' };
    } else if (unhealthyServices.length < Object.keys(results).length) {
      return { status: 'degraded', message: 'Some services unavailable' };
    } else {
      return { status: 'unhealthy', message: 'Multiple services down' };
    }
  }
}

// Setup health checks
const healthChecker = new HealthChecker();

healthChecker.addCheck('database', async () => {
  // Check database connectivity
  const response = await fetch('https://api.example.com/health/db');
  if (!response.ok) throw new Error('Database health check failed');
  return await response.json();
});

healthChecker.addCheck('external_api', async () => {
  // Check external API
  const response = await fetch('https://external-api.example.com/ping');
  if (!response.ok) throw new Error('External API unreachable');
  return { ping: 'pong' };
});

healthChecker.addCheck('cache', async () => {
  // Check cache service
  return { status: 'ok', memory_usage: '45%' };
});

// Run health checks
const healthResults = await healthChecker.runAllChecks();
const overallHealth = healthChecker.getOverallHealth(healthResults);

return [{
  json: {
    overall: overallHealth,
    services: healthResults,
    timestamp: new Date().toISOString()
  }
}];
```

## Production Error Handling

### Error Recovery Strategies
```javascript
// Production-grade error recovery
class ProductionErrorHandler {
  constructor(config = {}) {
    this.config = {
      maxRetries: config.maxRetries || 3,
      circuitBreakerThreshold: config.circuitBreakerThreshold || 5,
      alertThreshold: config.alertThreshold || 10,
      ...config
    };
    
    this.errorCounts = new Map();
    this.circuitBreakers = new Map();
  }
  
  async executeWithRecovery(operation, context = {}) {
    const operationKey = context.operationKey || 'default';
    
    // Check circuit breaker
    if (this.isCircuitOpen(operationKey)) {
      throw new Error(`Circuit breaker open for operation: ${operationKey}`);
    }
    
    try {
      const result = await this.retryOperation(operation, operationKey);
      this.recordSuccess(operationKey);
      return result;
    } catch (error) {
      this.recordFailure(operationKey, error);
      
      // Try fallback if available
      if (context.fallback) {
        try {
          return await context.fallback(error);
        } catch (fallbackError) {
          throw new Error(`Primary and fallback operations failed. Primary: ${error.message}, Fallback: ${fallbackError.message}`);
        }
      }
      
      throw error;
    }
  }
  
  async retryOperation(operation, operationKey) {
    let lastError;
    
    for (let attempt = 0; attempt < this.config.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt < this.config.maxRetries - 1 && this.isRetryableError(error)) {
          const delay = this.calculateBackoff(attempt);
          await this.sleep(delay);
          continue;
        }
        
        break;
      }
    }
    
    throw lastError;
  }
  
  recordFailure(operationKey, error) {
    if (!this.errorCounts.has(operationKey)) {
      this.errorCounts.set(operationKey, { count: 0, lastError: null });
    }
    
    const errorData = this.errorCounts.get(operationKey);
    errorData.count++;
    errorData.lastError = error;
    
    // Open circuit breaker if threshold reached
    if (errorData.count >= this.config.circuitBreakerThreshold) {
      this.circuitBreakers.set(operationKey, {
        openedAt: Date.now(),
        timeout: 60000 // 1 minute
      });
    }
  }
  
  recordSuccess(operationKey) {
    this.errorCounts.delete(operationKey);
    this.circuitBreakers.delete(operationKey);
  }
  
  isCircuitOpen(operationKey) {
    const circuit = this.circuitBreakers.get(operationKey);
    if (!circuit) return false;
    
    // Check if circuit should be closed
    if (Date.now() - circuit.openedAt > circuit.timeout) {
      this.circuitBreakers.delete(operationKey);
      return false;
    }
    
    return true;
  }
  
  isRetryableError(error) {
    const retryablePatterns = [
      'timeout', 'ETIMEDOUT', 'ECONNRESET', 'ENOTFOUND',
      'Rate limit', '429', '500', '502', '503', '504'
    ];
    
    return retryablePatterns.some(pattern => 
      error.message.toLowerCase().includes(pattern.toLowerCase())
    );
  }
  
  calculateBackoff(attempt) {
    return Math.min(1000 * Math.pow(2, attempt), 30000); // Max 30 seconds
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  getHealthStatus() {
    return {
      openCircuits: Array.from(this.circuitBreakers.keys()),
      errorCounts: Object.fromEntries(this.errorCounts),
      timestamp: new Date().toISOString()
    };
  }
}

// Usage in production workflow
const errorHandler = new ProductionErrorHandler({
  maxRetries: 3,
  circuitBreakerThreshold: 5,
  alertThreshold: 10
});

const results = [];

for (const item of $input.all()) {
  try {
    const result = await errorHandler.executeWithRecovery(
      async () => {
        // Primary operation
        return await processItem(item.json);
      },
      {
        operationKey: 'process_item',
        fallback: async (error) => {
          // Fallback operation
          return {
            ...item.json,
            processed: false,
            fallback: true,
            error: error.message
          };
        }
      }
    );
    
    results.push({ json: result });
    
  } catch (error) {
    results.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true,
        timestamp: new Date().toISOString()
      }
    });
  }
}

// Add health status
results.push({
  json: {
    healthStatus: errorHandler.getHealthStatus(),
    type: 'health_report'
  }
});

return results;

async function processItem(data) {
  // Simulate processing that might fail
  if (Math.random() < 0.2) { // 20% failure rate
    throw new Error('Random processing failure');
  }
  
  return {
    ...data,
    processed: true,
    timestamp: new Date().toISOString()
  };
}
```

## Best Practices

### Error Handling Checklist
```javascript
// Production error handling checklist
const ErrorHandlingBestPractices = {
  // 1. Always validate input data
  validateInput: (data, schema) => {
    const errors = [];
    
    for (const [field, rules] of Object.entries(schema)) {
      if (rules.required && !data[field]) {
        errors.push(`${field} is required`);
      }
      
      if (data[field] && rules.type && typeof data[field] !== rules.type) {
        errors.push(`${field} must be of type ${rules.type}`);
      }
    }
    
    if (errors.length > 0) {
      throw new Error(`Validation failed: ${errors.join(', ')}`);
    }
  },
  
  // 2. Use specific error types
  createSpecificError: (type, message, details = {}) => {
    const error = new Error(message);
    error.type = type;
    error.details = details;
    error.timestamp = new Date().toISOString();
    return error;
  },
  
  // 3. Log errors with context
  logError: (error, context = {}) => {
    console.error('Error occurred:', {
      message: error.message,
      type: error.type || error.constructor.name,
      stack: error.stack,
      context: context,
      timestamp: new Date().toISOString()
    });
  },
  
  // 4. Fail fast for unrecoverable errors
  shouldFailFast: (error) => {
    const unrecoverableErrors = [
      'AUTHENTICATION_FAILED',
      'PERMISSION_DENIED',
      'INVALID_CONFIGURATION',
      'SYNTAX_ERROR'
    ];
    
    return unrecoverableErrors.some(type => 
      error.message.includes(type) || error.type === type
    );
  },
  
  // 5. Provide meaningful error messages
  createUserFriendlyError: (error, context = {}) => {
    const errorMap = {
      'ENOTFOUND': 'Service is currently unavailable. Please try again later.',
      'ETIMEDOUT': 'Request timed out. Please check your connection and try again.',
      'AUTHENTICATION_FAILED': 'Authentication failed. Please check your credentials.',
      'RATE_LIMIT_EXCEEDED': 'Too many requests. Please wait before trying again.',
      'VALIDATION_ERROR': 'The provided data is invalid. Please check and try again.'
    };
    
    const userMessage = errorMap[error.type] || errorMap[error.message] || 
                       'An unexpected error occurred. Please try again or contact support.';
    
    return {
      userMessage: userMessage,
      technicalDetails: {
        error: error.message,
        type: error.type,
        context: context,
        timestamp: new Date().toISOString()
      }
    };
  }
};

// Example usage
const processedItems = [];

for (const item of $input.all()) {
  try {
    // 1. Validate input
    ErrorHandlingBestPractices.validateInput(item.json, {
      id: { required: true, type: 'string' },
      email: { required: true, type: 'string' },
      amount: { required: true, type: 'number' }
    });
    
    // 2. Process with specific error handling
    const result = await processWithBestPractices(item.json);
    processedItems.push({ json: result });
    
  } catch (error) {
    // 3. Log error with context
    ErrorHandlingBestPractices.logError(error, { 
      itemId: item.json.id, 
      workflow: 'data-processing',
      node: 'item-processor'
    });
    
    // 4. Check if should fail fast
    if (ErrorHandlingBestPractices.shouldFailFast(error)) {
      throw error; // Stop entire workflow
    }
    
    // 5. Create user-friendly error
    const userError = ErrorHandlingBestPractices.createUserFriendlyError(error, {
      itemId: item.json.id
    });
    
    processedItems.push({
      json: {
        ...item.json,
        error: userError,
        failed: true
      }
    });
  }
}

return processedItems;

async function processWithBestPractices(data) {
  try {
    // Simulate processing
    const result = await externalAPICall(data);
    return {
      ...data,
      result: result,
      processed: true
    };
  } catch (error) {
    // Transform generic errors to specific ones
    if (error.message.includes('401')) {
      throw ErrorHandlingBestPractices.createSpecificError(
        'AUTHENTICATION_FAILED', 
        'API authentication failed',
        { statusCode: 401 }
      );
    } else if (error.message.includes('429')) {
      throw ErrorHandlingBestPractices.createSpecificError(
        'RATE_LIMIT_EXCEEDED',
        'API rate limit exceeded',
        { statusCode: 429 }
      );
    }
    
    throw error;
  }
}

async function externalAPICall(data) {
  // Simulate API call
  const random = Math.random();
  if (random < 0.1) throw new Error('401 Unauthorized');
  if (random < 0.2) throw new Error('429 Too Many Requests');
  if (random < 0.3) throw new Error('ETIMEDOUT');
  
  return { processed: true, timestamp: Date.now() };
}
```

## Practice Exercises

### Exercise 1: Build a Resilient API Client
Create a robust API client that handles:
1. Network timeouts and retries
2. Authentication token refresh
3. Rate limiting
4. Circuit breaker pattern
5. Graceful degradation

### Exercise 2: Error Monitoring System
Build an error monitoring workflow that:
1. Collects errors from multiple workflows
2. Categorizes and analyzes error patterns
3. Sends alerts based on severity
4. Generates error reports
5. Tracks resolution status

### Exercise 3: Data Pipeline with Error Recovery
Design a data processing pipeline that:
1. Validates incoming data
2. Handles partial failures
3. Implements retry logic
4. Provides data quality reports
5. Supports manual error resolution

## Next Steps

- Master [Performance Optimization](./11-performance-optimization.md)
- Learn [Security Best Practices](./12-security-practices.md)
- Explore [Enterprise Patterns](./13-enterprise-patterns.md)

---

Robust error handling is the foundation of reliable automation systems. Master these patterns to build workflows that gracefully handle any situation!
