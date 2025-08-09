# Data Transformation in N8N

Master the art of transforming and manipulating data in your N8N workflows.

## Table of Contents
- [Understanding Data Flow](#understanding-data-flow)
- [Basic Data Transformation](#basic-data-transformation)
- [Advanced Data Manipulation](#advanced-data-manipulation)
- [Working with Arrays](#working-with-arrays)
- [JSON Operations](#json-operations)
- [Data Validation](#data-validation)
- [Real-world Examples](#real-world-examples)
- [Best Practices](#best-practices)

## Understanding Data Flow

### How Data Moves in N8N
```javascript
// Data structure in N8N
{
  "json": {
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30
  },
  "binary": {},
  "pairedItem": 0
}
```

### Data Types
- **JSON**: Structured data objects
- **Binary**: Files, images, documents
- **Arrays**: Lists of items
- **Primitives**: Strings, numbers, booleans

## Basic Data Transformation

### Using the Set Node
```javascript
// Transform user data
{
  "fullName": "{{$json.firstName}} {{$json.lastName}}",
  "email": "{{$json.email.toLowerCase()}}",
  "isAdult": "{{$json.age >= 18}}"
}
```

### Function Node Basics
```javascript
// Simple transformation
for (const item of $input.all()) {
  item.json.processedAt = new Date().toISOString();
  item.json.fullName = `${item.json.firstName} ${item.json.lastName}`;
}

return $input.all();
```

### Code Node for Complex Logic
```javascript
// Advanced data processing
const processedItems = [];

for (const item of $input.all()) {
  const data = item.json;
  
  // Calculate age from birthdate
  const birthDate = new Date(data.birthDate);
  const age = Math.floor((Date.now() - birthDate.getTime()) / (365.25 * 24 * 60 * 60 * 1000));
  
  // Process address
  const address = {
    street: data.address?.street || '',
    city: data.address?.city || '',
    country: data.address?.country || 'Unknown'
  };
  
  processedItems.push({
    json: {
      id: data.id,
      name: data.name,
      age: age,
      address: address,
      tags: data.tags || [],
      createdAt: new Date().toISOString()
    }
  });
}

return processedItems;
```

## Advanced Data Manipulation

### Filtering Data
```javascript
// Filter items based on criteria
const filteredItems = $input.all().filter(item => {
  const data = item.json;
  return data.status === 'active' && data.score > 50;
});

return filteredItems;
```

### Grouping Data
```javascript
// Group items by category
const grouped = {};

for (const item of $input.all()) {
  const category = item.json.category;
  if (!grouped[category]) {
    grouped[category] = [];
  }
  grouped[category].push(item.json);
}

// Convert to array format
const result = Object.keys(grouped).map(category => ({
  json: {
    category: category,
    items: grouped[category],
    count: grouped[category].length
  }
}));

return result;
```

### Merging Data from Multiple Sources
```javascript
// Merge user data with order data
const users = $input.first().json;
const orders = $input.last().json;

const merged = users.map(user => {
  const userOrders = orders.filter(order => order.userId === user.id);
  return {
    ...user,
    orders: userOrders,
    totalOrders: userOrders.length,
    totalSpent: userOrders.reduce((sum, order) => sum + order.amount, 0)
  };
});

return merged.map(item => ({ json: item }));
```

## Working with Arrays

### Array Operations
```javascript
// Array manipulation examples
const data = $json.items; // Assuming items is an array

// Map operation
const mapped = data.map(item => ({
  id: item.id,
  name: item.name.toUpperCase(),
  slug: item.name.toLowerCase().replace(/\s+/g, '-')
}));

// Filter operation
const filtered = data.filter(item => item.active === true);

// Reduce operation
const summary = data.reduce((acc, item) => {
  acc.total += item.price;
  acc.count += 1;
  return acc;
}, { total: 0, count: 0 });

return [{
  json: {
    mapped: mapped,
    filtered: filtered,
    summary: summary
  }
}];
```

### Flattening Nested Arrays
```javascript
// Flatten nested structure
const flattenedItems = [];

for (const item of $input.all()) {
  const categories = item.json.categories || [];
  
  categories.forEach(category => {
    flattenedItems.push({
      json: {
        itemId: item.json.id,
        itemName: item.json.name,
        categoryId: category.id,
        categoryName: category.name
      }
    });
  });
}

return flattenedItems;
```

## JSON Operations

### Deep Object Manipulation
```javascript
// Working with nested objects
function updateNestedValue(obj, path, value) {
  const keys = path.split('.');
  let current = obj;
  
  for (let i = 0; i < keys.length - 1; i++) {
    if (!current[keys[i]]) {
      current[keys[i]] = {};
    }
    current = current[keys[i]];
  }
  
  current[keys[keys.length - 1]] = value;
  return obj;
}

const processedItems = $input.all().map(item => {
  const data = { ...item.json };
  
  // Update nested values
  updateNestedValue(data, 'metadata.updatedAt', new Date().toISOString());
  updateNestedValue(data, 'settings.notifications.email', true);
  
  return { json: data };
});

return processedItems;
```

### JSON Schema Validation
```javascript
// Validate data structure
function validateSchema(data, schema) {
  const errors = [];
  
  for (const [key, type] of Object.entries(schema)) {
    if (!(key in data)) {
      errors.push(`Missing required field: ${key}`);
    } else if (typeof data[key] !== type) {
      errors.push(`Invalid type for ${key}: expected ${type}, got ${typeof data[key]}`);
    }
  }
  
  return errors;
}

const schema = {
  id: 'number',
  name: 'string',
  email: 'string',
  active: 'boolean'
};

const validatedItems = $input.all().map(item => {
  const errors = validateSchema(item.json, schema);
  
  return {
    json: {
      ...item.json,
      validation: {
        isValid: errors.length === 0,
        errors: errors
      }
    }
  };
});

return validatedItems;
```

## Data Validation

### Input Validation
```javascript
// Comprehensive validation
function validateUser(user) {
  const errors = [];
  
  // Required fields
  if (!user.email) errors.push('Email is required');
  if (!user.name) errors.push('Name is required');
  
  // Format validation
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (user.email && !emailRegex.test(user.email)) {
    errors.push('Invalid email format');
  }
  
  // Range validation
  if (user.age && (user.age < 0 || user.age > 150)) {
    errors.push('Age must be between 0 and 150');
  }
  
  return {
    isValid: errors.length === 0,
    errors: errors,
    data: user
  };
}

const validatedUsers = $input.all().map(item => {
  const validation = validateUser(item.json);
  
  return {
    json: {
      ...validation.data,
      _validation: validation
    }
  };
});

return validatedUsers;
```

### Data Sanitization
```javascript
// Clean and sanitize data
function sanitizeData(data) {
  const sanitized = {};
  
  for (const [key, value] of Object.entries(data)) {
    if (typeof value === 'string') {
      // Trim whitespace and remove special characters
      sanitized[key] = value.trim().replace(/[<>]/g, '');
    } else if (typeof value === 'number') {
      // Ensure valid numbers
      sanitized[key] = isNaN(value) ? 0 : value;
    } else if (Array.isArray(value)) {
      // Filter out invalid array items
      sanitized[key] = value.filter(item => item != null);
    } else {
      sanitized[key] = value;
    }
  }
  
  return sanitized;
}

const sanitizedItems = $input.all().map(item => ({
  json: sanitizeData(item.json)
}));

return sanitizedItems;
```

## Real-world Examples

### E-commerce Order Processing
```javascript
// Process e-commerce orders
const orders = $input.all();
const processedOrders = [];

for (const order of orders) {
  const data = order.json;
  
  // Calculate totals
  const subtotal = data.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
  const tax = subtotal * 0.1; // 10% tax
  const shipping = subtotal > 100 ? 0 : 15; // Free shipping over $100
  const total = subtotal + tax + shipping;
  
  // Determine shipping method
  const shippingMethod = total > 200 ? 'express' : 'standard';
  
  // Process customer data
  const customer = {
    id: data.customer.id,
    name: `${data.customer.firstName} ${data.customer.lastName}`,
    email: data.customer.email.toLowerCase(),
    tier: total > 500 ? 'premium' : 'standard'
  };
  
  processedOrders.push({
    json: {
      orderId: data.id,
      customer: customer,
      items: data.items,
      pricing: {
        subtotal: subtotal,
        tax: tax,
        shipping: shipping,
        total: total
      },
      shipping: {
        method: shippingMethod,
        address: data.shippingAddress
      },
      status: 'processing',
      createdAt: new Date().toISOString()
    }
  });
}

return processedOrders;
```

### Customer Data Enrichment
```javascript
// Enrich customer data from multiple sources
const customers = $input.first().json; // Customer data
const interactions = $input.last().json; // Interaction data

const enrichedCustomers = customers.map(customer => {
  // Find customer interactions
  const customerInteractions = interactions.filter(int => int.customerId === customer.id);
  
  // Calculate engagement metrics
  const totalInteractions = customerInteractions.length;
  const lastInteraction = customerInteractions.length > 0 
    ? Math.max(...customerInteractions.map(int => new Date(int.date).getTime()))
    : null;
  
  // Determine customer segment
  let segment = 'new';
  if (totalInteractions > 50) segment = 'champion';
  else if (totalInteractions > 20) segment = 'loyal';
  else if (totalInteractions > 5) segment = 'engaged';
  
  // Calculate lifetime value
  const lifetimeValue = customerInteractions
    .filter(int => int.type === 'purchase')
    .reduce((sum, int) => sum + (int.value || 0), 0);
  
  return {
    json: {
      ...customer,
      metrics: {
        totalInteractions: totalInteractions,
        lastInteractionDate: lastInteraction ? new Date(lastInteraction).toISOString() : null,
        segment: segment,
        lifetimeValue: lifetimeValue,
        averageOrderValue: totalInteractions > 0 ? lifetimeValue / totalInteractions : 0
      },
      enrichedAt: new Date().toISOString()
    }
  };
});

return enrichedCustomers;
```

## Best Practices

### Performance Optimization
```javascript
// Efficient data processing
const BATCH_SIZE = 1000;
const items = $input.all();
const processedBatches = [];

// Process in batches
for (let i = 0; i < items.length; i += BATCH_SIZE) {
  const batch = items.slice(i, i + BATCH_SIZE);
  
  const processedBatch = batch.map(item => {
    // Lightweight processing only
    return {
      json: {
        id: item.json.id,
        processed: true,
        timestamp: Date.now()
      }
    };
  });
  
  processedBatches.push(...processedBatch);
}

return processedBatches;
```

### Error Handling
```javascript
// Robust error handling
const results = [];
const errors = [];

for (const item of $input.all()) {
  try {
    // Risky operation
    const processed = {
      id: item.json.id,
      result: JSON.parse(item.json.data),
      processedAt: new Date().toISOString()
    };
    
    results.push({ json: processed });
  } catch (error) {
    errors.push({
      json: {
        itemId: item.json.id,
        error: error.message,
        originalData: item.json,
        failedAt: new Date().toISOString()
      }
    });
  }
}

// Return both successful and failed items
return [...results, ...errors];
```

### Data Consistency
```javascript
// Ensure data consistency
function normalizeData(data) {
  return {
    id: data.id ? String(data.id) : null,
    name: data.name ? data.name.trim() : '',
    email: data.email ? data.email.toLowerCase().trim() : null,
    phone: data.phone ? data.phone.replace(/\D/g, '') : null,
    createdAt: data.createdAt ? new Date(data.createdAt).toISOString() : new Date().toISOString(),
    updatedAt: new Date().toISOString()
  };
}

const normalizedItems = $input.all().map(item => ({
  json: normalizeData(item.json)
}));

return normalizedItems;
```

## Practice Exercises

### Exercise 1: Customer Segmentation
Create a workflow that segments customers based on:
- Purchase frequency
- Total spend
- Last purchase date
- Product categories

### Exercise 2: Data Quality Report
Build a data quality checker that identifies:
- Missing required fields
- Invalid data formats
- Duplicate records
- Outliers

### Exercise 3: Multi-source Data Merge
Combine data from:
- CRM system
- E-commerce platform
- Support tickets
- Marketing campaigns

## Next Steps

- Practice with [API Integration Guide](./07-api-integration.md)
- Learn about [Webhook & Triggers](./08-webhooks-triggers.md)
- Explore [Custom Functions & Expressions](./09-custom-functions.md)

---

Remember: Clean, well-structured data transformations are the foundation of reliable automation workflows!
