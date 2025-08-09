# Performance Optimization for N8N Workflows

Master the art of building fast, efficient, and scalable N8N workflows that perform optimally under any load.

## Table of Contents
- [Performance Fundamentals](#performance-fundamentals)
- [Workflow Architecture Optimization](#workflow-architecture-optimization)
- [Node-Level Optimizations](#node-level-optimizations)
- [Memory Management](#memory-management)
- [Database Query Optimization](#database-query-optimization)
- [Caching Strategies](#caching-strategies)
- [Parallel Processing](#parallel-processing)
- [Resource Monitoring](#resource-monitoring)
- [Scaling Strategies](#scaling-strategies)
- [Performance Testing](#performance-testing)

## Performance Fundamentals

### Understanding N8N Performance Metrics
```javascript
// Key performance indicators to monitor
const performanceMetrics = {
  executionTime: 'Total time to complete workflow',
  nodeExecutionTime: 'Time per individual node',
  memoryUsage: 'RAM consumption during execution',
  cpuUtilization: 'Processor usage percentage',
  throughput: 'Items processed per second',
  concurrency: 'Simultaneous workflow executions',
  errorRate: 'Percentage of failed executions',
  resourceUtilization: 'Overall system resource usage'
};

// Performance monitoring utility
class PerformanceProfiler {
  constructor() {
    this.metrics = new Map();
    this.startTimes = new Map();
  }
  
  startTimer(label) {
    this.startTimes.set(label, process.hrtime.bigint());
  }
  
  endTimer(label) {
    const startTime = this.startTimes.get(label);
    if (startTime) {
      const endTime = process.hrtime.bigint();
      const duration = Number(endTime - startTime) / 1000000; // Convert to milliseconds
      
      if (!this.metrics.has(label)) {
        this.metrics.set(label, []);
      }
      this.metrics.get(label).push(duration);
      
      return duration;
    }
    return null;
  }
  
  getStats(label) {
    const times = this.metrics.get(label) || [];
    if (times.length === 0) return null;
    
    const sorted = times.sort((a, b) => a - b);
    return {
      count: times.length,
      min: Math.min(...times),
      max: Math.max(...times),
      avg: times.reduce((a, b) => a + b, 0) / times.length,
      p50: sorted[Math.floor(times.length * 0.5)],
      p95: sorted[Math.floor(times.length * 0.95)],
      p99: sorted[Math.floor(times.length * 0.99)]
    };
  }
  
  getAllStats() {
    const stats = {};
    for (const label of this.metrics.keys()) {
      stats[label] = this.getStats(label);
    }
    return stats;
  }
}

// Usage in workflow
const profiler = new PerformanceProfiler();
```

### Performance Bottleneck Identification
```javascript
// Bottleneck detection system
class BottleneckDetector {
  constructor(thresholds = {}) {
    this.thresholds = {
      slowNodeMs: thresholds.slowNodeMs || 5000,
      highMemoryMB: thresholds.highMemoryMB || 512,
      lowThroughput: thresholds.lowThroughput || 10,
      ...thresholds
    };
    this.metrics = [];
  }
  
  analyzeExecution(executionData) {
    const bottlenecks = [];
    
    // Check slow nodes
    if (executionData.nodeExecutionTime > this.thresholds.slowNodeMs) {
      bottlenecks.push({
        type: 'SLOW_NODE',
        severity: 'HIGH',
        node: executionData.nodeName,
        value: executionData.nodeExecutionTime,
        threshold: this.thresholds.slowNodeMs,
        suggestion: 'Optimize node logic or add caching'
      });
    }
    
    // Check memory usage
    if (executionData.memoryUsage > this.thresholds.highMemoryMB) {
      bottlenecks.push({
        type: 'HIGH_MEMORY',
        severity: 'MEDIUM',
        value: executionData.memoryUsage,
        threshold: this.thresholds.highMemoryMB,
        suggestion: 'Implement streaming or batch processing'
      });
    }
    
    // Check throughput
    if (executionData.throughput < this.thresholds.lowThroughput) {
      bottlenecks.push({
        type: 'LOW_THROUGHPUT',
        severity: 'MEDIUM',
        value: executionData.throughput,
        threshold: this.thresholds.lowThroughput,
        suggestion: 'Parallelize processing or optimize algorithms'
      });
    }
    
    return bottlenecks;
  }
  
  generateReport() {
    const grouped = this.metrics.reduce((acc, metric) => {
      const key = `${metric.type}_${metric.severity}`;
      if (!acc[key]) acc[key] = [];
      acc[key].push(metric);
      return acc;
    }, {});
    
    return {
      summary: {
        totalBottlenecks: this.metrics.length,
        highSeverity: this.metrics.filter(m => m.severity === 'HIGH').length,
        mediumSeverity: this.metrics.filter(m => m.severity === 'MEDIUM').length
      },
      groupedBottlenecks: grouped,
      recommendations: this.generateRecommendations()
    };
  }
  
  generateRecommendations() {
    const recommendations = [];
    const bottleneckTypes = [...new Set(this.metrics.map(m => m.type))];
    
    if (bottleneckTypes.includes('SLOW_NODE')) {
      recommendations.push('Consider breaking down complex nodes into smaller, focused operations');
    }
    
    if (bottleneckTypes.includes('HIGH_MEMORY')) {
      recommendations.push('Implement streaming processing for large datasets');
    }
    
    if (bottleneckTypes.includes('LOW_THROUGHPUT')) {
      recommendations.push('Investigate parallel processing opportunities');
    }
    
    return recommendations;
  }
}
```

## Workflow Architecture Optimization

### Efficient Workflow Design Patterns
```javascript
// Optimized workflow architecture patterns
const OptimizedWorkflowPatterns = {
  // 1. Fan-out/Fan-in Pattern for Parallel Processing
  fanOutFanIn: {
    description: 'Process multiple items in parallel, then aggregate results',
    implementation: `
      Trigger → Split Data → [Process A, Process B, Process C] → Merge Results → Output
    `,
    benefits: ['Reduced total execution time', 'Better resource utilization', 'Improved throughput']
  },
  
  // 2. Pipeline Pattern for Sequential Optimization
  pipeline: {
    description: 'Break complex processing into optimized stages',
    implementation: `
      Stage 1 (Fast) → Stage 2 (Medium) → Stage 3 (Slow) → Output
    `,
    benefits: ['Predictable performance', 'Easy to debug', 'Resource-efficient']
  },
  
  // 3. Circuit Breaker Pattern for Resilience
  circuitBreaker: {
    description: 'Prevent cascading failures and resource exhaustion',
    implementation: `
      Input → Health Check → [Process | Fallback] → Output
    `,
    benefits: ['Prevents system overload', 'Graceful degradation', 'Faster failure recovery']
  }
};

// Workflow optimization analyzer
function analyzeWorkflowArchitecture(workflow) {
  const analysis = {
    nodeCount: workflow.nodes.length,
    maxDepth: calculateMaxDepth(workflow),
    parallelBranches: countParallelBranches(workflow),
    bottleneckNodes: identifyBottleneckNodes(workflow),
    optimizationOpportunities: []
  };
  
  // Check for optimization opportunities
  if (analysis.nodeCount > 20) {
    analysis.optimizationOpportunities.push({
      type: 'COMPLEXITY',
      suggestion: 'Consider breaking workflow into sub-workflows',
      impact: 'Improved maintainability and debugging'
    });
  }
  
  if (analysis.maxDepth > 10) {
    analysis.optimizationOpportunities.push({
      type: 'DEPTH',
      suggestion: 'Implement parallel processing to reduce execution depth',
      impact: 'Reduced total execution time'
    });
  }
  
  if (analysis.parallelBranches < 2 && analysis.nodeCount > 5) {
    analysis.optimizationOpportunities.push({
      type: 'PARALLELIZATION',
      suggestion: 'Identify opportunities for parallel execution',
      impact: 'Better resource utilization and faster processing'
    });
  }
  
  return analysis;
}

function calculateMaxDepth(workflow) {
  // Calculate the maximum execution depth
  const visited = new Set();
  const depths = new Map();
  
  function dfs(nodeId, depth = 0) {
    if (visited.has(nodeId)) return depths.get(nodeId) || 0;
    
    visited.add(nodeId);
    depths.set(nodeId, depth);
    
    const node = workflow.nodes.find(n => n.id === nodeId);
    if (!node || !node.connections) return depth;
    
    let maxChildDepth = depth;
    for (const connection of node.connections) {
      const childDepth = dfs(connection.targetNodeId, depth + 1);
      maxChildDepth = Math.max(maxChildDepth, childDepth);
    }
    
    return maxChildDepth;
  }
  
  const startNodes = workflow.nodes.filter(n => n.type === 'trigger');
  return Math.max(...startNodes.map(n => dfs(n.id)));
}

function countParallelBranches(workflow) {
  // Count nodes that can execute in parallel
  let maxParallel = 0;
  const levels = new Map();
  
  // Group nodes by execution level
  workflow.nodes.forEach(node => {
    const level = calculateNodeLevel(node, workflow);
    if (!levels.has(level)) levels.set(level, []);
    levels.get(level).push(node);
  });
  
  // Find maximum parallel execution at any level
  for (const nodesAtLevel of levels.values()) {
    maxParallel = Math.max(maxParallel, nodesAtLevel.length);
  }
  
  return maxParallel;
}

function calculateNodeLevel(targetNode, workflow) {
  // Calculate execution level (depth from trigger)
  const visited = new Set();
  
  function findLevel(nodeId, level = 0) {
    if (visited.has(nodeId)) return level;
    visited.add(nodeId);
    
    const node = workflow.nodes.find(n => n.id === nodeId);
    if (!node) return level;
    
    if (node.type === 'trigger') return level;
    
    // Find all nodes that connect to this node
    const predecessors = workflow.nodes.filter(n => 
      n.connections && n.connections.some(c => c.targetNodeId === nodeId)
    );
    
    if (predecessors.length === 0) return level;
    
    const maxPredecessorLevel = Math.max(
      ...predecessors.map(p => findLevel(p.id, level))
    );
    
    return maxPredecessorLevel + 1;
  }
  
  return findLevel(targetNode.id);
}
```

### Sub-workflow Optimization
```javascript
// Sub-workflow performance optimization
class SubWorkflowOptimizer {
  constructor() {
    this.cache = new Map();
    this.metrics = new Map();
  }
  
  optimizeSubWorkflow(subWorkflowConfig) {
    const optimizations = [];
    
    // 1. Input/Output optimization
    const ioOptimization = this.optimizeIO(subWorkflowConfig);
    if (ioOptimization) optimizations.push(ioOptimization);
    
    // 2. Caching optimization
    const cachingOptimization = this.optimizeCaching(subWorkflowConfig);
    if (cachingOptimization) optimizations.push(cachingOptimization);
    
    // 3. Batch processing optimization
    const batchOptimization = this.optimizeBatching(subWorkflowConfig);
    if (batchOptimization) optimizations.push(batchOptimization);
    
    return {
      originalConfig: subWorkflowConfig,
      optimizations: optimizations,
      estimatedImprovement: this.calculateImprovement(optimizations)
    };
  }
  
  optimizeIO(config) {
    // Minimize data transfer between parent and sub-workflow
    const inputSize = this.estimateDataSize(config.inputs);
    const outputSize = this.estimateDataSize(config.outputs);
    
    if (inputSize > 1024 * 1024) { // > 1MB
      return {
        type: 'IO_OPTIMIZATION',
        description: 'Large input data detected',
        recommendation: 'Pass data references instead of full data',
        implementation: {
          before: 'Pass entire dataset to sub-workflow',
          after: 'Pass data ID/reference and fetch within sub-workflow'
        },
        estimatedSaving: '70% reduction in data transfer time'
      };
    }
    
    return null;
  }
  
  optimizeCaching(config) {
    // Check if sub-workflow results can be cached
    const isDeterministic = this.isDeterministicWorkflow(config);
    const hasExpensiveOperations = this.hasExpensiveOperations(config);
    
    if (isDeterministic && hasExpensiveOperations) {
      return {
        type: 'CACHING_OPTIMIZATION',
        description: 'Deterministic workflow with expensive operations',
        recommendation: 'Implement result caching based on input hash',
        implementation: {
          cacheKey: 'SHA256 hash of normalized inputs',
          ttl: '1 hour for stable data, 5 minutes for dynamic data',
          storage: 'Redis or in-memory cache'
        },
        estimatedSaving: '90% reduction for cache hits'
      };
    }
    
    return null;
  }
  
  optimizeBatching(config) {
    // Check if sub-workflow can benefit from batch processing
    const canBatch = this.canBatchProcess(config);
    const itemCount = config.expectedItemCount || 1;
    
    if (canBatch && itemCount > 10) {
      return {
        type: 'BATCH_OPTIMIZATION',
        description: 'Multiple items can be processed in batches',
        recommendation: 'Implement batch processing within sub-workflow',
        implementation: {
          batchSize: Math.min(100, Math.ceil(itemCount / 10)),
          strategy: 'Group items by common processing requirements',
          benefits: ['Reduced API calls', 'Better resource utilization']
        },
        estimatedSaving: '60% reduction in total execution time'
      };
    }
    
    return null;
  }
  
  estimateDataSize(data) {
    return JSON.stringify(data || {}).length;
  }
  
  isDeterministicWorkflow(config) {
    // Check if workflow produces same output for same input
    const nonDeterministicNodes = [
      'randomNumber', 'dateTime', 'uuid', 'httpRequest'
    ];
    
    return !config.nodes.some(node => 
      nonDeterministicNodes.includes(node.type)
    );
  }
  
  hasExpensiveOperations(config) {
    const expensiveNodes = [
      'httpRequest', 'database', 'apiCall', 'fileOperation'
    ];
    
    return config.nodes.some(node => 
      expensiveNodes.includes(node.type)
    );
  }
  
  canBatchProcess(config) {
    // Check if workflow can process multiple items together
    const batchableNodes = [
      'database', 'httpRequest', 'emailSend', 'fileWrite'
    ];
    
    return config.nodes.some(node => 
      batchableNodes.includes(node.type)
    );
  }
  
  calculateImprovement(optimizations) {
    if (optimizations.length === 0) return 0;
    
    // Estimate combined improvement (not additive)
    const improvements = optimizations.map(opt => {
      const match = opt.estimatedSaving.match(/(\d+)%/);
      return match ? parseInt(match[1]) / 100 : 0;
    });
    
    // Calculate compound improvement
    const combinedImprovement = 1 - improvements.reduce(
      (acc, improvement) => acc * (1 - improvement), 1
    );
    
    return Math.round(combinedImprovement * 100);
  }
}

// Usage example
const optimizer = new SubWorkflowOptimizer();

const subWorkflowConfig = {
  inputs: { userIds: [1, 2, 3, 4, 5] },
  outputs: { processedUsers: [] },
  nodes: [
    { type: 'httpRequest', name: 'fetchUserData' },
    { type: 'database', name: 'updateUserProfile' },
    { type: 'emailSend', name: 'sendNotification' }
  ],
  expectedItemCount: 100
};

const optimizationPlan = optimizer.optimizeSubWorkflow(subWorkflowConfig);
```

## Node-Level Optimizations

### HTTP Request Optimization
```javascript
// Optimized HTTP request patterns
class OptimizedHTTPClient {
  constructor(options = {}) {
    this.options = {
      maxConcurrent: options.maxConcurrent || 10,
      timeout: options.timeout || 30000,
      retryAttempts: options.retryAttempts || 3,
      retryDelay: options.retryDelay || 1000,
      keepAlive: options.keepAlive !== false,
      ...options
    };
    
    this.activeRequests = new Map();
    this.requestQueue = [];
    this.stats = {
      totalRequests: 0,
      successfulRequests: 0,
      failedRequests: 0,
      averageResponseTime: 0,
      cacheHits: 0
    };
    
    this.cache = new Map();
  }
  
  async makeRequest(config) {
    this.stats.totalRequests++;
    
    // Check cache first
    const cacheKey = this.generateCacheKey(config);
    if (this.shouldUseCache(config) && this.cache.has(cacheKey)) {
      this.stats.cacheHits++;
      return this.cache.get(cacheKey);
    }
    
    const startTime = Date.now();
    
    try {
      // Add to queue if too many concurrent requests
      if (this.activeRequests.size >= this.options.maxConcurrent) {
        await this.waitForSlot();
      }
      
      const requestId = this.generateRequestId();
      this.activeRequests.set(requestId, config);
      
      const response = await this.executeRequest(config);
      
      this.activeRequests.delete(requestId);
      this.stats.successfulRequests++;
      
      // Update average response time
      const responseTime = Date.now() - startTime;
      this.updateAverageResponseTime(responseTime);
      
      // Cache if appropriate
      if (this.shouldCache(config, response)) {
        this.cache.set(cacheKey, response);
      }
      
      return response;
      
    } catch (error) {
      this.stats.failedRequests++;
      throw error;
    }
  }
  
  async executeBatchRequests(requests) {
    // Group requests by domain for optimal connection reuse
    const groupedRequests = this.groupRequestsByDomain(requests);
    const results = [];
    
    for (const [domain, domainRequests] of Object.entries(groupedRequests)) {
      // Process requests for each domain with controlled concurrency
      const domainResults = await this.processDomainRequests(domainRequests);
      results.push(...domainResults);
    }
    
    return results;
  }
  
  groupRequestsByDomain(requests) {
    return requests.reduce((groups, request) => {
      const url = new URL(request.url);
      const domain = url.hostname;
      
      if (!groups[domain]) groups[domain] = [];
      groups[domain].push(request);
      
      return groups;
    }, {});
  }
  
  async processDomainRequests(requests) {
    const semaphore = new Semaphore(5); // Max 5 concurrent per domain
    
    return Promise.all(requests.map(async (request) => {
      await semaphore.acquire();
      
      try {
        return await this.makeRequest(request);
      } finally {
        semaphore.release();
      }
    }));
  }
  
  generateCacheKey(config) {
    const normalized = {
      url: config.url,
      method: config.method || 'GET',
      body: config.body,
      headers: this.normalizeHeaders(config.headers || {})
    };
    
    return JSON.stringify(normalized);
  }
  
  shouldUseCache(config) {
    return (config.method || 'GET').toUpperCase() === 'GET' && 
           !config.noCache;
  }
  
  shouldCache(config, response) {
    return this.shouldUseCache(config) && 
           response.status >= 200 && 
           response.status < 300;
  }
  
  normalizeHeaders(headers) {
    const normalized = {};
    
    // Exclude dynamic headers that shouldn't affect caching
    const excludeHeaders = ['user-agent', 'accept-encoding', 'authorization'];
    
    for (const [key, value] of Object.entries(headers)) {
      if (!excludeHeaders.includes(key.toLowerCase())) {
        normalized[key.toLowerCase()] = value;
      }
    }
    
    return normalized;
  }
  
  async executeRequest(config) {
    let lastError = null;
    
    for (let attempt = 1; attempt <= this.options.retryAttempts; attempt++) {
      try {
        const controller = new AbortController();
        const timeoutId = setTimeout(() => controller.abort(), this.options.timeout);
        
        const response = await fetch(config.url, {
          ...config,
          signal: controller.signal
        });
        
        clearTimeout(timeoutId);
        
        if (!response.ok && !this.shouldRetry(response.status)) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        if (response.ok) {
          return {
            status: response.status,
            headers: Object.fromEntries(response.headers.entries()),
            data: await response.json()
          };
        }
        
      } catch (error) {
        lastError = error;
        
        if (attempt < this.options.retryAttempts && this.shouldRetry(null, error)) {
          const delay = this.options.retryDelay * Math.pow(2, attempt - 1);
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        
        break;
      }
    }
    
    throw lastError;
  }
  
  shouldRetry(statusCode, error) {
    // Retry on network errors or specific HTTP status codes
    if (error && (error.name === 'AbortError' || error.code === 'ECONNRESET')) {
      return true;
    }
    
    if (statusCode) {
      return [408, 429, 500, 502, 503, 504].includes(statusCode);
    }
    
    return false;
  }
  
  async waitForSlot() {
    return new Promise(resolve => {
      const checkSlot = () => {
        if (this.activeRequests.size < this.options.maxConcurrent) {
          resolve();
        } else {
          setTimeout(checkSlot, 10);
        }
      };
      checkSlot();
    });
  }
  
  generateRequestId() {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  updateAverageResponseTime(responseTime) {
    const totalRequests = this.stats.successfulRequests;
    const currentAverage = this.stats.averageResponseTime;
    
    this.stats.averageResponseTime = 
      (currentAverage * (totalRequests - 1) + responseTime) / totalRequests;
  }
  
  getStats() {
    return {
      ...this.stats,
      successRate: this.stats.totalRequests > 0 
        ? (this.stats.successfulRequests / this.stats.totalRequests) * 100 
        : 0,
      cacheHitRate: this.stats.totalRequests > 0
        ? (this.stats.cacheHits / this.stats.totalRequests) * 100
        : 0,
      activeRequests: this.activeRequests.size,
      cacheSize: this.cache.size
    };
  }
  
  clearCache() {
    this.cache.clear();
  }
}

// Semaphore utility for concurrency control
class Semaphore {
  constructor(maxConcurrent) {
    this.maxConcurrent = maxConcurrent;
    this.running = 0;
    this.waiting = [];
  }
  
  async acquire() {
    if (this.running >= this.maxConcurrent) {
      await new Promise(resolve => this.waiting.push(resolve));
    }
    this.running++;
  }
  
  release() {
    this.running--;
    if (this.waiting.length > 0) {
      const resolve = this.waiting.shift();
      resolve();
    }
  }
}

// Usage in N8N workflow
const httpClient = new OptimizedHTTPClient({
  maxConcurrent: 10,
  timeout: 30000,
  retryAttempts: 3
});

const results = [];

for (const item of $input.all()) {
  try {
    const result = await httpClient.makeRequest({
      url: `https://api.example.com/data/${item.json.id}`,
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${$credentials.token}`
      }
    });
    
    results.push({
      json: {
        ...item.json,
        apiData: result.data,
        performance: {
          responseTime: result.responseTime,
          fromCache: result.fromCache
        }
      }
    });
    
  } catch (error) {
    results.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true
      }
    });
  }
}

// Add performance statistics
results.push({
  json: {
    type: 'performance_stats',
    stats: httpClient.getStats()
  }
});

return results;
```

### Database Query Optimization
```javascript
// Database query optimization patterns
class DatabaseOptimizer {
  constructor(connectionConfig) {
    this.config = connectionConfig;
    this.queryCache = new Map();
    this.connectionPool = new ConnectionPool(connectionConfig);
    this.metrics = {
      totalQueries: 0,
      cacheHits: 0,
      averageQueryTime: 0,
      slowQueries: []
    };
  }
  
  async executeOptimizedQuery(query, params = [], options = {}) {
    const startTime = Date.now();
    this.metrics.totalQueries++;
    
    try {
      // Check cache for read queries
      if (this.isReadQuery(query) && options.cache !== false) {
        const cacheKey = this.generateCacheKey(query, params);
        
        if (this.queryCache.has(cacheKey)) {
          this.metrics.cacheHits++;
          return this.queryCache.get(cacheKey);
        }
      }
      
      // Analyze and optimize query
      const optimizedQuery = this.optimizeQuery(query);
      
      // Execute query with connection pooling
      const connection = await this.connectionPool.getConnection();
      
      try {
        const result = await connection.execute(optimizedQuery, params);
        
        // Cache read query results
        if (this.isReadQuery(query) && options.cache !== false) {
          const cacheKey = this.generateCacheKey(query, params);
          this.queryCache.set(cacheKey, result);
          
          // Set cache expiration
          setTimeout(() => {
            this.queryCache.delete(cacheKey);
          }, options.cacheTTL || 300000); // 5 minutes default
        }
        
        return result;
        
      } finally {
        this.connectionPool.releaseConnection(connection);
      }
      
    } finally {
      const queryTime = Date.now() - startTime;
      this.updateQueryMetrics(query, queryTime);
    }
  }
  
  async executeBatchQuery(queries) {
    const connection = await this.connectionPool.getConnection();
    
    try {
      // Begin transaction for batch operations
      await connection.beginTransaction();
      
      const results = [];
      
      for (const { query, params } of queries) {
        const result = await connection.execute(query, params);
        results.push(result);
      }
      
      await connection.commit();
      return results;
      
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      this.connectionPool.releaseConnection(connection);
    }
  }
  
  optimizeQuery(query) {
    // Basic query optimization techniques
    let optimized = query;
    
    // 1. Add LIMIT if not present for potentially large result sets
    if (this.isSelectQuery(query) && !query.toUpperCase().includes('LIMIT')) {
      // Only add LIMIT if it's a simple SELECT without aggregation
      if (!this.hasAggregation(query)) {
        console.warn('Consider adding LIMIT to prevent large result sets');
      }
    }
    
    // 2. Suggest indexes for WHERE clauses
    const whereColumns = this.extractWhereColumns(query);
    if (whereColumns.length > 0) {
      console.log(`Consider indexes on columns: ${whereColumns.join(', ')}`);
    }
    
    // 3. Optimize JOIN order (larger tables first)
    if (query.toUpperCase().includes('JOIN')) {
      console.log('Review JOIN order - larger tables should be joined first');
    }
    
    return optimized;
  }
  
  isReadQuery(query) {
    const readOperations = ['SELECT', 'SHOW', 'DESCRIBE', 'EXPLAIN'];
    const operation = query.trim().split(' ')[0].toUpperCase();
    return readOperations.includes(operation);
  }
  
  isSelectQuery(query) {
    return query.trim().toUpperCase().startsWith('SELECT');
  }
  
  hasAggregation(query) {
    const aggregateFunctions = ['COUNT', 'SUM', 'AVG', 'MAX', 'MIN', 'GROUP BY'];
    const upperQuery = query.toUpperCase();
    return aggregateFunctions.some(func => upperQuery.includes(func));
  }
  
  extractWhereColumns(query) {
    // Simple regex to extract column names from WHERE clauses
    const whereMatch = query.match(/WHERE\s+(.+?)(?:\s+ORDER\s+BY|\s+GROUP\s+BY|\s+LIMIT|$)/i);
    if (!whereMatch) return [];
    
    const whereClause = whereMatch[1];
    const columnMatches = whereClause.match(/(\w+)\s*[=<>!]/g) || [];
    
    return columnMatches.map(match => match.replace(/\s*[=<>!].*/, ''));
  }
  
  generateCacheKey(query, params) {
    const normalized = query.replace(/\s+/g, ' ').trim().toLowerCase();
    return `${normalized}_${JSON.stringify(params)}`;
  }
  
  updateQueryMetrics(query, queryTime) {
    // Update average query time
    const totalQueries = this.metrics.totalQueries;
    const currentAverage = this.metrics.averageQueryTime;
    
    this.metrics.averageQueryTime = 
      (currentAverage * (totalQueries - 1) + queryTime) / totalQueries;
    
    // Track slow queries
    if (queryTime > 5000) { // Queries slower than 5 seconds
      this.metrics.slowQueries.push({
        query: query.substring(0, 100) + '...',
        executionTime: queryTime,
        timestamp: new Date().toISOString()
      });
      
      // Keep only last 10 slow queries
      if (this.metrics.slowQueries.length > 10) {
        this.metrics.slowQueries.shift();
      }
    }
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      cacheHitRate: this.metrics.totalQueries > 0
        ? (this.metrics.cacheHits / this.metrics.totalQueries) * 100
        : 0,
      cacheSize: this.queryCache.size
    };
  }
  
  clearCache() {
    this.queryCache.clear();
  }
}

// Connection pool implementation
class ConnectionPool {
  constructor(config) {
    this.config = config;
    this.pool = [];
    this.maxConnections = config.maxConnections || 10;
    this.activeConnections = 0;
    this.waitingQueue = [];
  }
  
  async getConnection() {
    if (this.pool.length > 0) {
      return this.pool.pop();
    }
    
    if (this.activeConnections < this.maxConnections) {
      this.activeConnections++;
      return await this.createConnection();
    }
    
    // Wait for available connection
    return new Promise((resolve) => {
      this.waitingQueue.push(resolve);
    });
  }
  
  releaseConnection(connection) {
    if (this.waitingQueue.length > 0) {
      const resolve = this.waitingQueue.shift();
      resolve(connection);
    } else {
      this.pool.push(connection);
    }
  }
  
  async createConnection() {
    // Mock connection creation
    return {
      execute: async (query, params) => {
        // Simulate database query execution
        await new Promise(resolve => setTimeout(resolve, 10));
        return { rows: [], rowCount: 0 };
      },
      beginTransaction: async () => {},
      commit: async () => {},
      rollback: async () => {}
    };
  }
}

// Usage in N8N workflow
const dbOptimizer = new DatabaseOptimizer({
  host: 'localhost',
  database: 'mydb',
  maxConnections: 10
});

const processedItems = [];

for (const item of $input.all()) {
  try {
    // Optimized single query
    const userData = await dbOptimizer.executeOptimizedQuery(
      'SELECT * FROM users WHERE id = ?',
      [item.json.userId],
      { cache: true, cacheTTL: 600000 } // 10 minutes cache
    );
    
    processedItems.push({
      json: {
        ...item.json,
        userData: userData.rows[0],
        fromCache: userData.fromCache
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

// Add database performance metrics
processedItems.push({
  json: {
    type: 'database_metrics',
    metrics: dbOptimizer.getMetrics()
  }
});

return processedItems;
```

## Memory Management

### Efficient Memory Usage Patterns
```javascript
// Memory-efficient data processing patterns
class MemoryOptimizer {
  constructor(options = {}) {
    this.options = {
      maxMemoryMB: options.maxMemoryMB || 512,
      chunkSize: options.chunkSize || 1000,
      gcThreshold: options.gcThreshold || 0.8,
      ...options
    };
    
    this.memoryUsage = new Map();
    this.gcStats = {
      collections: 0,
      totalTime: 0,
      lastCollection: null
    };
  }
  
  async processLargeDataset(data, processor, options = {}) {
    const chunkSize = options.chunkSize || this.options.chunkSize;
    const results = [];
    
    console.log(`Processing ${data.length} items in chunks of ${chunkSize}`);
    
    for (let i = 0; i < data.length; i += chunkSize) {
      const chunk = data.slice(i, i + chunkSize);
      
      try {
        // Monitor memory before processing chunk
        const memoryBefore = this.getMemoryUsage();
        
        // Process chunk
        const chunkResults = await this.processChunk(chunk, processor);
        results.push(...chunkResults);
        
        // Monitor memory after processing
        const memoryAfter = this.getMemoryUsage();
        this.logMemoryUsage(i / chunkSize, memoryBefore, memoryAfter);
        
        // Trigger garbage collection if memory usage is high
        if (this.shouldTriggerGC(memoryAfter)) {
          await this.triggerGC();
        }
        
        // Allow event loop to process other tasks
        await this.yield();
        
      } catch (error) {
        console.error(`Error processing chunk ${i / chunkSize}:`, error);
        
        // Add error info for failed chunk
        results.push({
          error: error.message,
          chunkIndex: Math.floor(i / chunkSize),
          chunkSize: chunk.length
        });
      }
    }
    
    return {
      results: results,
      processedItems: data.length,
      chunksProcessed: Math.ceil(data.length / chunkSize),
      memoryStats: this.getMemoryStats()
    };
  }
  
  async processChunk(chunk, processor) {
    const results = [];
    
    for (const item of chunk) {
      try {
        const result = await processor(item);
        results.push(result);
      } catch (error) {
        results.push({
          error: error.message,
          originalItem: item
        });
      }
    }
    
    return results;
  }
  
  async streamProcess(dataSource, processor, options = {}) {
    const results = [];
    let processedCount = 0;
    
    for await (const item of dataSource) {
      try {
        const result = await processor(item);
        results.push(result);
        processedCount++;
        
        // Periodically check memory and yield
        if (processedCount % 100 === 0) {
          const memoryUsage = this.getMemoryUsage();
          
          if (this.shouldTriggerGC(memoryUsage)) {
            await this.triggerGC();
          }
          
          await this.yield();
        }
        
      } catch (error) {
        results.push({
          error: error.message,
          originalItem: item
        });
      }
    }
    
    return {
      results: results,
      processedItems: processedCount,
      memoryStats: this.getMemoryStats()
    };
  }
  
  getMemoryUsage() {
    // In Node.js environment, you could use process.memoryUsage()
    // For N8N, we'll simulate memory monitoring
    return {
      used: Math.random() * this.options.maxMemoryMB, // Simulated
      free: this.options.maxMemoryMB - (Math.random() * this.options.maxMemoryMB),
      timestamp: Date.now()
    };
  }
  
  shouldTriggerGC(memoryUsage) {
    const usagePercentage = memoryUsage.used / this.options.maxMemoryMB;
    return usagePercentage > this.options.gcThreshold;
  }
  
  async triggerGC() {
    const gcStart = Date.now();
    
    // In real environment, you might trigger garbage collection
    // For simulation, we'll just add a small delay
    await new Promise(resolve => setTimeout(resolve, 10));
    
    const gcTime = Date.now() - gcStart;
    
    this.gcStats.collections++;
    this.gcStats.totalTime += gcTime;
    this.gcStats.lastCollection = new Date().toISOString();
    
    console.log(`Garbage collection completed in ${gcTime}ms`);
  }
  
  async yield() {
    // Yield control to allow other operations
    return new Promise(resolve => setImmediate(resolve));
  }
  
  logMemoryUsage(chunkIndex, before, after) {
    const usedDiff = after.used - before.used;
    
    console.log(`Chunk ${chunkIndex}: Memory ${before.used.toFixed(1)}MB → ${after.used.toFixed(1)}MB (${usedDiff > 0 ? '+' : ''}${usedDiff.toFixed(1)}MB)`);
  }
  
  getMemoryStats() {
    return {
      maxMemoryMB: this.options.maxMemoryMB,
      gcStats: this.gcStats,
      averageGCTime: this.gcStats.collections > 0 
        ? this.gcStats.totalTime / this.gcStats.collections 
        : 0
    };
  }
  
  // Memory-efficient object processing
  createObjectProcessor(schema) {
    return {
      process: (obj) => {
        const result = {};
        
        // Only include fields defined in schema to reduce memory usage
        for (const field of schema.fields) {
          if (obj.hasOwnProperty(field.name)) {
            result[field.name] = this.processField(obj[field.name], field);
          }
        }
        
        return result;
      },
      
      processField: (value, fieldConfig) => {
        // Apply field-specific optimizations
        if (fieldConfig.type === 'string' && fieldConfig.maxLength) {
          return value.substring(0, fieldConfig.maxLength);
        }
        
        if (fieldConfig.type === 'array' && fieldConfig.maxItems) {
          return Array.isArray(value) ? value.slice(0, fieldConfig.maxItems) : value;
        }
        
        return value;
      }
    };
  }
}

// Usage in memory-intensive N8N workflow
const memoryOptimizer = new MemoryOptimizer({
  maxMemoryMB: 256,
  chunkSize: 500,
  gcThreshold: 0.8
});

// Define data processing schema to optimize memory usage
const processingSchema = {
  fields: [
    { name: 'id', type: 'string' },
    { name: 'email', type: 'string', maxLength: 100 },
    { name: 'data', type: 'object' },
    { name: 'tags', type: 'array', maxItems: 10 }
  ]
};

const processor = memoryOptimizer.createObjectProcessor(processingSchema);

// Process large dataset efficiently
const largeDataset = $input.all(); // Potentially thousands of items

const processingResult = await memoryOptimizer.processLargeDataset(
  largeDataset.map(item => item.json),
  async (item) => {
    // Memory-efficient processing
    const optimizedItem = processor.process(item);
    
    // Simulate some processing
    await new Promise(resolve => setTimeout(resolve, 1));
    
    return {
      ...optimizedItem,
      processed: true,
      timestamp: Date.now()
    };
  },
  { chunkSize: 500 }
);

return [{
  json: {
    totalProcessed: processingResult.processedItems,
    chunksProcessed: processingResult.chunksProcessed,
    memoryStats: processingResult.memoryStats,
    results: processingResult.results
  }
}];
```

## Caching Strategies

### Multi-Level Caching System
```javascript
// Advanced caching system for N8N workflows
class MultiLevelCache {
  constructor(config = {}) {
    this.config = {
      l1Size: config.l1Size || 100,        // In-memory cache size
      l2Size: config.l2Size || 1000,       // Persistent cache size
      l1TTL: config.l1TTL || 300000,       // 5 minutes
      l2TTL: config.l2TTL || 3600000,      // 1 hour
      ...config
    };
    
    // Level 1: Fast in-memory cache
    this.l1Cache = new Map();
    this.l1AccessTimes = new Map();
    
    // Level 2: Persistent cache (simulated with Map)
    this.l2Cache = new Map();
    this.l2AccessTimes = new Map();
    
    // Cache statistics
    this.stats = {
      l1Hits: 0,
      l2Hits: 0,
      misses: 0,
      evictions: 0,
      writes: 0
    };
  }
  
  async get(key) {
    const now = Date.now();
    
    // Check L1 cache first
    if (this.l1Cache.has(key)) {
      const item = this.l1Cache.get(key);
      const accessTime = this.l1AccessTimes.get(key);
      
      if (now - accessTime < this.config.l1TTL) {
        this.l1AccessTimes.set(key, now);
        this.stats.l1Hits++;
        return item;
      } else {
        // L1 expired, remove from L1
        this.l1Cache.delete(key);
        this.l1AccessTimes.delete(key);
      }
    }
    
    // Check L2 cache
    if (this.l2Cache.has(key)) {
      const item = this.l2Cache.get(key);
      const accessTime = this.l2AccessTimes.get(key);
      
      if (now - accessTime < this.config.l2TTL) {
        // Promote to L1
        this.promoteToL1(key, item);
        this.l2AccessTimes.set(key, now);
        this.stats.l2Hits++;
        return item;
      } else {
        // L2 expired, remove from L2
        this.l2Cache.delete(key);
        this.l2AccessTimes.delete(key);
      }
    }
    
    // Cache miss
    this.stats.misses++;
    return null;
  }
  
  async set(key, value) {
    const now = Date.now();
    
    // Always store in L1 first
    this.promoteToL1(key, value);
    
    // Also store in L2 for persistence
    this.ensureL2Space();
    this.l2Cache.set(key, value);
    this.l2AccessTimes.set(key, now);
    
    this.stats.writes++;
  }
  
  promoteToL1(key, value) {
    const now = Date.now();
    
    // Ensure L1 has space
    this.ensureL1Space();
    
    this.l1Cache.set(key, value);
    this.l1AccessTimes.set(key, now);
  }
  
  ensureL1Space() {
    if (this.l1Cache.size >= this.config.l1Size) {
      this.evictFromL1();
    }
  }
  
  ensureL2Space() {
    if (this.l2Cache.size >= this.config.l2Size) {
      this.evictFromL2();
    }
  }
  
  evictFromL1() {
    // LRU eviction for L1
    let oldestKey = null;
    let oldestTime = Infinity;
    
    for (const [key, time] of this.l1AccessTimes.entries()) {
      if (time < oldestTime) {
        oldestTime = time;
        oldestKey = key;
      }
    }
    
    if (oldestKey) {
      this.l1Cache.delete(oldestKey);
      this.l1AccessTimes.delete(oldestKey);
      this.stats.evictions++;
    }
  }
  
  evictFromL2() {
    // LRU eviction for L2
    let oldestKey = null;
    let oldestTime = Infinity;
    
    for (const [key, time] of this.l2AccessTimes.entries()) {
      if (time < oldestTime) {
        oldestTime = time;
        oldestKey = key;
      }
    }
    
    if (oldestKey) {
      this.l2Cache.delete(oldestKey);
      this.l2AccessTimes.delete(oldestKey);
    }
  }
  
  async getOrSet(key, factory, options = {}) {
    let value = await this.get(key);
    
    if (value === null) {
      value = await factory();
      if (value !== null && value !== undefined) {
        await this.set(key, value);
      }
    }
    
    return value;
  }
  
  generateKey(baseKey, params = {}) {
    const normalized = Object.keys(params)
      .sort()
      .map(k => `${k}=${params[k]}`)
      .join('&');
    
    return normalized ? `${baseKey}?${normalized}` : baseKey;
  }
  
  clear() {
    this.l1Cache.clear();
    this.l1AccessTimes.clear();
    this.l2Cache.clear();
    this.l2AccessTimes.clear();
  }
  
  getStats() {
    const totalRequests = this.stats.l1Hits + this.stats.l2Hits + this.stats.misses;
    
    return {
      ...this.stats,
      totalRequests,
      hitRate: totalRequests > 0 
        ? ((this.stats.l1Hits + this.stats.l2Hits) / totalRequests) * 100 
        : 0,
      l1HitRate: totalRequests > 0 
        ? (this.stats.l1Hits / totalRequests) * 100 
        : 0,
      l2HitRate: totalRequests > 0 
        ? (this.stats.l2Hits / totalRequests) * 100 
        : 0,
      l1Size: this.l1Cache.size,
      l2Size: this.l2Cache.size
    };
  }
}

// Cache-aware HTTP client
class CachedHTTPClient {
  constructor(cache, options = {}) {
    this.cache = cache;
    this.options = {
      defaultCacheTTL: options.defaultCacheTTL || 300000, // 5 minutes
      ...options
    };
  }
  
  async get(url, options = {}) {
    const cacheKey = this.cache.generateKey(`http:${url}`, {
      headers: JSON.stringify(options.headers || {}),
      params: JSON.stringify(options.params || {})
    });
    
    return await this.cache.getOrSet(
      cacheKey,
      async () => {
        console.log(`Cache miss for ${url}, fetching...`);
        
        const response = await fetch(url, {
          headers: options.headers,
          ...options
        });
        
        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        const data = await response.json();
        
        return {
          data,
          status: response.status,
          headers: Object.fromEntries(response.headers.entries()),
          cachedAt: new Date().toISOString()
        };
      }
    );
  }
  
  async post(url, data, options = {}) {
    // POST requests are typically not cached, but we can cache
    // based on the request signature if the API is idempotent
    if (options.cache && options.idempotent) {
      const cacheKey = this.cache.generateKey(`http:POST:${url}`, {
        body: JSON.stringify(data),
        headers: JSON.stringify(options.headers || {})
      });
      
      return await this.cache.getOrSet(cacheKey, async () => {
        return await this.executePost(url, data, options);
      });
    }
    
    return await this.executePost(url, data, options);
  }
  
  async executePost(url, data, options) {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      },
      body: JSON.stringify(data),
      ...options
    });
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    return {
      data: await response.json(),
      status: response.status,
      headers: Object.fromEntries(response.headers.entries())
    };
  }
}

// Usage in N8N workflow with caching
const cache = new MultiLevelCache({
  l1Size: 50,
  l2Size: 500,
  l1TTL: 300000,  // 5 minutes
  l2TTL: 1800000  // 30 minutes
});

const httpClient = new CachedHTTPClient(cache);

const results = [];

for (const item of $input.all()) {
  try {
    // This will use cache if available
    const apiResponse = await httpClient.get(
      `https://api.example.com/users/${item.json.userId}`,
      {
        headers: {
          'Authorization': `Bearer ${$credentials.token}`
        }
      }
    );
    
    results.push({
      json: {
        ...item.json,
        userData: apiResponse.data,
        fromCache: apiResponse.cachedAt ? true : false,
        cacheTimestamp: apiResponse.cachedAt
      }
    });
    
  } catch (error) {
    results.push({
      json: {
        ...item.json,
        error: error.message,
        failed: true
      }
    });
  }
}

// Add cache statistics
results.push({
  json: {
    type: 'cache_stats',
    stats: cache.getStats()
  }
});

return results;
```

## Practice Exercises

### Exercise 1: Workflow Performance Audit
Create a performance auditing system that:
1. Measures execution time for each node
2. Identifies memory usage patterns
3. Detects bottlenecks and optimization opportunities
4. Generates performance reports

### Exercise 2: Auto-Scaling System
Build an auto-scaling solution that:
1. Monitors workflow load and performance
2. Automatically adjusts concurrency limits
3. Implements circuit breakers for overload protection
4. Scales up/down based on demand

### Exercise 3: Performance Testing Framework
Develop a testing framework that:
1. Simulates different load scenarios
2. Measures performance under various conditions
3. Compares performance across workflow versions
4. Generates performance benchmarks

## Next Steps

- Master [Security Best Practices](./12-security-practices.md)
- Explore [Enterprise Patterns](./13-enterprise-patterns.md)
- Learn [Custom Node Development](./14-custom-nodes.md)

---

Performance optimization is crucial for scalable automation systems. Master these techniques to build workflows that perform efficiently at any scale!
