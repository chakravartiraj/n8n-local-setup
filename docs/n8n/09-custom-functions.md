# Custom Functions & Expressions in N8N

Master advanced N8N programming with custom functions, expressions, and JavaScript code.

## Table of Contents
- [Expression Basics](#expression-basics)
- [JavaScript in N8N](#javascript-in-n8n)
- [Function Node Mastery](#function-node-mastery)
- [Code Node Advanced](#code-node-advanced)
- [Custom Utility Functions](#custom-utility-functions)
- [Data Processing Patterns](#data-processing-patterns)
- [Performance Optimization](#performance-optimization)
- [Real-world Examples](#real-world-examples)
- [Best Practices](#best-practices)

## Expression Basics

### N8N Expression Syntax
```javascript
// Basic syntax
{{$json.fieldName}}

// Nested objects
{{$json.user.profile.email}}

// Array access
{{$json.items[0].name}}

// With default values
{{$json.optional_field || "default_value"}}

// Multiple operations
{{$json.first_name + " " + $json.last_name}}
```

### Available Variables
```javascript
// Current item data
$json           // Current item's JSON data
$binary         // Current item's binary data
$position       // Current item's position in array
$runIndex       // Current execution run index

// Previous node data
$('Node Name').item.json.field    // Specific item
$('Node Name').all()              // All items
$('Node Name').first()            // First item
$('Node Name').last()             // Last item

// Workflow context
$workflow       // Workflow metadata
$execution      // Execution information
$vars           // Workflow variables
$credentials    // Stored credentials

// Environment
$env            // Environment variables
$now            // Current timestamp
$today          // Today's date
```

### Expression Functions
```javascript
// Date/Time functions
{{DateTime.now()}}                           // Current datetime
{{DateTime.format($json.date, 'yyyy-MM-dd')}} // Format date
{{DateTime.plus(DateTime.now(), {days: 7})}}   // Add time

// String functions
{{$json.text.toUpperCase()}}                 // Uppercase
{{$json.text.toLowerCase()}}                 // Lowercase
{{$json.text.trim()}}                        // Remove whitespace
{{$json.text.split(',')}}                    // Split string
{{$json.text.replace('old', 'new')}}         // Replace text

// Math functions
{{Math.round($json.price * 1.1)}}           // Round number
{{Math.max($json.values)}}                  // Maximum value
{{Math.random()}}                           // Random number

// Array functions
{{$json.items.length}}                      // Array length
{{$json.items.join(', ')}}                  // Join array
{{$json.items.includes('value')}}           // Check if includes

// Object functions
{{Object.keys($json)}}                      // Object keys
{{Object.values($json)}}                    // Object values
```

## JavaScript in N8N

### Basic JavaScript Patterns
```javascript
// Conditional logic
const status = $json.score > 80 ? 'excellent' : 
               $json.score > 60 ? 'good' : 'needs_improvement';

// Array processing
const processedItems = $json.items.map(item => ({
  id: item.id,
  name: item.name.toUpperCase(),
  category: item.category || 'uncategorized'
}));

// Filtering
const activeUsers = $json.users.filter(user => user.status === 'active');

// Reducing
const totalAmount = $json.orders.reduce((sum, order) => sum + order.amount, 0);

// Complex transformations
const userSummary = $json.users.reduce((acc, user) => {
  const dept = user.department || 'Unknown';
  if (!acc[dept]) {
    acc[dept] = { count: 0, totalSalary: 0 };
  }
  acc[dept].count++;
  acc[dept].totalSalary += user.salary || 0;
  return acc;
}, {});
```

### Working with Dates
```javascript
// Date manipulation
const now = new Date();
const tomorrow = new Date(now.getTime() + 24 * 60 * 60 * 1000);
const nextWeek = new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000);

// Date formatting
const formatDate = (date) => {
  return date.toISOString().split('T')[0]; // YYYY-MM-DD
};

// Business days calculation
function addBusinessDays(date, days) {
  const result = new Date(date);
  while (days > 0) {
    result.setDate(result.getDate() + 1);
    if (result.getDay() !== 0 && result.getDay() !== 6) { // Not weekend
      days--;
    }
  }
  return result;
}

// Age calculation
function calculateAge(birthDate) {
  const today = new Date();
  const birth = new Date(birthDate);
  let age = today.getFullYear() - birth.getFullYear();
  const monthDiff = today.getMonth() - birth.getMonth();
  
  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birth.getDate())) {
    age--;
  }
  
  return age;
}
```

### String Processing
```javascript
// Advanced string operations
function slugify(text) {
  return text
    .toLowerCase()
    .replace(/[^\w\s-]/g, '') // Remove special characters
    .replace(/[\s_-]+/g, '-') // Replace spaces and underscores with hyphens
    .replace(/^-+|-+$/g, ''); // Remove leading/trailing hyphens
}

function extractEmails(text) {
  const emailRegex = /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g;
  return text.match(emailRegex) || [];
}

function formatPhoneNumber(phone) {
  // Remove all non-digit characters
  const cleaned = phone.replace(/\D/g, '');
  
  // Format as (XXX) XXX-XXXX
  if (cleaned.length === 10) {
    return `(${cleaned.slice(0, 3)}) ${cleaned.slice(3, 6)}-${cleaned.slice(6)}`;
  }
  
  return phone; // Return original if not 10 digits
}

function capitalizeWords(str) {
  return str.replace(/\w\S*/g, (txt) => 
    txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase()
  );
}
```

## Function Node Mastery

### Basic Function Node Structure
```javascript
// Function node template
for (const item of $input.all()) {
  // Process each item
  const data = item.json;
  
  // Your logic here
  const processedData = {
    ...data,
    processed: true,
    timestamp: new Date().toISOString()
  };
  
  // Modify the item
  item.json = processedData;
}

return $input.all();
```

### Advanced Data Transformations
```javascript
// Complex data transformation
const transformedItems = [];

for (const item of $input.all()) {
  const data = item.json;
  
  // Nested object flattening
  function flattenObject(obj, prefix = '') {
    const flattened = {};
    
    for (const [key, value] of Object.entries(obj)) {
      const newKey = prefix ? `${prefix}.${key}` : key;
      
      if (value !== null && typeof value === 'object' && !Array.isArray(value)) {
        Object.assign(flattened, flattenObject(value, newKey));
      } else {
        flattened[newKey] = value;
      }
    }
    
    return flattened;
  }
  
  // Apply transformation
  const flattened = flattenObject(data);
  
  // Add computed fields
  flattened.fullName = `${data.firstName || ''} ${data.lastName || ''}`.trim();
  flattened.ageGroup = data.age < 18 ? 'minor' : 
                      data.age < 65 ? 'adult' : 'senior';
  flattened.isVip = data.totalSpent > 10000;
  
  transformedItems.push({
    json: flattened,
    binary: item.binary
  });
}

return transformedItems;
```

### Conditional Processing
```javascript
// Dynamic workflow branching
const results = {
  highPriority: [],
  normalPriority: [],
  errors: []
};

for (const item of $input.all()) {
  try {
    const data = item.json;
    
    // Validation
    if (!data.id || !data.type) {
      results.errors.push({
        json: {
          error: 'Missing required fields',
          originalData: data,
          timestamp: new Date().toISOString()
        }
      });
      continue;
    }
    
    // Priority determination
    const isHighPriority = 
      data.type === 'urgent' ||
      data.amount > 1000 ||
      data.customerTier === 'platinum';
    
    const processedItem = {
      json: {
        ...data,
        priority: isHighPriority ? 'high' : 'normal',
        processedAt: new Date().toISOString(),
        estimatedProcessingTime: isHighPriority ? '15 minutes' : '2 hours'
      }
    };
    
    if (isHighPriority) {
      results.highPriority.push(processedItem);
    } else {
      results.normalPriority.push(processedItem);
    }
    
  } catch (error) {
    results.errors.push({
      json: {
        error: error.message,
        originalData: item.json,
        timestamp: new Date().toISOString()
      }
    });
  }
}

// Return all results (you can use different outputs in the node)
return [
  ...results.highPriority,
  ...results.normalPriority,
  ...results.errors
];
```

## Code Node Advanced

### Async Operations
```javascript
// Async/await in Code node
const processedItems = [];

for (const item of $input.all()) {
  try {
    // Simulated async operation
    const result = await new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          processed: true,
          timestamp: Date.now(),
          randomValue: Math.random()
        });
      }, 100);
    });
    
    processedItems.push({
      json: {
        ...item.json,
        asyncResult: result
      }
    });
    
  } catch (error) {
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true
      }
    });
  }
}

return processedItems;
```

### External API Calls
```javascript
// Making HTTP requests in Code node
const results = [];

for (const item of $input.all()) {
  try {
    const userId = item.json.userId;
    
    // Fetch additional user data
    const userResponse = await fetch(`https://api.example.com/users/${userId}`, {
      headers: {
        'Authorization': `Bearer ${$credentials.apiToken}`,
        'Content-Type': 'application/json'
      }
    });
    
    if (!userResponse.ok) {
      throw new Error(`API request failed: ${userResponse.status}`);
    }
    
    const userData = await userResponse.json();
    
    // Enrich original data
    results.push({
      json: {
        ...item.json,
        userDetails: {
          name: userData.name,
          email: userData.email,
          joinDate: userData.created_at,
          isActive: userData.status === 'active'
        },
        enrichedAt: new Date().toISOString()
      }
    });
    
  } catch (error) {
    results.push({
      json: {
        ...item.json,
        enrichmentError: error.message,
        enrichedAt: new Date().toISOString()
      }
    });
  }
}

return results;
```

### Database Operations (if available)
```javascript
// Database operations in Code node
const mysql = require('mysql2/promise');

// Create connection
const connection = await mysql.createConnection({
  host: $credentials.dbHost,
  user: $credentials.dbUser,
  password: $credentials.dbPassword,
  database: $credentials.dbName
});

const results = [];

try {
  for (const item of $input.all()) {
    const data = item.json;
    
    // Insert/Update operations
    const query = `
      INSERT INTO user_events (user_id, event_type, event_data, created_at)
      VALUES (?, ?, ?, NOW())
      ON DUPLICATE KEY UPDATE
      event_data = VALUES(event_data),
      updated_at = NOW()
    `;
    
    const [result] = await connection.execute(query, [
      data.userId,
      data.eventType,
      JSON.stringify(data.eventData)
    ]);
    
    results.push({
      json: {
        ...data,
        dbResult: {
          insertId: result.insertId,
          affectedRows: result.affectedRows
        }
      }
    });
  }
  
} finally {
  await connection.end();
}

return results;
```

## Custom Utility Functions

### Data Validation Library
```javascript
// Comprehensive validation utilities
const validators = {
  email: (email) => {
    const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return regex.test(email);
  },
  
  phone: (phone) => {
    const cleaned = phone.replace(/\D/g, '');
    return cleaned.length >= 10 && cleaned.length <= 15;
  },
  
  url: (url) => {
    try {
      new URL(url);
      return true;
    } catch {
      return false;
    }
  },
  
  creditCard: (number) => {
    // Luhn algorithm
    const cleaned = number.replace(/\D/g, '');
    let sum = 0;
    let isEven = false;
    
    for (let i = cleaned.length - 1; i >= 0; i--) {
      let digit = parseInt(cleaned[i]);
      
      if (isEven) {
        digit *= 2;
        if (digit > 9) digit -= 9;
      }
      
      sum += digit;
      isEven = !isEven;
    }
    
    return sum % 10 === 0 && cleaned.length >= 13;
  },
  
  required: (value) => {
    return value !== null && value !== undefined && value !== '';
  },
  
  minLength: (value, min) => {
    return value && value.length >= min;
  },
  
  maxLength: (value, max) => {
    return !value || value.length <= max;
  },
  
  numeric: (value) => {
    return !isNaN(value) && !isNaN(parseFloat(value));
  },
  
  dateFormat: (value, format = 'YYYY-MM-DD') => {
    const regex = /^\d{4}-\d{2}-\d{2}$/;
    return regex.test(value);
  }
};

function validateObject(obj, schema) {
  const errors = {};
  
  for (const [field, rules] of Object.entries(schema)) {
    const value = obj[field];
    const fieldErrors = [];
    
    for (const rule of rules) {
      if (typeof rule === 'string') {
        if (!validators[rule](value)) {
          fieldErrors.push(`Failed ${rule} validation`);
        }
      } else if (typeof rule === 'object') {
        const { type, params = [] } = rule;
        if (!validators[type](value, ...params)) {
          fieldErrors.push(`Failed ${type} validation with params: ${params.join(', ')}`);
        }
      }
    }
    
    if (fieldErrors.length > 0) {
      errors[field] = fieldErrors;
    }
  }
  
  return {
    isValid: Object.keys(errors).length === 0,
    errors
  };
}

// Usage example
const schema = {
  email: ['required', 'email'],
  phone: ['required', 'phone'],
  name: ['required', { type: 'minLength', params: [2] }],
  age: ['required', 'numeric']
};

const validatedItems = $input.all().map(item => {
  const validation = validateObject(item.json, schema);
  
  return {
    json: {
      ...item.json,
      validation: validation
    }
  };
});

return validatedItems;
```

### Data Transformation Utilities
```javascript
// Advanced transformation utilities
const transformUtils = {
  // Deep merge objects
  deepMerge: (target, source) => {
    const result = { ...target };
    
    for (const [key, value] of Object.entries(source)) {
      if (value && typeof value === 'object' && !Array.isArray(value)) {
        result[key] = transformUtils.deepMerge(result[key] || {}, value);
      } else {
        result[key] = value;
      }
    }
    
    return result;
  },
  
  // Pick specific fields
  pick: (obj, fields) => {
    const result = {};
    for (const field of fields) {
      if (field in obj) {
        result[field] = obj[field];
      }
    }
    return result;
  },
  
  // Omit specific fields
  omit: (obj, fields) => {
    const result = { ...obj };
    for (const field of fields) {
      delete result[field];
    }
    return result;
  },
  
  // Rename fields
  rename: (obj, mapping) => {
    const result = {};
    for (const [oldKey, newKey] of Object.entries(mapping)) {
      if (oldKey in obj) {
        result[newKey] = obj[oldKey];
      }
    }
    // Add non-renamed fields
    for (const [key, value] of Object.entries(obj)) {
      if (!(key in mapping)) {
        result[key] = value;
      }
    }
    return result;
  },
  
  // Group by field
  groupBy: (array, field) => {
    return array.reduce((groups, item) => {
      const key = item[field];
      if (!groups[key]) {
        groups[key] = [];
      }
      groups[key].push(item);
      return groups;
    }, {});
  },
  
  // Calculate statistics
  calculateStats: (numbers) => {
    const sorted = [...numbers].sort((a, b) => a - b);
    const sum = numbers.reduce((a, b) => a + b, 0);
    const mean = sum / numbers.length;
    
    return {
      count: numbers.length,
      sum: sum,
      mean: mean,
      median: sorted[Math.floor(sorted.length / 2)],
      min: Math.min(...numbers),
      max: Math.max(...numbers),
      range: Math.max(...numbers) - Math.min(...numbers)
    };
  },
  
  // Normalize text
  normalizeText: (text) => {
    return text
      .toLowerCase()
      .trim()
      .replace(/\s+/g, ' ') // Multiple spaces to single space
      .replace(/[^\w\s-]/g, ''); // Remove special characters except hyphens
  }
};

// Usage in transformation
const transformedData = $input.all().map(item => {
  const data = item.json;
  
  // Apply various transformations
  const picked = transformUtils.pick(data, ['id', 'name', 'email', 'orders']);
  const renamed = transformUtils.rename(picked, { 'name': 'fullName' });
  
  // Add computed fields
  if (data.orders && Array.isArray(data.orders)) {
    const orderAmounts = data.orders.map(order => order.amount);
    renamed.orderStats = transformUtils.calculateStats(orderAmounts);
  }
  
  return { json: renamed };
});

return transformedData;
```

### Business Logic Patterns
```javascript
// Business rule engine
class BusinessRuleEngine {
  constructor() {
    this.rules = [];
  }
  
  addRule(name, condition, action) {
    this.rules.push({ name, condition, action });
    return this;
  }
  
  evaluate(data) {
    const results = {
      data: data,
      appliedRules: [],
      actions: []
    };
    
    for (const rule of this.rules) {
      try {
        if (rule.condition(data)) {
          const actionResult = rule.action(data);
          results.appliedRules.push(rule.name);
          results.actions.push(actionResult);
          
          // Apply action result to data if it returns modifications
          if (actionResult && actionResult.modifications) {
            Object.assign(results.data, actionResult.modifications);
          }
        }
      } catch (error) {
        console.error(`Error in rule ${rule.name}:`, error);
      }
    }
    
    return results;
  }
}

// Define business rules
const engine = new BusinessRuleEngine()
  .addRule('VIP Customer', 
    (data) => data.totalSpent > 10000,
    (data) => ({
      type: 'classification',
      classification: 'vip',
      benefits: ['free-shipping', 'priority-support'],
      modifications: { customerTier: 'vip' }
    })
  )
  .addRule('Bulk Discount',
    (data) => data.orderQuantity > 100,
    (data) => ({
      type: 'discount',
      discountPercent: 15,
      modifications: { 
        discount: data.orderTotal * 0.15,
        finalTotal: data.orderTotal * 0.85
      }
    })
  )
  .addRule('New Customer Welcome',
    (data) => data.isFirstOrder,
    (data) => ({
      type: 'marketing',
      action: 'send_welcome_series',
      modifications: { welcomeSeriesScheduled: true }
    })
  );

// Apply rules to data
const processedItems = $input.all().map(item => {
  const result = engine.evaluate(item.json);
  
  return {
    json: {
      ...result.data,
      businessRules: {
        appliedRules: result.appliedRules,
        actions: result.actions,
        processedAt: new Date().toISOString()
      }
    }
  };
});

return processedItems;
```

## Data Processing Patterns

### Stream Processing Pattern
```javascript
// Process large datasets efficiently
class DataStream {
  constructor(data) {
    this.data = data;
    this.transformations = [];
  }
  
  filter(predicate) {
    this.transformations.push({ type: 'filter', fn: predicate });
    return this;
  }
  
  map(transformer) {
    this.transformations.push({ type: 'map', fn: transformer });
    return this;
  }
  
  reduce(reducer, initial) {
    this.transformations.push({ type: 'reduce', fn: reducer, initial });
    return this;
  }
  
  groupBy(keySelector) {
    this.transformations.push({ type: 'groupBy', fn: keySelector });
    return this;
  }
  
  execute() {
    let result = this.data;
    
    for (const transformation of this.transformations) {
      switch (transformation.type) {
        case 'filter':
          result = result.filter(transformation.fn);
          break;
        case 'map':
          result = result.map(transformation.fn);
          break;
        case 'reduce':
          result = result.reduce(transformation.fn, transformation.initial);
          break;
        case 'groupBy':
          result = this.groupByKey(result, transformation.fn);
          break;
      }
    }
    
    return result;
  }
  
  groupByKey(array, keySelector) {
    return array.reduce((groups, item) => {
      const key = keySelector(item);
      if (!groups[key]) groups[key] = [];
      groups[key].push(item);
      return groups;
    }, {});
  }
}

// Usage
const inputData = $input.all().map(item => item.json);

const result = new DataStream(inputData)
  .filter(item => item.status === 'active')
  .map(item => ({
    ...item,
    processedScore: item.score * 1.1,
    category: item.score > 80 ? 'excellent' : 'good'
  }))
  .groupBy(item => item.category)
  .execute();

return [{ json: result }];
```

### Batch Processing Pattern
```javascript
// Process items in batches
async function processBatches(items, batchSize = 10, processor) {
  const results = [];
  
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    
    try {
      const batchResults = await processor(batch);
      results.push(...batchResults);
      
      // Optional delay between batches
      if (i + batchSize < items.length) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
      
    } catch (error) {
      console.error(`Batch ${Math.floor(i / batchSize)} failed:`, error);
      
      // Add error entries for failed batch
      results.push(...batch.map(item => ({
        ...item,
        processingError: error.message,
        batchIndex: Math.floor(i / batchSize)
      })));
    }
  }
  
  return results;
}

// Batch processor function
async function processBatch(batch) {
  return batch.map(item => {
    // Simulate processing
    return {
      ...item,
      processed: true,
      processedAt: new Date().toISOString(),
      batchProcessed: true
    };
  });
}

// Execute batch processing
const inputItems = $input.all().map(item => item.json);
const processedItems = await processBatches(inputItems, 5, processBatch);

return processedItems.map(item => ({ json: item }));
```

## Performance Optimization

### Memory Management
```javascript
// Efficient memory usage
function processLargeDataset(items) {
  const results = [];
  const CHUNK_SIZE = 1000;
  
  // Process in chunks to avoid memory issues
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    
    // Process chunk
    const processedChunk = chunk.map(item => {
      // Only keep necessary fields
      return {
        id: item.id,
        result: processItem(item),
        timestamp: Date.now()
      };
    });
    
    results.push(...processedChunk);
    
    // Clear chunk reference
    chunk.length = 0;
  }
  
  return results;
}

function processItem(item) {
  // Lightweight processing only
  return {
    score: calculateScore(item),
    status: determineStatus(item)
  };
}

function calculateScore(item) {
  // Simple calculation
  return (item.value1 || 0) + (item.value2 || 0);
}

function determineStatus(item) {
  return item.score > 50 ? 'pass' : 'fail';
}

// Execute with memory management
const inputData = $input.all().map(item => item.json);
const results = processLargeDataset(inputData);

return results.map(item => ({ json: item }));
```

### Caching Strategies
```javascript
// Implement caching for expensive operations
class SimpleCache {
  constructor(ttl = 300000) { // 5 minutes default
    this.cache = new Map();
    this.ttl = ttl;
  }
  
  get(key) {
    const item = this.cache.get(key);
    if (!item) return null;
    
    if (Date.now() - item.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return item.value;
  }
  
  set(key, value) {
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    });
  }
  
  clear() {
    this.cache.clear();
  }
}

const cache = new SimpleCache();

function expensiveOperation(data) {
  const cacheKey = `operation_${data.id}_${data.type}`;
  
  // Check cache first
  let result = cache.get(cacheKey);
  if (result) {
    return { ...result, fromCache: true };
  }
  
  // Perform expensive operation
  result = {
    computed: data.value1 * data.value2 * Math.random(),
    processedAt: new Date().toISOString(),
    fromCache: false
  };
  
  // Cache the result
  cache.set(cacheKey, result);
  
  return result;
}

// Process with caching
const processedItems = $input.all().map(item => {
  const result = expensiveOperation(item.json);
  
  return {
    json: {
      ...item.json,
      operationResult: result
    }
  };
});

return processedItems;
```

## Real-world Examples

### Customer Segmentation Engine
```javascript
// Advanced customer segmentation
class CustomerSegmentation {
  constructor() {
    this.segments = {
      'champions': {
        criteria: (customer) => 
          customer.recency <= 30 && 
          customer.frequency >= 10 && 
          customer.monetary >= 1000,
        actions: ['vip_treatment', 'exclusive_offers', 'personal_account_manager']
      },
      'loyal_customers': {
        criteria: (customer) => 
          customer.recency <= 90 && 
          customer.frequency >= 5 && 
          customer.monetary >= 500,
        actions: ['loyalty_rewards', 'early_access', 'premium_support']
      },
      'potential_loyalists': {
        criteria: (customer) => 
          customer.recency <= 60 && 
          customer.frequency >= 3 && 
          customer.monetary >= 200,
        actions: ['engagement_campaign', 'product_recommendations', 'feedback_request']
      },
      'at_risk': {
        criteria: (customer) => 
          customer.recency > 120 && 
          customer.frequency >= 5 && 
          customer.monetary >= 300,
        actions: ['win_back_campaign', 'special_discount', 'personal_outreach']
      },
      'cannot_lose_them': {
        criteria: (customer) => 
          customer.recency > 90 && 
          customer.frequency >= 8 && 
          customer.monetary >= 800,
        actions: ['urgent_intervention', 'executive_contact', 'custom_offer']
      },
      'new_customers': {
        criteria: (customer) => 
          customer.recency <= 30 && 
          customer.frequency === 1,
        actions: ['welcome_series', 'onboarding_support', 'first_purchase_followup']
      }
    };
  }
  
  calculateRFM(customer) {
    const today = new Date();
    const lastPurchase = new Date(customer.lastPurchaseDate);
    
    return {
      recency: Math.floor((today - lastPurchase) / (1000 * 60 * 60 * 24)),
      frequency: customer.totalOrders || 0,
      monetary: customer.totalSpent || 0
    };
  }
  
  segmentCustomer(customer) {
    const rfm = this.calculateRFM(customer);
    const customerWithRFM = { ...customer, ...rfm };
    
    for (const [segmentName, segment] of Object.entries(this.segments)) {
      if (segment.criteria(customerWithRFM)) {
        return {
          customer: customerWithRFM,
          segment: segmentName,
          actions: segment.actions,
          segmentedAt: new Date().toISOString()
        };
      }
    }
    
    return {
      customer: customerWithRFM,
      segment: 'others',
      actions: ['general_newsletter'],
      segmentedAt: new Date().toISOString()
    };
  }
}

// Execute segmentation
const segmentation = new CustomerSegmentation();
const segmentedCustomers = $input.all().map(item => {
  const result = segmentation.segmentCustomer(item.json);
  return { json: result };
});

return segmentedCustomers;
```

### Financial Calculations
```javascript
// Advanced financial calculations
class FinancialCalculator {
  // Compound interest calculation
  static compoundInterest(principal, rate, time, compound = 12) {
    return principal * Math.pow((1 + rate / compound), compound * time);
  }
  
  // Loan payment calculation
  static loanPayment(principal, rate, periods) {
    const monthlyRate = rate / 12;
    return (principal * monthlyRate * Math.pow(1 + monthlyRate, periods)) / 
           (Math.pow(1 + monthlyRate, periods) - 1);
  }
  
  // Net Present Value
  static npv(rate, cashFlows) {
    return cashFlows.reduce((npv, cashFlow, index) => {
      return npv + cashFlow / Math.pow(1 + rate, index);
    }, 0);
  }
  
  // Internal Rate of Return (approximation)
  static irr(cashFlows, guess = 0.1) {
    const maxIterations = 100;
    const tolerance = 0.0001;
    
    for (let i = 0; i < maxIterations; i++) {
      const npv = this.npv(guess, cashFlows);
      const derivative = cashFlows.reduce((sum, cashFlow, index) => {
        return sum - index * cashFlow / Math.pow(1 + guess, index + 1);
      }, 0);
      
      if (Math.abs(npv) < tolerance) {
        return guess;
      }
      
      guess = guess - npv / derivative;
    }
    
    return guess;
  }
  
  // Amortization schedule
  static amortizationSchedule(principal, rate, periods) {
    const monthlyPayment = this.loanPayment(principal, rate, periods);
    const monthlyRate = rate / 12;
    const schedule = [];
    
    let remainingBalance = principal;
    
    for (let period = 1; period <= periods; period++) {
      const interestPayment = remainingBalance * monthlyRate;
      const principalPayment = monthlyPayment - interestPayment;
      remainingBalance -= principalPayment;
      
      schedule.push({
        period,
        payment: monthlyPayment,
        principalPayment,
        interestPayment,
        remainingBalance: Math.max(0, remainingBalance)
      });
    }
    
    return schedule;
  }
}

// Process financial data
const processedLoans = $input.all().map(item => {
  const loan = item.json;
  
  const monthlyPayment = FinancialCalculator.loanPayment(
    loan.principal,
    loan.annualRate / 100,
    loan.termMonths
  );
  
  const schedule = FinancialCalculator.amortizationSchedule(
    loan.principal,
    loan.annualRate / 100,
    loan.termMonths
  );
  
  const totalInterest = schedule.reduce((sum, payment) => sum + payment.interestPayment, 0);
  
  return {
    json: {
      ...loan,
      calculations: {
        monthlyPayment: Math.round(monthlyPayment * 100) / 100,
        totalInterest: Math.round(totalInterest * 100) / 100,
        totalPayments: Math.round((monthlyPayment * loan.termMonths) * 100) / 100,
        effectiveRate: (totalInterest / loan.principal) * 100
      },
      amortizationSchedule: schedule.slice(0, 12), // First year only
      calculatedAt: new Date().toISOString()
    }
  };
});

return processedLoans;
```

## Best Practices

### Code Organization
```javascript
// Modular function organization
const utils = {
  validation: {
    isEmail: (email) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email),
    isPhone: (phone) => /^\+?[\d\s\-\(\)]{10,}$/.test(phone),
    isUrl: (url) => {
      try { new URL(url); return true; } catch { return false; }
    }
  },
  
  formatting: {
    currency: (amount, currency = 'USD') => 
      new Intl.NumberFormat('en-US', { 
        style: 'currency', 
        currency 
      }).format(amount),
    
    phone: (phone) => {
      const cleaned = phone.replace(/\D/g, '');
      return cleaned.replace(/(\d{3})(\d{3})(\d{4})/, '($1) $2-$3');
    },
    
    date: (date, format = 'en-US') => 
      new Date(date).toLocaleDateString(format)
  },
  
  calculations: {
    percentage: (value, total) => 
      total > 0 ? (value / total) * 100 : 0,
    
    average: (numbers) => 
      numbers.reduce((a, b) => a + b, 0) / numbers.length,
    
    median: (numbers) => {
      const sorted = [...numbers].sort((a, b) => a - b);
      const mid = Math.floor(sorted.length / 2);
      return sorted.length % 2 ? sorted[mid] : (sorted[mid - 1] + sorted[mid]) / 2;
    }
  }
};

// Main processing logic
function processCustomerData(customer) {
  const processed = {
    id: customer.id,
    name: customer.name,
    email: utils.validation.isEmail(customer.email) ? customer.email : null,
    phone: customer.phone ? utils.formatting.phone(customer.phone) : null,
    isValid: utils.validation.isEmail(customer.email) && 
             utils.validation.isPhone(customer.phone),
    createdAt: utils.formatting.date(customer.createdAt),
    processedAt: new Date().toISOString()
  };
  
  if (customer.orders && Array.isArray(customer.orders)) {
    const orderAmounts = customer.orders.map(order => order.amount);
    processed.orderStats = {
      total: orderAmounts.reduce((a, b) => a + b, 0),
      average: utils.calculations.average(orderAmounts),
      median: utils.calculations.median(orderAmounts),
      count: orderAmounts.length
    };
  }
  
  return processed;
}

// Execute processing
const processedCustomers = $input.all().map(item => ({
  json: processCustomerData(item.json)
}));

return processedCustomers;
```

### Error Handling
```javascript
// Comprehensive error handling
function safeProcess(data, processor) {
  try {
    const result = processor(data);
    return {
      success: true,
      data: result,
      error: null
    };
  } catch (error) {
    return {
      success: false,
      data: null,
      error: {
        message: error.message,
        stack: error.stack,
        timestamp: new Date().toISOString()
      }
    };
  }
}

function processWithRetry(data, processor, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const result = safeProcess(data, processor);
    
    if (result.success) {
      return { ...result, attempts: attempt };
    }
    
    if (attempt === maxRetries) {
      return { ...result, attempts: attempt, maxRetriesExceeded: true };
    }
    
    // Wait before retry (exponential backoff)
    const delay = Math.pow(2, attempt) * 100;
    // Note: setTimeout won't work in N8N Code node, this is conceptual
  }
}

