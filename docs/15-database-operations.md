# Database Operations in N8N

Master database integration and operations in N8N to build robust data-driven automation workflows with advanced querying, transaction management, and performance optimization.

## Table of Contents
- [Database Connection Strategies](#database-connection-strategies)
- [SQL Query Optimization](#sql-query-optimization)
- [Transaction Management](#transaction-management)
- [Connection Pooling](#connection-pooling)
- [Data Migration Workflows](#data-migration-workflows)
- [Advanced Query Patterns](#advanced-query-patterns)
- [Performance Monitoring](#performance-monitoring)
- [Database Security](#database-security)
- [Multi-Database Operations](#multi-database-operations)
- [Real-time Data Synchronization](#real-time-data-synchronization)

## Database Connection Strategies

### PostgreSQL Advanced Configuration
```javascript
// PostgreSQL with connection pooling and advanced features
{
  "nodes": [
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT * FROM users WHERE created_at > $1 AND status = $2",
        "options": {
          "queryParameters": [
            "2023-01-01",
            "active"
          ],
          "nodeFormat": "object",
          "connectionPool": {
            "min": 2,
            "max": 10,
            "idleTimeoutMillis": 30000,
            "connectionTimeoutMillis": 5000
          },
          "ssl": {
            "rejectUnauthorized": false,
            "ca": "{{$env.POSTGRES_SSL_CA}}",
            "cert": "{{$env.POSTGRES_SSL_CERT}}",
            "key": "{{$env.POSTGRES_SSL_KEY}}"
          },
          "retryOptions": {
            "retries": 3,
            "retryDelay": 1000,
            "retryDelayType": "exponential"
          }
        }
      },
      "name": "PostgreSQL Advanced",
      "type": "n8n-nodes-base.postgres",
      "position": [250, 300],
      "typeVersion": 2.4
    }
  ]
}

// Dynamic query builder with complex conditions
const queryBuilder = {
  buildUserQuery: (filters = {}) => {
    let query = `
      SELECT 
        u.id,
        u.username,
        u.email,
        u.created_at,
        p.name as profile_name,
        COUNT(o.id) as order_count,
        SUM(o.total_amount) as total_spent
      FROM users u
      LEFT JOIN profiles p ON u.id = p.user_id
      LEFT JOIN orders o ON u.id = o.user_id
      WHERE 1=1
    `;
    
    const params = [];
    let paramIndex = 1;
    
    if (filters.dateRange) {
      query += ` AND u.created_at BETWEEN $${paramIndex} AND $${paramIndex + 1}`;
      params.push(filters.dateRange.start, filters.dateRange.end);
      paramIndex += 2;
    }
    
    if (filters.status) {
      query += ` AND u.status = $${paramIndex}`;
      params.push(filters.status);
      paramIndex++;
    }
    
    if (filters.searchTerm) {
      query += ` AND (u.username ILIKE $${paramIndex} OR u.email ILIKE $${paramIndex})`;
      params.push(`%${filters.searchTerm}%`);
      paramIndex++;
    }
    
    query += `
      GROUP BY u.id, u.username, u.email, u.created_at, p.name
      ORDER BY u.created_at DESC
    `;
    
    if (filters.limit) {
      query += ` LIMIT $${paramIndex}`;
      params.push(filters.limit);
      paramIndex++;
    }
    
    if (filters.offset) {
      query += ` OFFSET $${paramIndex}`;
      params.push(filters.offset);
    }
    
    return { query, params };
  }
};

// Usage in N8N node
const filters = {
  dateRange: {
    start: '2023-01-01',
    end: '2023-12-31'
  },
  status: 'active',
  searchTerm: 'john',
  limit: 50,
  offset: 0
};

const { query, params } = queryBuilder.buildUserQuery(filters);

return [{
  json: {
    query: query,
    parameters: params,
    generatedAt: new Date().toISOString()
  }
}];
```

### MySQL Optimization Patterns
```javascript
// MySQL with optimization and monitoring
{
  "nodes": [
    {
      "parameters": {
        "operation": "executeQuery",
        "query": `
          SELECT /*+ USE_INDEX(users, idx_users_status_created) */
            u.id,
            u.username,
            u.email,
            u.last_login,
            CASE 
              WHEN u.last_login > DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 'active'
              WHEN u.last_login > DATE_SUB(NOW(), INTERVAL 90 DAY) THEN 'inactive'
              ELSE 'dormant'
            END as activity_status,
            (
              SELECT COUNT(*) 
              FROM user_sessions s 
              WHERE s.user_id = u.id 
                AND s.created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
            ) as recent_sessions
          FROM users u
          WHERE u.status = ?
            AND u.created_at > ?
          ORDER BY u.last_login DESC
          LIMIT ?
        `,
        "options": {
          "queryParameters": [
            "active",
            "2023-01-01",
            100
          ],
          "queryTimeout": 30000,
          "useIndex": true,
          "explainPlan": true
        }
      },
      "name": "MySQL Optimized Query",
      "type": "n8n-nodes-base.mysql",
      "position": [250, 300],
      "typeVersion": 2.1
    }
  ]
}

// Query performance analyzer
const queryAnalyzer = {
  analyzeQuery: async (query, params = []) => {
    // Get execution plan
    const explainQuery = `EXPLAIN FORMAT=JSON ${query}`;
    const explainResult = await mysql.query(explainQuery, params);
    
    // Analyze execution plan
    const analysis = {
      estimatedRows: 0,
      indexUsage: [],
      warnings: [],
      optimizationSuggestions: []
    };
    
    const queryPlan = JSON.parse(explainResult[0]['EXPLAIN']);
    
    const analyzeNode = (node) => {
      if (node.table) {
        analysis.estimatedRows += node.rows_examined_per_scan || 0;
        
        if (node.possible_keys && node.key) {
          analysis.indexUsage.push({
            table: node.table.table_name,
            index: node.key,
            type: node.access_type
          });
        } else if (node.access_type === 'ALL') {
          analysis.warnings.push(`Full table scan on ${node.table.table_name}`);
          analysis.optimizationSuggestions.push(
            `Consider adding index on ${node.table.table_name}`
          );
        }
      }
      
      if (node.nested_loop && node.nested_loop.length > 0) {
        node.nested_loop.forEach(analyzeNode);
      }
    };
    
    queryPlan.query_block && analyzeNode(queryPlan.query_block);
    
    return analysis;
  },
  
  optimizeQuery: (originalQuery, analysis) => {
    let optimizedQuery = originalQuery;
    
    // Add query hints based on analysis
    if (analysis.warnings.some(w => w.includes('Full table scan'))) {
      optimizedQuery = optimizedQuery.replace(
        'SELECT',
        'SELECT /*+ USE_INDEX_MERGE(users) */'
      );
    }
    
    // Add LIMIT if not present and result set is large
    if (analysis.estimatedRows > 1000 && !originalQuery.includes('LIMIT')) {
      optimizedQuery += ' LIMIT 1000';
    }
    
    return optimizedQuery;
  }
};
```

### MongoDB Aggregation Pipelines
```javascript
// Advanced MongoDB aggregation with optimization
{
  "nodes": [
    {
      "parameters": {
        "operation": "aggregate",
        "collection": "orders",
        "pipeline": [
          {
            "$match": {
              "created_at": {
                "$gte": "{{new Date('2023-01-01')}}",
                "$lte": "{{new Date('2023-12-31')}}"
              },
              "status": { "$in": ["completed", "shipped"] }
            }
          },
          {
            "$lookup": {
              "from": "users",
              "localField": "user_id",
              "foreignField": "_id",
              "as": "user",
              "pipeline": [
                {
                  "$project": {
                    "username": 1,
                    "email": 1,
                    "tier": 1
                  }
                }
              ]
            }
          },
          {
            "$unwind": "$user"
          },
          {
            "$lookup": {
              "from": "products",
              "localField": "items.product_id",
              "foreignField": "_id",
              "as": "product_details"
            }
          },
          {
            "$addFields": {
              "order_value": {
                "$sum": {
                  "$map": {
                    "input": "$items",
                    "as": "item",
                    "in": {
                      "$multiply": ["$$item.quantity", "$$item.price"]
                    }
                  }
                }
              },
              "item_count": { "$size": "$items" }
            }
          },
          {
            "$group": {
              "_id": {
                "user_id": "$user_id",
                "user_tier": "$user.tier",
                "month": {
                  "$dateToString": {
                    "format": "%Y-%m",
                    "date": "$created_at"
                  }
                }
              },
              "total_orders": { "$sum": 1 },
              "total_value": { "$sum": "$order_value" },
              "total_items": { "$sum": "$item_count" },
              "avg_order_value": { "$avg": "$order_value" },
              "user_info": { "$first": "$user" },
              "orders": {
                "$push": {
                  "order_id": "$_id",
                  "value": "$order_value",
                  "items": "$item_count",
                  "date": "$created_at"
                }
              }
            }
          },
          {
            "$project": {
              "user_id": "$_id.user_id",
              "user_tier": "$_id.user_tier",
              "month": "$_id.month",
              "user_info": 1,
              "statistics": {
                "total_orders": "$total_orders",
                "total_value": "$total_value",
                "total_items": "$total_items",
                "avg_order_value": "$avg_order_value"
              },
              "orders": 1,
              "_id": 0
            }
          },
          {
            "$sort": {
              "month": 1,
              "statistics.total_value": -1
            }
          }
        ],
        "options": {
          "allowDiskUse": true,
          "maxTimeMS": 30000,
          "hint": { "user_id": 1, "created_at": 1 },
          "collation": {
            "locale": "en",
            "strength": 2
          }
        }
      },
      "name": "MongoDB Advanced Aggregation",
      "type": "n8n-nodes-base.mongodb",
      "position": [250, 300],
      "typeVersion": 1.0
    }
  ]
}

// Dynamic aggregation pipeline builder
const pipelineBuilder = {
  buildAnalyticsPipeline: (options = {}) => {
    const pipeline = [];
    
    // Base match stage
    const matchStage = {
      "$match": {}
    };
    
    if (options.dateRange) {
      matchStage["$match"]["created_at"] = {
        "$gte": new Date(options.dateRange.start),
        "$lte": new Date(options.dateRange.end)
      };
    }
    
    if (options.status) {
      matchStage["$match"]["status"] = Array.isArray(options.status) 
        ? { "$in": options.status }
        : options.status;
    }
    
    pipeline.push(matchStage);
    
    // User lookup with projection
    if (options.includeUserDetails) {
      pipeline.push({
        "$lookup": {
          "from": "users",
          "localField": "user_id",
          "foreignField": "_id",
          "as": "user",
          "pipeline": [
            {
              "$project": options.userProjection || {
                "username": 1,
                "email": 1,
                "tier": 1,
                "created_at": 1
              }
            }
          ]
        }
      });
      
      pipeline.push({ "$unwind": "$user" });
    }
    
    // Product details lookup
    if (options.includeProductDetails) {
      pipeline.push({
        "$lookup": {
          "from": "products",
          "localField": "items.product_id",
          "foreignField": "_id",
          "as": "product_details"
        }
      });
    }
    
    // Add calculated fields
    pipeline.push({
      "$addFields": {
        "order_value": {
          "$sum": {
            "$map": {
              "input": "$items",
              "as": "item",
              "in": {
                "$multiply": ["$$item.quantity", "$$item.price"]
              }
            }
          }
        },
        "item_count": { "$size": "$items" },
        "order_month": {
          "$dateToString": {
            "format": "%Y-%m",
            "date": "$created_at"
          }
        }
      }
    });
    
    // Grouping stage
    if (options.groupBy) {
      const groupStage = {
        "$group": {
          "_id": {},
          "count": { "$sum": 1 },
          "total_value": { "$sum": "$order_value" },
          "avg_value": { "$avg": "$order_value" }
        }
      };
      
      // Build group ID
      options.groupBy.forEach(field => {
        if (field === 'month') {
          groupStage["$group"]["_id"]["month"] = "$order_month";
        } else if (field === 'user_tier') {
          groupStage["$group"]["_id"]["user_tier"] = "$user.tier";
        } else {
          groupStage["$group"]["_id"][field] = `$${field}`;
        }
      });
      
      // Add additional aggregations
      if (options.aggregations) {
        Object.assign(groupStage["$group"], options.aggregations);
      }
      
      pipeline.push(groupStage);
    }
    
    // Sorting
    if (options.sort) {
      pipeline.push({ "$sort": options.sort });
    }
    
    // Limit
    if (options.limit) {
      pipeline.push({ "$limit": options.limit });
    }
    
    return pipeline;
  }
};

// Usage example
const analyticsOptions = {
  dateRange: {
    start: '2023-01-01',
    end: '2023-12-31'
  },
  status: ['completed', 'shipped'],
  includeUserDetails: true,
  includeProductDetails: true,
  groupBy: ['month', 'user_tier'],
  aggregations: {
    "unique_users": { "$addToSet": "$user_id" },
    "top_products": {
      "$push": {
        "$arrayElemAt": ["$product_details.name", 0]
      }
    }
  },
  sort: { "_id.month": 1, "total_value": -1 },
  limit: 100
};

const pipeline = pipelineBuilder.buildAnalyticsPipeline(analyticsOptions);
```

## SQL Query Optimization

### Index Strategy Implementation
```javascript
// Index analysis and optimization workflow
{
  "nodes": [
    {
      "parameters": {
        "jsCode": `
          // Index analysis utility
          const indexAnalyzer = {
            analyzeTableIndexes: async (tableName) => {
              // Get current indexes
              const indexQuery = \`
                SELECT 
                  i.indexname,
                  i.indexdef,
                  s.idx_scan as usage_count,
                  s.idx_tup_read as tuples_read,
                  s.idx_tup_fetch as tuples_fetched,
                  pg_size_pretty(pg_relation_size(i.indexname::regclass)) as size
                FROM pg_indexes i
                LEFT JOIN pg_stat_user_indexes s ON i.indexname = s.indexname
                WHERE i.tablename = '\${tableName}'
                ORDER BY s.idx_scan DESC NULLS LAST
              \`;
              
              const indexes = await postgres.query(indexQuery);
              
              // Analyze query patterns
              const queryPatterns = await this.analyzeQueryPatterns(tableName);
              
              // Generate recommendations
              const recommendations = this.generateIndexRecommendations(
                indexes, 
                queryPatterns
              );
              
              return {
                currentIndexes: indexes,
                queryPatterns: queryPatterns,
                recommendations: recommendations
              };
            },
            
            analyzeQueryPatterns: async (tableName) => {
              // Analyze slow query log for patterns
              const slowQueries = await postgres.query(\`
                SELECT 
                  query,
                  calls,
                  total_time,
                  mean_time,
                  rows,
                  100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
                FROM pg_stat_statements 
                WHERE query LIKE '%\${tableName}%'
                ORDER BY total_time DESC
                LIMIT 20
              \`);
              
              const patterns = slowQueries.map(query => {
                return {
                  query: query.query,
                  performance: {
                    calls: query.calls,
                    avgTime: query.mean_time,
                    totalTime: query.total_time,
                    hitPercent: query.hit_percent
                  },
                  suggestedIndexes: this.extractIndexSuggestions(query.query)
                };
              });
              
              return patterns;
            },
            
            extractIndexSuggestions: (query) => {
              const suggestions = [];
              
              // Extract WHERE clauses
              const whereMatches = query.match(/WHERE\\s+([^ORDER|GROUP|LIMIT]+)/gi);
              if (whereMatches) {
                whereMatches.forEach(whereClause => {
                  // Extract column names from conditions
                  const columnMatches = whereClause.match(/\\b([a-zA-Z_][a-zA-Z0-9_]*)\\s*[=<>]/g);
                  if (columnMatches) {
                    columnMatches.forEach(match => {
                      const column = match.replace(/[=<>\\s]/g, '');
                      suggestions.push({
                        type: 'single_column',
                        columns: [column],
                        reason: 'WHERE clause condition'
                      });
                    });
                  }
                });
              }
              
              // Extract ORDER BY clauses
              const orderMatches = query.match(/ORDER\\s+BY\\s+([^LIMIT]+)/gi);
              if (orderMatches) {
                orderMatches.forEach(orderClause => {
                  const columns = orderClause
                    .replace(/ORDER\\s+BY\\s+/gi, '')
                    .split(',')
                    .map(col => col.trim().split(' ')[0]);
                  
                  suggestions.push({
                    type: 'composite',
                    columns: columns,
                    reason: 'ORDER BY optimization'
                  });
                });
              }
              
              // Extract JOIN conditions
              const joinMatches = query.match(/JOIN\\s+\\w+\\s+ON\\s+([^WHERE|ORDER|GROUP]+)/gi);
              if (joinMatches) {
                joinMatches.forEach(joinClause => {
                  const columns = joinClause.match(/\\b([a-zA-Z_][a-zA-Z0-9_]*\\.[a-zA-Z_][a-zA-Z0-9_]*)\\b/g);
                  if (columns) {
                    columns.forEach(column => {
                      const cleanColumn = column.split('.')[1];
                      suggestions.push({
                        type: 'foreign_key',
                        columns: [cleanColumn],
                        reason: 'JOIN condition'
                      });
                    });
                  }
                });
              }
              
              return suggestions;
            },
            
            generateIndexRecommendations: (currentIndexes, queryPatterns) => {
              const recommendations = [];
              
              // Find unused indexes
              const unusedIndexes = currentIndexes.filter(idx => 
                idx.usage_count === 0 || idx.usage_count < 10
              );
              
              unusedIndexes.forEach(idx => {
                recommendations.push({
                  type: 'DROP',
                  index: idx.indexname,
                  reason: \`Low usage: \${idx.usage_count} scans\`,
                  priority: 'HIGH',
                  impact: \`Saves \${idx.size} storage\`
                });
              });
              
              // Collect all suggested indexes from query patterns
              const suggestedIndexes = new Map();
              
              queryPatterns.forEach(pattern => {
                pattern.suggestedIndexes.forEach(suggestion => {
                  const key = suggestion.columns.join(',');
                  if (!suggestedIndexes.has(key)) {
                    suggestedIndexes.set(key, {
                      columns: suggestion.columns,
                      type: suggestion.type,
                      reasons: [],
                      totalTime: 0,
                      callCount: 0
                    });
                  }
                  
                  const existing = suggestedIndexes.get(key);
                  existing.reasons.push(suggestion.reason);
                  existing.totalTime += pattern.performance.totalTime;
                  existing.callCount += pattern.performance.calls;
                });
              });
              
              // Check if suggested indexes already exist
              suggestedIndexes.forEach((suggestion, key) => {
                const existingIndex = currentIndexes.find(idx => {
                  const indexColumns = this.extractIndexColumns(idx.indexdef);
                  return indexColumns.join(',') === key;
                });
                
                if (!existingIndex) {
                  recommendations.push({
                    type: 'CREATE',
                    columns: suggestion.columns,
                    indexType: this.determineIndexType(suggestion),
                    reason: \`Optimize queries: \${suggestion.reasons.join(', ')}\`,
                    priority: this.calculatePriority(suggestion),
                    estimatedImprovement: \`\${Math.round(suggestion.totalTime / 1000)}s saved\`,
                    sql: this.generateCreateIndexSQL(suggestion.columns, suggestion.type)
                  });
                }
              });
              
              return recommendations;
            },
            
            extractIndexColumns: (indexDef) => {
              const match = indexDef.match(/\\(([^)]+)\\)/);
              if (match) {
                return match[1].split(',').map(col => col.trim());
              }
              return [];
            },
            
            determineIndexType: (suggestion) => {
              if (suggestion.type === 'foreign_key') return 'BTREE';
              if (suggestion.columns.length > 1) return 'COMPOSITE';
              return 'BTREE';
            },
            
            calculatePriority: (suggestion) => {
              if (suggestion.totalTime > 10000) return 'HIGH';
              if (suggestion.totalTime > 1000) return 'MEDIUM';
              return 'LOW';
            },
            
            generateCreateIndexSQL: (columns, type) => {
              const indexName = \`idx_\${columns.join('_')}_\${Date.now()}\`;
              return \`CREATE INDEX CONCURRENTLY \${indexName} ON table_name (\${columns.join(', ')});\`;
            }
          };
          
          // Analyze current table
          const tableName = $input.first().json.tableName || 'users';
          const analysis = await indexAnalyzer.analyzeTableIndexes(tableName);
          
          return [{
            json: {
              tableName: tableName,
              analysis: analysis,
              timestamp: new Date().toISOString()
            }
          }];
        `
      },
      "name": "Index Analyzer",
      "type": "n8n-nodes-base.code",
      "position": [250, 300],
      "typeVersion": 2
    }
  ]
}

// Query plan analyzer for complex queries
const queryPlanAnalyzer = {
  analyzeExecutionPlan: async (query, parameters = []) => {
    // Get execution plan with timing
    const explainQuery = `
      EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
      ${query}
    `;
    
    const result = await postgres.query(explainQuery, parameters);
    const plan = result[0]['QUERY PLAN'][0];
    
    const analysis = {
      totalCost: plan.Plan['Total Cost'],
      actualTime: plan.Plan['Actual Total Time'],
      planningTime: plan['Planning Time'],
      executionTime: plan['Execution Time'],
      bufferHits: plan.Plan['Shared Hit Blocks'] || 0,
      bufferReads: plan.Plan['Shared Read Blocks'] || 0,
      bufferDirtied: plan.Plan['Shared Dirtied Blocks'] || 0,
      operations: [],
      bottlenecks: [],
      recommendations: []
    };
    
    // Recursively analyze plan nodes
    const analyzePlanNode = (node, depth = 0) => {
      const operation = {
        nodeType: node['Node Type'],
        relationName: node['Relation Name'],
        actualRows: node['Actual Rows'],
        plannedRows: node['Plan Rows'],
        actualTime: node['Actual Total Time'],
        plannedCost: node['Total Cost'],
        bufferHits: node['Shared Hit Blocks'] || 0,
        bufferReads: node['Shared Read Blocks'] || 0,
        depth: depth
      };
      
      analysis.operations.push(operation);
      
      // Identify bottlenecks
      if (operation.actualTime > 100) { // More than 100ms
        analysis.bottlenecks.push({
          operation: operation.nodeType,
          table: operation.relationName,
          actualTime: operation.actualTime,
          issue: 'High execution time'
        });
      }
      
      if (operation.actualRows / operation.plannedRows > 10) {
        analysis.bottlenecks.push({
          operation: operation.nodeType,
          table: operation.relationName,
          plannedRows: operation.plannedRows,
          actualRows: operation.actualRows,
          issue: 'Poor cardinality estimation'
        });
      }
      
      if (operation.bufferReads > operation.bufferHits) {
        analysis.bottlenecks.push({
          operation: operation.nodeType,
          table: operation.relationName,
          bufferReads: operation.bufferReads,
          bufferHits: operation.bufferHits,
          issue: 'High disk I/O'
        });
      }
      
      // Analyze child nodes
      if (node.Plans) {
        node.Plans.forEach(childNode => analyzePlanNode(childNode, depth + 1));
      }
    };
    
    analyzePlanNode(plan.Plan);
    
    // Generate recommendations
    analysis.recommendations = this.generateOptimizationRecommendations(analysis);
    
    return analysis;
  },
  
  generateOptimizationRecommendations: (analysis) => {
    const recommendations = [];
    
    // Check for sequential scans on large tables
    const sequentialScans = analysis.operations.filter(op => 
      op.nodeType === 'Seq Scan' && op.actualRows > 1000
    );
    
    sequentialScans.forEach(scan => {
      recommendations.push({
        type: 'INDEX',
        priority: 'HIGH',
        message: `Add index on ${scan.relationName} to avoid sequential scan`,
        estimatedImprovement: `${Math.round((scan.actualTime * 0.8))}ms`
      });
    });
    
    // Check for nested loop joins with high cost
    const expensiveNestedLoops = analysis.operations.filter(op => 
      op.nodeType === 'Nested Loop' && op.actualTime > 50
    );
    
    expensiveNestedLoops.forEach(loop => {
      recommendations.push({
        type: 'JOIN_OPTIMIZATION',
        priority: 'MEDIUM',
        message: 'Consider hash join or merge join instead of nested loop',
        suggestion: 'Add indexes on join columns or increase work_mem'
      });
    });
    
    // Check for high buffer reads
    if (analysis.bufferReads > analysis.bufferHits * 2) {
      recommendations.push({
        type: 'MEMORY',
        priority: 'HIGH',
        message: 'High disk I/O detected',
        suggestion: 'Increase shared_buffers or add covering indexes'
      });
    }
    
    // Check for poor cardinality estimation
    const poorEstimations = analysis.bottlenecks.filter(b => 
      b.issue === 'Poor cardinality estimation'
    );
    
    if (poorEstimations.length > 0) {
      recommendations.push({
        type: 'STATISTICS',
        priority: 'MEDIUM',
        message: 'Update table statistics',
        suggestion: 'Run ANALYZE on affected tables'
      });
    }
    
    return recommendations;
  }
};
```

## Transaction Management

### ACID Transaction Patterns
```javascript
// Advanced transaction management with savepoints
{
  "nodes": [
    {
      "parameters": {
        "jsCode": `
          // Transaction manager with sophisticated error handling
          class TransactionManager {
            constructor(connection) {
              this.connection = connection;
              this.savepoints = [];
              this.transactionStartTime = null;
              this.operations = [];
            }
            
            async beginTransaction(isolationLevel = 'READ COMMITTED') {
              try {
                await this.connection.query('BEGIN');
                await this.connection.query(\`SET TRANSACTION ISOLATION LEVEL \${isolationLevel}\`);
                this.transactionStartTime = Date.now();
                
                console.log(\`Transaction started with isolation level: \${isolationLevel}\`);
                return true;
              } catch (error) {
                console.error('Failed to begin transaction:', error);
                throw error;
              }
            }
            
            async createSavepoint(name) {
              try {
                const savepointName = \`sp_\${name}_\${Date.now()}\`;
                await this.connection.query(\`SAVEPOINT \${savepointName}\`);
                this.savepoints.push({
                  name: savepointName,
                  createdAt: Date.now(),
                  operations: [...this.operations]
                });
                
                console.log(\`Savepoint created: \${savepointName}\`);
                return savepointName;
              } catch (error) {
                console.error('Failed to create savepoint:', error);
                throw error;
              }
            }
            
            async rollbackToSavepoint(savepointName) {
              try {
                await this.connection.query(\`ROLLBACK TO SAVEPOINT \${savepointName}\`);
                
                // Remove operations after savepoint
                const savepoint = this.savepoints.find(sp => sp.name === savepointName);
                if (savepoint) {
                  this.operations = savepoint.operations;
                }
                
                console.log(\`Rolled back to savepoint: \${savepointName}\`);
                return true;
              } catch (error) {
                console.error('Failed to rollback to savepoint:', error);
                throw error;
              }
            }
            
            async executeOperation(operation) {
              const startTime = Date.now();
              
              try {
                const result = await this.connection.query(operation.query, operation.params);
                
                const executionTime = Date.now() - startTime;
                const operationLog = {
                  id: this.operations.length + 1,
                  query: operation.query,
                  params: operation.params,
                  executionTime: executionTime,
                  rowsAffected: result.rowCount || result.length,
                  timestamp: new Date().toISOString(),
                  status: 'SUCCESS'
                };
                
                this.operations.push(operationLog);
                console.log(\`Operation completed in \${executionTime}ms\`);
                
                return {
                  success: true,
                  result: result,
                  operation: operationLog
                };
                
              } catch (error) {
                const executionTime = Date.now() - startTime;
                const operationLog = {
                  id: this.operations.length + 1,
                  query: operation.query,
                  params: operation.params,
                  executionTime: executionTime,
                  error: error.message,
                  timestamp: new Date().toISOString(),
                  status: 'ERROR'
                };
                
                this.operations.push(operationLog);
                console.error(\`Operation failed after \${executionTime}ms:`, error);
                
                throw {
                  error: error,
                  operation: operationLog
                };
              }
            }
            
            async commitTransaction() {
              try {
                await this.connection.query('COMMIT');
                
                const totalTime = Date.now() - this.transactionStartTime;
                const summary = {
                  success: true,
                  totalOperations: this.operations.length,
                  successfulOperations: this.operations.filter(op => op.status === 'SUCCESS').length,
                  failedOperations: this.operations.filter(op => op.status === 'ERROR').length,
                  totalExecutionTime: totalTime,
                  operations: this.operations,
                  committedAt: new Date().toISOString()
                };
                
                console.log(\`Transaction committed successfully in \${totalTime}ms\`);
                this.reset();
                
                return summary;
              } catch (error) {
                console.error('Failed to commit transaction:', error);
                await this.rollbackTransaction();
                throw error;
              }
            }
            
            async rollbackTransaction() {
              try {
                await this.connection.query('ROLLBACK');
                
                const totalTime = Date.now() - this.transactionStartTime;
                const summary = {
                  success: false,
                  totalOperations: this.operations.length,
                  successfulOperations: this.operations.filter(op => op.status === 'SUCCESS').length,
                  failedOperations: this.operations.filter(op => op.status === 'ERROR').length,
                  totalExecutionTime: totalTime,
                  operations: this.operations,
                  rolledBackAt: new Date().toISOString()
                };
                
                console.log(\`Transaction rolled back after \${totalTime}ms\`);
                this.reset();
                
                return summary;
              } catch (error) {
                console.error('Failed to rollback transaction:', error);
                this.reset();
                throw error;
              }
            }
            
            reset() {
              this.savepoints = [];
              this.operations = [];
              this.transactionStartTime = null;
            }
            
            getTransactionStatus() {
              return {
                isActive: this.transactionStartTime !== null,
                startTime: this.transactionStartTime,
                operationCount: this.operations.length,
                savepointCount: this.savepoints.length,
                elapsedTime: this.transactionStartTime ? Date.now() - this.transactionStartTime : 0
              };
            }
          }
          
          // Usage example: Complex multi-step transaction
          const txManager = new TransactionManager(postgres);
          
          try {
            // Begin transaction with serializable isolation
            await txManager.beginTransaction('SERIALIZABLE');
            
            // Step 1: Create user
            const createUserSavepoint = await txManager.createSavepoint('create_user');
            const userResult = await txManager.executeOperation({
              query: 'INSERT INTO users (username, email, status) VALUES ($1, $2, $3) RETURNING id',
              params: ['john_doe', 'john@example.com', 'active']
            });
            
            const userId = userResult.result[0].id;
            
            // Step 2: Create user profile
            try {
              await txManager.executeOperation({
                query: 'INSERT INTO profiles (user_id, first_name, last_name) VALUES ($1, $2, $3)',
                params: [userId, 'John', 'Doe']
              });
            } catch (error) {
              // Rollback to savepoint and try alternative approach
              await txManager.rollbackToSavepoint(createUserSavepoint);
              
              // Alternative: Create user with basic profile
              const userWithProfileResult = await txManager.executeOperation({
                query: \`
                  WITH new_user AS (
                    INSERT INTO users (username, email, status) VALUES ($1, $2, $3) RETURNING id
                  )
                  INSERT INTO profiles (user_id, first_name, last_name)
                  SELECT id, $4, $5 FROM new_user
                  RETURNING user_id
                \`,
                params: ['john_doe', 'john@example.com', 'active', 'John', 'Doe']
              });
              
              userId = userWithProfileResult.result[0].user_id;
            }
            
            // Step 3: Create initial orders
            const ordersSavepoint = await txManager.createSavepoint('create_orders');
            const orders = [
              { product_id: 1, quantity: 2, price: 29.99 },
              { product_id: 2, quantity: 1, price: 59.99 }
            ];
            
            for (const order of orders) {
              await txManager.executeOperation({
                query: 'INSERT INTO orders (user_id, product_id, quantity, price, status) VALUES ($1, $2, $3, $4, $5)',
                params: [userId, order.product_id, order.quantity, order.price, 'pending']
              });
            }
            
            // Step 4: Update user statistics
            await txManager.executeOperation({
              query: \`
                UPDATE users 
                SET 
                  order_count = (SELECT COUNT(*) FROM orders WHERE user_id = $1),
                  total_spent = (SELECT COALESCE(SUM(quantity * price), 0) FROM orders WHERE user_id = $1),
                  last_order_date = NOW()
                WHERE id = $1
              \`,
              params: [userId]
            });
            
            // Commit transaction
            const summary = await txManager.commitTransaction();
            
            return [{
              json: {
                success: true,
                userId: userId,
                transactionSummary: summary,
                message: 'User and related data created successfully'
              }
            }];
            
          } catch (error) {
            const rollbackSummary = await txManager.rollbackTransaction();
            
            return [{
              json: {
                success: false,
                error: error.message,
                transactionSummary: rollbackSummary,
                message: 'Transaction failed and was rolled back'
              }
            }];
          }
        `
      },
      "name": "Advanced Transaction Manager",
      "type": "n8n-nodes-base.code",
      "position": [250, 300],
      "typeVersion": 2
    }
  ]
}

// Distributed transaction coordinator (2PC pattern)
const distributedTransactionCoordinator = {
  async executeDistributedTransaction(operations) {
    const participants = new Map();
    const transactionId = `tx_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    
    console.log(`Starting distributed transaction: ${transactionId}`);
    
    try {
      // Phase 1: Prepare all participants
      for (const operation of operations) {
        const participant = {
          database: operation.database,
          connection: operation.connection,
          operations: operation.operations,
          status: 'PREPARING'
        };
        
        participants.set(operation.database, participant);
        
        try {
          await this.prepareParticipant(participant, transactionId);
          participant.status = 'PREPARED';
          console.log(`Participant ${operation.database} prepared successfully`);
        } catch (error) {
          participant.status = 'PREPARE_FAILED';
          participant.error = error;
          console.error(`Participant ${operation.database} prepare failed:`, error);
          throw new Error(`Prepare phase failed for ${operation.database}: ${error.message}`);
        }
      }
      
      // Phase 2: Commit all participants
      const commitResults = [];
      for (const [database, participant] of participants) {
        try {
          const result = await this.commitParticipant(participant, transactionId);
          participant.status = 'COMMITTED';
          commitResults.push({
            database: database,
            success: true,
            result: result
          });
          console.log(`Participant ${database} committed successfully`);
        } catch (error) {
          participant.status = 'COMMIT_FAILED';
          participant.error = error;
          commitResults.push({
            database: database,
            success: false,
            error: error.message
          });
          console.error(`Participant ${database} commit failed:`, error);
          
          // This is a critical error in 2PC
          throw new Error(`Commit phase failed for ${database}: ${error.message}`);
        }
      }
      
      return {
        success: true,
        transactionId: transactionId,
        participants: Array.from(participants.entries()).map(([db, p]) => ({
          database: db,
          status: p.status,
          operationCount: p.operations.length
        })),
        commitResults: commitResults,
        completedAt: new Date().toISOString()
      };
      
    } catch (error) {
      console.error(`Distributed transaction ${transactionId} failed:`, error);
      
      // Rollback all participants
      const rollbackResults = [];
      for (const [database, participant] of participants) {
        if (participant.status === 'PREPARED' || participant.status === 'COMMIT_FAILED') {
          try {
            await this.rollbackParticipant(participant, transactionId);
            participant.status = 'ROLLED_BACK';
            rollbackResults.push({
              database: database,
              success: true
            });
            console.log(`Participant ${database} rolled back successfully`);
          } catch (rollbackError) {
            participant.status = 'ROLLBACK_FAILED';
            rollbackResults.push({
              database: database,
              success: false,
              error: rollbackError.message
            });
            console.error(`Participant ${database} rollback failed:`, rollbackError);
          }
        }
      }
      
      return {
        success: false,
        transactionId: transactionId,
        error: error.message,
        participants: Array.from(participants.entries()).map(([db, p]) => ({
          database: db,
          status: p.status,
          operationCount: p.operations.length,
          error: p.error?.message
        })),
        rollbackResults: rollbackResults,
        failedAt: new Date().toISOString()
      };
    }
  },
  
  async prepareParticipant(participant, transactionId) {
    const connection = participant.connection;
    
    // Begin transaction
    await connection.query('BEGIN');
    
    // Execute all operations
    for (const operation of participant.operations) {
      await connection.query(operation.query, operation.params);
    }
    
    // Prepare transaction (PostgreSQL specific)
    await connection.query(`PREPARE TRANSACTION '${transactionId}_${participant.database}'`);
    
    return true;
  },
  
  async commitParticipant(participant, transactionId) {
    const connection = participant.connection;
    
    // Commit prepared transaction
    await connection.query(`COMMIT PREPARED '${transactionId}_${participant.database}'`);
    
    return {
      operationsExecuted: participant.operations.length,
      committedAt: new Date().toISOString()
    };
  },
  
  async rollbackParticipant(participant, transactionId) {
    const connection = participant.connection;
    
    // Rollback prepared transaction
    await connection.query(`ROLLBACK PREPARED '${transactionId}_${participant.database}'`);
    
    return true;
  }
};
```

## Connection Pooling

### Advanced Pool Management
```javascript
// Sophisticated connection pool with monitoring and optimization
{
  "nodes": [
    {
      "parameters": {
        "jsCode": `
          // Advanced connection pool manager
          class ConnectionPoolManager {
            constructor(config) {
              this.pools = new Map();
              this.config = config;
              this.metrics = {
                totalConnections: 0,
                activeConnections: 0,
                idleConnections: 0,
                waitingClients: 0,
                connectionErrors: 0,
                queryExecutions: 0,
                avgQueryTime: 0,
                poolHits: 0,
                poolMisses: 0
              };
              this.monitoringInterval = null;
              this.startMonitoring();
            }
            
            async createPool(name, poolConfig) {
              const pool = new Pool({
                ...this.config.defaultPoolConfig,
                ...poolConfig,
                // Enhanced pool configuration
                max: poolConfig.max || 20,
                min: poolConfig.min || 2,
                idleTimeoutMillis: poolConfig.idleTimeoutMillis || 30000,
                connectionTimeoutMillis: poolConfig.connectionTimeoutMillis || 5000,
                maxUses: poolConfig.maxUses || 7500,
                
                // Pool event handlers
                onCreateConnection: (connection) => {
                  this.metrics.totalConnections++;
                  console.log(\`New connection created for pool: \${name}\`);
                  
                  // Set connection-level configuration
                  connection.query('SET application_name = $1', [\`n8n_pool_\${name}\`]);
                  connection.query('SET statement_timeout = $1', [poolConfig.statementTimeout || '30s']);
                },
                
                onRemoveConnection: (connection) => {
                  this.metrics.totalConnections--;
                  console.log(\`Connection removed from pool: \${name}\`);
                },
                
                onConnect: (client) => {
                  this.metrics.activeConnections++;
                  console.log(\`Client connected to pool: \${name}\`);
                },
                
                onRelease: (err, client) => {
                  this.metrics.activeConnections--;
                  if (err) {
                    this.metrics.connectionErrors++;
                    console.error(\`Connection error in pool \${name}:\`, err);
                  }
                },
                
                onError: (err, client) => {
                  this.metrics.connectionErrors++;
                  console.error(\`Pool error in \${name}:\`, err);
                }
              });
              
              this.pools.set(name, {
                pool: pool,
                config: poolConfig,
                stats: {
                  created: Date.now(),
                  queryCount: 0,
                  errorCount: 0,
                  avgResponseTime: 0,
                  lastUsed: null
                }
              });
              
              console.log(\`Connection pool '\${name}' created with config:\`, poolConfig);
              return pool;
            }
            
            async getConnection(poolName) {
              const poolInfo = this.pools.get(poolName);
              if (!poolInfo) {
                this.metrics.poolMisses++;
                throw new Error(\`Pool '\${poolName}' not found\`);
              }
              
              this.metrics.poolHits++;
              const startTime = Date.now();
              
              try {
                const client = await poolInfo.pool.connect();
                const connectionTime = Date.now() - startTime;
                
                poolInfo.stats.lastUsed = Date.now();
                
                // Wrap client with monitoring
                return this.wrapClientWithMonitoring(client, poolName, poolInfo);
                
              } catch (error) {
                this.metrics.connectionErrors++;
                poolInfo.stats.errorCount++;
                console.error(\`Failed to get connection from pool '\${poolName}':\`, error);
                throw error;
              }
            }
            
            wrapClientWithMonitoring(client, poolName, poolInfo) {
              const originalQuery = client.query.bind(client);
              
              client.query = async (text, params) => {
                const queryStartTime = Date.now();
                this.metrics.queryExecutions++;
                poolInfo.stats.queryCount++;
                
                try {
                  const result = await originalQuery(text, params);
                  
                  const queryTime = Date.now() - queryStartTime;
                  
                  // Update metrics
                  this.metrics.avgQueryTime = (
                    (this.metrics.avgQueryTime * (this.metrics.queryExecutions - 1) + queryTime) / 
                    this.metrics.queryExecutions
                  );
                  
                  poolInfo.stats.avgResponseTime = (
                    (poolInfo.stats.avgResponseTime * (poolInfo.stats.queryCount - 1) + queryTime) /
                    poolInfo.stats.queryCount
                  );
                  
                  // Log slow queries
                  if (queryTime > (poolInfo.config.slowQueryThreshold || 1000)) {
                    console.warn(\`Slow query detected in pool '\${poolName}' (\${queryTime}ms):\`, {
                      query: text.substring(0, 200),
                      params: params?.slice(0, 5),
                      executionTime: queryTime
                    });
                  }
                  
                  return result;
                  
                } catch (error) {
                  poolInfo.stats.errorCount++;
                  this.metrics.connectionErrors++;
                  console.error(\`Query error in pool '\${poolName}':\`, error);
                  throw error;
                }
              };
              
              return client;
            }
            
            async optimizePools() {
              const optimizations = [];
              
              for (const [name, poolInfo] of this.pools) {
                const pool = poolInfo.pool;
                const stats = poolInfo.stats;
                const currentStats = this.getPoolStats(name);
                
                // Check if pool is underutilized
                if (currentStats.idleCount > poolInfo.config.max * 0.7) {
                  optimizations.push({
                    pool: name,
                    type: 'DOWNSIZE',
                    reason: 'Too many idle connections',
                    recommendation: \`Reduce max connections from \${poolInfo.config.max} to \${Math.ceil(poolInfo.config.max * 0.7)}\`,
                    impact: 'Reduce memory usage'
                  });
                }
                
                // Check if pool is overutilized
                if (currentStats.waitingCount > 0) {
                  optimizations.push({
                    pool: name,
                    type: 'UPSIZE',
                    reason: 'Clients waiting for connections',
                    recommendation: \`Increase max connections from \${poolInfo.config.max} to \${poolInfo.config.max + 5}\`,
                    impact: 'Reduce wait times'
                  });
                }
                
                // Check for high error rate
                const errorRate = stats.errorCount / (stats.queryCount || 1);
                if (errorRate > 0.05) { // More than 5% error rate
                  optimizations.push({
                    pool: name,
                    type: 'HEALTH_CHECK',
                    reason: \`High error rate: \${(errorRate * 100).toFixed(2)}%\`,
                    recommendation: 'Review connection health and database configuration',
                    impact: 'Improve reliability'
                  });
                }
                
                // Check for consistently slow queries
                if (stats.avgResponseTime > 2000) { // More than 2 seconds average
                  optimizations.push({
                    pool: name,
                    type: 'PERFORMANCE',
                    reason: \`Slow average response time: \${stats.avgResponseTime}ms\`,
                    recommendation: 'Review query performance and add indexes',
                    impact: 'Improve application response time'
                  });
                }
              }
              
              return optimizations;
            }
            
            getPoolStats(poolName) {
              const poolInfo = this.pools.get(poolName);
              if (!poolInfo) return null;
              
              const pool = poolInfo.pool;
              
              return {
                totalCount: pool.totalCount,
                idleCount: pool.idleCount,
                waitingCount: pool.waitingCount,
                maxSize: poolInfo.config.max,
                minSize: poolInfo.config.min,
                ...poolInfo.stats
              };
            }
            
            getAllPoolStats() {
              const allStats = {};
              
              for (const [name, poolInfo] of this.pools) {
                allStats[name] = this.getPoolStats(name);
              }
              
              return {
                global: this.metrics,
                pools: allStats,
                timestamp: new Date().toISOString()
              };
            }
            
            startMonitoring() {
              this.monitoringInterval = setInterval(() => {
                const stats = this.getAllPoolStats();
                
                // Log pool health periodically
                console.log('Pool Health Report:', {
                  globalMetrics: stats.global,
                  poolCount: Object.keys(stats.pools).length,
                  timestamp: stats.timestamp
                });
                
                // Auto-optimize pools if enabled
                if (this.config.autoOptimize) {
                  this.optimizePools().then(optimizations => {
                    if (optimizations.length > 0) {
                      console.log('Pool optimization recommendations:', optimizations);
                    }
                  }).catch(error => {
                    console.error('Pool optimization failed:', error);
                  });
                }
                
              }, this.config.monitoringInterval || 60000); // Every minute
            }
            
            stopMonitoring() {
              if (this.monitoringInterval) {
                clearInterval(this.monitoringInterval);
                this.monitoringInterval = null;
              }
            }
            
            async closeAllPools() {
              const closePromises = [];
              
              for (const [name, poolInfo] of this.pools) {
                closePromises.push(
                  poolInfo.pool.end().then(() => {
                    console.log(\`Pool '\${name}' closed successfully\`);
                  }).catch(error => {
                    console.error(\`Error closing pool '\${name}':\`, error);
                  })
                );
              }
              
              await Promise.all(closePromises);
              this.pools.clear();
              this.stopMonitoring();
            }
          }
          
          // Initialize pool manager
          const poolManager = new ConnectionPoolManager({
            defaultPoolConfig: {
              host: process.env.DB_HOST || 'localhost',
              port: process.env.DB_PORT || 5432,
              database: process.env.DB_NAME || 'n8n_db',
              user: process.env.DB_USER || 'n8n_user',
              password: process.env.DB_PASSWORD || 'n8n_password'
            },
            autoOptimize: true,
            monitoringInterval: 30000 // 30 seconds
          });
          
          // Create different pools for different use cases
          await poolManager.createPool('main', {
            max: 20,
            min: 5,
            idleTimeoutMillis: 30000,
            slowQueryThreshold: 1000
          });
          
          await poolManager.createPool('analytics', {
            max: 10,
            min: 2,
            idleTimeoutMillis: 60000,
            slowQueryThreshold: 5000,
            statementTimeout: '5min'
          });
          
          await poolManager.createPool('background', {
            max: 5,
            min: 1,
            idleTimeoutMillis: 120000,
            slowQueryThreshold: 10000,
            statementTimeout: '10min'
          });
          
          // Usage example
          try {
            const mainClient = await poolManager.getConnection('main');
            
            const users = await mainClient.query(
              'SELECT * FROM users WHERE created_at > $1',
              [new Date('2023-01-01')]
            );
            
            mainClient.release();
            
            const stats = poolManager.getAllPoolStats();
            const optimizations = await poolManager.optimizePool();
            
            return [{
              json: {
                success: true,
                userCount: users.rows.length,
                poolStats: stats,
                optimizations: optimizations,
                timestamp: new Date().toISOString()
              }
            }];
            
          } catch (error) {
            return [{
              json: {
                success: false,
                error: error.message,
                poolStats: poolManager.getAllPoolStats(),
                timestamp: new Date().toISOString()
              }
            }];
          }
        `
      },
      "name": "Advanced Pool Manager",
      "type": "n8n-nodes-base.code",
      "position": [250, 300],
      "typeVersion": 2
    }
  ]
}
```

## Practice Exercises

### Exercise 1: Build a Data Migration Pipeline
Create a comprehensive migration system that:
1. Handles schema migrations with rollback capabilities
2. Implements data validation and transformation
3. Supports incremental data migration
4. Includes progress monitoring and error recovery

### Exercise 2: Create a Multi-Database Sync System
Develop a synchronization workflow that:
1. Monitors changes across multiple databases
2. Implements conflict resolution strategies
3. Handles different database types (PostgreSQL, MySQL, MongoDB)
4. Provides real-time sync status and metrics

### Exercise 3: Design a Database Performance Monitor
Build a monitoring system that:
1. Analyzes query performance automatically
2. Provides index optimization recommendations
3. Monitors connection pool health
4. Alerts on performance degradation

## Next Steps

- Complete the learning path with [Monitoring & Maintenance](./16-monitoring-maintenance.md)
- Apply these patterns in real production environments
- Contribute to the N8N database integration community

---

Database operations are the backbone of robust automation workflows. Master these advanced patterns to build scalable, reliable, and high-performance data-driven automations!