// Process items with error handling
const results = $input.all().map(item => {
  const result = safeProcess(item.json, (data) => {
    // Your processing logic here
    if (!data.id) throw new Error('ID is required');
    
    return {
      ...data,
      processed: true,
      timestamp: new Date().toISOString()
    };
  });
  
  return {
    json: {
      originalData: item.json,
      processing: result
    }
  };
});

return results;
```

### Performance Monitoring
```javascript
// Performance monitoring utilities
class PerformanceMonitor {
  constructor() {
    this.metrics = {};
  }
  
  startTimer(label) {
    this.metrics[label] = { startTime: Date.now() };
  }
  
  endTimer(label) {
    if (this.metrics[label]) {
      this.metrics[label].duration = Date.now() - this.metrics[label].startTime;
    }
  }
  
  getMetrics() {
    return { ...this.metrics };
  }
  
  reset() {
    this.metrics = {};
  }
}

const monitor = new PerformanceMonitor();

// Monitor performance
monitor.startTimer('total_processing');

const processedItems = [];

for (const item of $input.all()) {
  monitor.startTimer('item_processing');
  
  try {
    // Your processing logic
    const processed = {
      ...item.json,
      processed: true,
      timestamp: new Date().toISOString()
    };
    
    processedItems.push({ json: processed });
    
  } catch (error) {
    processedItems.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true
      }
    });
  }
  
  monitor.endTimer('item_processing');
}

monitor.endTimer('total_processing');

// Add performance metrics to result
const metrics = monitor.getMetrics();
console.log('Performance Metrics:', JSON.stringify(metrics, null, 2));

// Add summary to first item
if (processedItems.length > 0) {
  processedItems[0].json._performanceMetrics = {
    totalProcessingTime: metrics.total_processing.duration,
    averageItemTime: metrics.item_processing.duration / $input.all().length,
    itemsProcessed: processedItems.length,
    processingRate: (processedItems.length / metrics.total_processing.duration) * 1000 // items per second
  };
}

return processedItems;
```

## Practice Exercises

### Exercise 1: Advanced Data Pipeline
Create a function that:
1. Validates incoming data
2. Applies business rules
3. Transforms the data structure
4. Calculates derived metrics
5. Handles errors gracefully

### Exercise 2: Custom Analytics Engine
Build a system that:
1. Processes user behavior events
2. Calculates engagement scores
3. Determines user segments
4. Generates actionable insights

### Exercise 3: Financial Portfolio Calculator
Develop a portfolio analyzer that:
1. Calculates portfolio value
2. Determines asset allocation
3. Computes risk metrics
4. Generates rebalancing recommendations

## Next Steps

- Master [Error Handling & Debugging](./10-error-handling.md)
- Learn [Performance Optimization](./11-performance-optimization.md)
- Explore [Security Best Practices](./12-security-practices.md)

---

With custom functions and expressions, you can transform N8N into a powerful data processing and automation platform!
