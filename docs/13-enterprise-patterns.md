# Enterprise Patterns for N8N Workflows

Master enterprise-grade automation patterns, architectures, and practices for building scalable, maintainable, and robust automation solutions.

## Table of Contents
- [Enterprise Architecture Principles](#enterprise-architecture-principles)
- [Microservices Patterns](#microservices-patterns)
- [Event-Driven Architecture](#event-driven-architecture)
- [Integration Patterns](#integration-patterns)
- [Workflow Orchestration](#workflow-orchestration)
- [Data Governance](#data-governance)
- [Scalability Patterns](#scalability-patterns)
- [DevOps Integration](#devops-integration)
- [Multi-Tenancy](#multi-tenancy)
- [Enterprise Monitoring](#enterprise-monitoring)

## Enterprise Architecture Principles

### SOLID Principles for Workflows
```javascript
// Applying SOLID principles to N8N workflow design
const SOLIDWorkflowPatterns = {
  // Single Responsibility Principle
  singleResponsibility: {
    principle: 'Each workflow should have one reason to change',
    implementation: {
      goodExample: 'User Registration Workflow - only handles user registration',
      badExample: 'User Management Workflow - handles registration, updates, deletion, notifications',
      pattern: 'Break complex workflows into focused sub-workflows'
    }
  },
  
  // Open/Closed Principle
  openClosed: {
    principle: 'Workflows should be open for extension, closed for modification',
    implementation: {
      strategy: 'Use configuration-driven workflows and plugin patterns',
      example: 'Parameterized workflows that accept configuration without code changes'
    }
  },
  
  // Liskov Substitution Principle
  liskovSubstitution: {
    principle: 'Sub-workflows should be replaceable with their implementations',
    implementation: {
      strategy: 'Standardized interfaces between workflows',
      example: 'Payment processing workflows that can be swapped without affecting calling workflows'
    }
  },
  
  // Interface Segregation Principle
  interfaceSegregation: {
    principle: 'Workflows should not depend on interfaces they dont use',
    implementation: {
      strategy: 'Minimal, focused APIs between workflows',
      example: 'Separate read and write interfaces for data operations'
    }
  },
  
  // Dependency Inversion Principle
  dependencyInversion: {
    principle: 'Depend on abstractions, not concretions',
    implementation: {
      strategy: 'Use service abstractions and dependency injection',
      example: 'Email service abstraction that can use different providers'
    }
  }
};

// Enterprise workflow architecture framework
class EnterpriseWorkflowArchitecture {
  constructor(config = {}) {
    this.config = {
      environment: config.environment || 'production',
      scalingStrategy: config.scalingStrategy || 'horizontal',
      resilienceLevel: config.resilienceLevel || 'high',
      ...config
    };
    
    this.architecturalPatterns = new Map();
    this.serviceCatalog = new Map();
    this.dependencyGraph = new Map();
  }
  
  defineArchitecturalPattern(name, pattern) {
    this.architecturalPatterns.set(name, {
      name: name,
      pattern: pattern,
      applicableScenarios: pattern.scenarios,
      benefits: pattern.benefits,
      tradeoffs: pattern.tradeoffs,
      implementation: pattern.implementation
    });
  }
  
  registerService(serviceName, serviceDefinition) {
    this.serviceCatalog.set(serviceName, {
      name: serviceName,
      version: serviceDefinition.version,
      interface: serviceDefinition.interface,
      implementation: serviceDefinition.implementation,
      dependencies: serviceDefinition.dependencies || [],
      capabilities: serviceDefinition.capabilities || [],
      sla: serviceDefinition.sla || {},
      registeredAt: new Date().toISOString()
    });
    
    // Build dependency graph
    this.updateDependencyGraph(serviceName, serviceDefinition.dependencies);
  }
  
  updateDependencyGraph(serviceName, dependencies) {
    this.dependencyGraph.set(serviceName, dependencies);
    
    // Check for circular dependencies
    if (this.hasCircularDependencies()) {
      throw new Error(`Circular dependency detected involving service: ${serviceName}`);
    }
  }
  
  hasCircularDependencies() {
    const visited = new Set();
    const recursionStack = new Set();
    
    const dfs = (service) => {
      if (recursionStack.has(service)) return true;
      if (visited.has(service)) return false;
      
      visited.add(service);
      recursionStack.add(service);
      
      const dependencies = this.dependencyGraph.get(service) || [];
      for (const dep of dependencies) {
        if (dfs(dep)) return true;
      }
      
      recursionStack.delete(service);
      return false;
    };
    
    for (const service of this.dependencyGraph.keys()) {
      if (dfs(service)) return true;
    }
    
    return false;
  }
  
  generateArchitecturalView() {
    return {
      services: Array.from(this.serviceCatalog.values()),
      patterns: Array.from(this.architecturalPatterns.values()),
      dependencies: Object.fromEntries(this.dependencyGraph),
      healthMetrics: this.getArchitecturalHealth(),
      recommendations: this.generateArchitecturalRecommendations()
    };
  }
  
  getArchitecturalHealth() {
    const services = Array.from(this.serviceCatalog.values());
    const totalServices = services.length;
    
    // Calculate coupling metrics
    const avgDependencies = totalServices > 0 
      ? Array.from(this.dependencyGraph.values())
          .reduce((sum, deps) => sum + deps.length, 0) / totalServices
      : 0;
    
    // Calculate cohesion metrics
    const servicesByDomain = this.groupServicesByDomain(services);
    const domainCohesion = this.calculateDomainCohesion(servicesByDomain);
    
    return {
      totalServices: totalServices,
      averageDependencies: avgDependencies,
      domainCohesion: domainCohesion,
      couplingLevel: this.assessCouplingLevel(avgDependencies),
      architecturalDebt: this.calculateArchitecturalDebt()
    };
  }
  
  groupServicesByDomain(services) {
    return services.reduce((domains, service) => {
      const domain = this.extractDomain(service.name);
      if (!domains[domain]) domains[domain] = [];
      domains[domain].push(service);
      return domains;
    }, {});
  }
  
  extractDomain(serviceName) {
    // Extract domain from service name (e.g., "user-registration" -> "user")
    return serviceName.split('-')[0] || 'general';
  }
  
  calculateDomainCohesion(servicesByDomain) {
    const domains = Object.keys(servicesByDomain);
    const totalDomains = domains.length;
    
    if (totalDomains === 0) return 0;
    
    const avgServicesPerDomain = domains.reduce((sum, domain) => 
      sum + servicesByDomain[domain].length, 0) / totalDomains;
    
    // Higher cohesion = more services per domain (focused domains)
    return Math.min(avgServicesPerDomain / 5, 1); // Normalize to 0-1 scale
  }
  
  assessCouplingLevel(avgDependencies) {
    if (avgDependencies > 5) return 'HIGH';
    if (avgDependencies > 2) return 'MEDIUM';
    return 'LOW';
  }
  
  calculateArchitecturalDebt() {
    let debtScore = 0;
    
    // Penalize high coupling
    const avgDeps = Array.from(this.dependencyGraph.values())
      .reduce((sum, deps) => sum + deps.length, 0) / this.dependencyGraph.size;
    
    if (avgDeps > 5) debtScore += 30;
    else if (avgDeps > 3) debtScore += 15;
    
    // Penalize services without proper versioning
    const unversionedServices = Array.from(this.serviceCatalog.values())
      .filter(service => !service.version || service.version === '1.0.0');
    
    debtScore += (unversionedServices.length / this.serviceCatalog.size) * 20;
    
    // Penalize services without SLAs
    const servicesWithoutSLA = Array.from(this.serviceCatalog.values())
      .filter(service => !service.sla || Object.keys(service.sla).length === 0);
    
    debtScore += (servicesWithoutSLA.length / this.serviceCatalog.size) * 25;
    
    return Math.min(debtScore, 100); // Cap at 100
  }
  
  generateArchitecturalRecommendations() {
    const recommendations = [];
    const health = this.getArchitecturalHealth();
    
    if (health.couplingLevel === 'HIGH') {
      recommendations.push({
        priority: 'HIGH',
        category: 'COUPLING',
        title: 'Reduce service coupling',
        description: 'High average dependencies detected. Consider breaking down tightly coupled services.',
        action: 'Review service boundaries and extract shared concerns into separate services'
      });
    }
    
    if (health.domainCohesion < 0.5) {
      recommendations.push({
        priority: 'MEDIUM',
        category: 'COHESION',
        title: 'Improve domain cohesion',
        description: 'Services are scattered across too many domains.',
        action: 'Consolidate related services into well-defined domain boundaries'
      });
    }
    
    if (health.architecturalDebt > 50) {
      recommendations.push({
        priority: 'HIGH',
        category: 'TECHNICAL_DEBT',
        title: 'Address architectural debt',
        description: 'High technical debt score detected.',
        action: 'Implement proper versioning, SLAs, and service governance'
      });
    }
    
    return recommendations;
  }
}

// Define common enterprise patterns
const enterpriseArchitecture = new EnterpriseWorkflowArchitecture();

// Saga Pattern for distributed transactions
enterpriseArchitecture.defineArchitecturalPattern('saga', {
  scenarios: ['Distributed transactions', 'Multi-service operations'],
  benefits: ['Maintains data consistency', 'Handles partial failures'],
  tradeoffs: ['Complexity', 'Eventual consistency'],
  implementation: {
    type: 'orchestration',
    compensationStrategy: 'rollback',
    stateManagement: 'centralized'
  }
});

// Circuit Breaker Pattern
enterpriseArchitecture.defineArchitecturalPattern('circuitBreaker', {
  scenarios: ['External service integration', 'Fault tolerance'],
  benefits: ['Prevents cascading failures', 'Graceful degradation'],
  tradeoffs: ['Configuration complexity', 'False positives'],
  implementation: {
    thresholds: { failure: 5, timeout: 30000 },
    states: ['CLOSED', 'OPEN', 'HALF_OPEN']
  }
});

// CQRS Pattern
enterpriseArchitecture.defineArchitecturalPattern('cqrs', {
  scenarios: ['Complex read/write operations', 'Performance optimization'],
  benefits: ['Separate scaling', 'Optimized queries', 'Clear separation'],
  tradeoffs: ['Data consistency', 'Complexity'],
  implementation: {
    commandSide: 'write-optimized',
    querySide: 'read-optimized',
    synchronization: 'event-driven'
  }
});
```

### Domain-Driven Design for Workflows
```javascript
// Domain-Driven Design implementation for N8N workflows
class WorkflowDomainModel {
  constructor(domainName) {
    this.domainName = domainName;
    this.aggregates = new Map();
    this.services = new Map();
    this.repositories = new Map();
    this.domainEvents = new Map();
    this.valueObjects = new Map();
  }
  
  defineAggregate(name, aggregateRoot) {
    this.aggregates.set(name, {
      name: name,
      root: aggregateRoot,
      entities: aggregateRoot.entities || [],
      valueObjects: aggregateRoot.valueObjects || [],
      invariants: aggregateRoot.invariants || [],
      events: aggregateRoot.events || []
    });
  }
  
  defineDomainService(name, service) {
    this.services.set(name, {
      name: name,
      operations: service.operations,
      dependencies: service.dependencies || [],
      contracts: service.contracts || {}
    });
  }
  
  defineRepository(aggregateName, repository) {
    this.repositories.set(aggregateName, {
      aggregate: aggregateName,
      operations: repository.operations,
      persistence: repository.persistence || 'database',
      caching: repository.caching || false
    });
  }
  
  defineDomainEvent(name, event) {
    this.domainEvents.set(name, {
      name: name,
      schema: event.schema,
      triggers: event.triggers || [],
      handlers: event.handlers || []
    });
  }
  
  defineValueObject(name, valueObject) {
    this.valueObjects.set(name, {
      name: name,
      properties: valueObject.properties,
      validation: valueObject.validation || {},
      immutable: valueObject.immutable !== false
    });
  }
  
  generateWorkflowDefinitions() {
    const workflows = [];
    
    // Generate workflows for each aggregate
    for (const [name, aggregate] of this.aggregates.entries()) {
      workflows.push(this.generateAggregateWorkflows(name, aggregate));
    }
    
    // Generate workflows for domain services
    for (const [name, service] of this.services.entries()) {
      workflows.push(this.generateServiceWorkflows(name, service));
    }
    
    return workflows.flat();
  }
  
  generateAggregateWorkflows(aggregateName, aggregate) {
    const workflows = [];
    
    // Create workflow for each aggregate operation
    for (const operation of aggregate.root.operations || []) {
      workflows.push({
        name: `${this.domainName}-${aggregateName}-${operation.name}`,
        type: 'aggregate-operation',
        aggregate: aggregateName,
        operation: operation.name,
        nodes: this.generateOperationNodes(operation, aggregate),
        events: this.getAggregateEvents(aggregate),
        invariants: aggregate.invariants
      });
    }
    
    return workflows;
  }
  
  generateServiceWorkflows(serviceName, service) {
    const workflows = [];
    
    for (const operation of service.operations) {
      workflows.push({
        name: `${this.domainName}-service-${serviceName}-${operation.name}`,
        type: 'domain-service',
        service: serviceName,
        operation: operation.name,
        nodes: this.generateServiceNodes(operation, service),
        contracts: service.contracts
      });
    }
    
    return workflows;
  }
  
  generateOperationNodes(operation, aggregate) {
    const nodes = [];
    
    // Input validation node
    nodes.push({
      type: 'validation',
      name: 'validate-input',
      config: {
        rules: operation.validation || {},
        valueObjects: this.extractRelevantValueObjects(operation)
      }
    });
    
    // Business logic node
    nodes.push({
      type: 'business-logic',
      name: 'execute-operation',
      config: {
        operation: operation.name,
        invariants: aggregate.invariants,
        businessRules: operation.businessRules || []
      }
    });
    
    // Event publishing node
    if (operation.events && operation.events.length > 0) {
      nodes.push({
        type: 'event-publisher',
        name: 'publish-events',
        config: {
          events: operation.events
        }
      });
    }
    
    // Persistence node
    nodes.push({
      type: 'repository',
      name: 'persist-changes',
      config: {
        repository: aggregate.name,
        operation: 'save'
      }
    });
    
    return nodes;
  }
  
  generateServiceNodes(operation, service) {
    const nodes = [];
    
    // Service orchestration node
    nodes.push({
      type: 'service-orchestration',
      name: 'orchestrate-operation',
      config: {
        service: service.name,
        operation: operation.name,
        dependencies: service.dependencies,
        contracts: service.contracts
      }
    });
    
    return nodes;
  }
  
  extractRelevantValueObjects(operation) {
    const relevant = [];
    
    for (const [name, valueObject] of this.valueObjects.entries()) {
      // Check if value object is used in operation
      if (operation.parameters && operation.parameters.includes(name)) {
        relevant.push(valueObject);
      }
    }
    
    return relevant;
  }
  
  getAggregateEvents(aggregate) {
    return aggregate.events.map(eventName => this.domainEvents.get(eventName));
  }
  
  validateDomainModel() {
    const validation = {
      isValid: true,
      errors: [],
      warnings: []
    };
    
    // Check aggregate consistency
    for (const [name, aggregate] of this.aggregates.entries()) {
      if (!aggregate.root || !aggregate.root.operations) {
        validation.errors.push(`Aggregate ${name} missing operations`);
        validation.isValid = false;
      }
    }
    
    // Check service dependencies
    for (const [name, service] of this.services.entries()) {
      for (const dep of service.dependencies) {
        if (!this.services.has(dep) && !this.aggregates.has(dep)) {
          validation.errors.push(`Service ${name} has unresolved dependency: ${dep}`);
          validation.isValid = false;
        }
      }
    }
    
    // Check event handlers
    for (const [name, event] of this.domainEvents.entries()) {
      if (!event.handlers || event.handlers.length === 0) {
        validation.warnings.push(`Domain event ${name} has no handlers`);
      }
    }
    
    return validation;
  }
}

// Example: E-commerce domain model
const ecommerceDomain = new WorkflowDomainModel('ecommerce');

// Define value objects
ecommerceDomain.defineValueObject('Money', {
  properties: ['amount', 'currency'],
  validation: {
    amount: { type: 'number', min: 0 },
    currency: { type: 'string', length: 3 }
  }
});

ecommerceDomain.defineValueObject('Email', {
  properties: ['address'],
  validation: {
    address: { type: 'email' }
  }
});

// Define aggregates
ecommerceDomain.defineAggregate('Order', {
  entities: ['OrderItem', 'ShippingAddress'],
  valueObjects: ['Money', 'Email'],
  operations: [
    {
      name: 'createOrder',
      parameters: ['customerId', 'items', 'shippingAddress'],
      validation: { customerId: { required: true } },
      events: ['OrderCreated'],
      businessRules: ['minimumOrderAmount', 'inventoryAvailability']
    },
    {
      name: 'cancelOrder',
      parameters: ['orderId', 'reason'],
      events: ['OrderCancelled'],
      businessRules: ['cancellationPolicy']
    }
  ],
  invariants: [
    'Order must have at least one item',
    'Order total must be positive',
    'Shipping address must be valid'
  ],
  events: ['OrderCreated', 'OrderCancelled', 'OrderShipped']
});

// Define domain services
ecommerceDomain.defineDomainService('PricingService', {
  operations: [
    {
      name: 'calculateOrderTotal',
      parameters: ['orderItems', 'discounts', 'taxes']
    },
    {
      name: 'applyPromotions',
      parameters: ['order', 'promotionCodes']
    }
  ],
  dependencies: ['PromotionRepository', 'TaxRepository']
});

// Define domain events
ecommerceDomain.defineDomainEvent('OrderCreated', {
  schema: {
    orderId: 'string',
    customerId: 'string',
    orderTotal: 'Money',
    timestamp: 'datetime'
  },
  triggers: ['order creation'],
  handlers: ['InventoryService', 'NotificationService', 'AnalyticsService']
});

// Generate workflow definitions
const workflowDefinitions = ecommerceDomain.generateWorkflowDefinitions();
const domainValidation = ecommerceDomain.validateDomainModel();

console.log('Generated Workflows:', workflowDefinitions);
console.log('Domain Model Validation:', domainValidation);
```

## Microservices Patterns

### Service Decomposition Strategies
```javascript
// Service decomposition framework for workflow-based microservices
class ServiceDecompositionFramework {
  constructor() {
    this.decompositionStrategies = new Map();
    this.serviceRegistry = new Map();
    this.boundaryAnalysis = new Map();
  }
  
  defineDecompositionStrategy(name, strategy) {
    this.decompositionStrategies.set(name, {
      name: name,
      criteria: strategy.criteria,
      implementation: strategy.implementation,
      benefits: strategy.benefits,
      challenges: strategy.challenges
    });
  }
  
  analyzeServiceBoundaries(monolithWorkflow) {
    const analysis = {
      workflow: monolithWorkflow.name,
      currentStructure: this.analyzeCurrentStructure(monolithWorkflow),
      decompositionOptions: this.generateDecompositionOptions(monolithWorkflow),
      recommendations: this.generateDecompositionRecommendations(monolithWorkflow)
    };
    
    this.boundaryAnalysis.set(monolithWorkflow.name, analysis);
    return analysis;
  }
  
  analyzeCurrentStructure(workflow) {
    const structure = {
      totalNodes: workflow.nodes.length,
      nodeTypes: this.categorizeNodes(workflow.nodes),
      dataFlow: this.analyzeDataFlow(workflow),
      coupling: this.analyzeCoupling(workflow),
      cohesion: this.analyzeCohesion(workflow)
    };
    
    return structure;
  }
  
  categorizeNodes(nodes) {
    const categories = {
      triggers: [],
      businessLogic: [],
      dataAccess: [],
      integrations: [],
      utilities: []
    };
    
    nodes.forEach(node => {
      switch (node.type) {
        case 'trigger':
        case 'webhook':
          categories.triggers.push(node);
          break;
        case 'function':
        case 'code':
          categories.businessLogic.push(node);
          break;
        case 'database':
        case 'file':
          categories.dataAccess.push(node);
          break;
        case 'http':
        case 'api':
          categories.integrations.push(node);
          break;
        default:
          categories.utilities.push(node);
      }
    });
    
    return categories;
  }
  
  analyzeDataFlow(workflow) {
    const dataFlow = {
      sources: [],
      sinks: [],
      transformations: [],
      sharedData: []
    };
    
    // Analyze data dependencies between nodes
    workflow.nodes.forEach(node => {
      if (node.inputData) {
        dataFlow.sources.push({
          nodeId: node.id,
          dataTypes: Object.keys(node.inputData)
        });
      }
      
      if (node.outputData) {
        dataFlow.sinks.push({
          nodeId: node.id,
          dataTypes: Object.keys(node.outputData)
        });
      }
      
      if (node.type === 'function' && node.transformsData) {
        dataFlow.transformations.push({
          nodeId: node.id,
          transformations: node.transformsData
        });
      }
    });
    
    return dataFlow;
  }
  
  analyzeCoupling(workflow) {
    const coupling = {
      dataCoupling: this.calculateDataCoupling(workflow),
      controlCoupling: this.calculateControlCoupling(workflow),
      temporalCoupling: this.calculateTemporalCoupling(workflow)
    };
    
    return coupling;
  }
  
  analyzeCohesion(workflow) {
    const cohesion = {
      functionalCohesion: this.calculateFunctionalCohesion(workflow),
      dataCohesion: this.calculateDataCohesion(workflow),
      temporalCohesion: this.calculateTemporalCohesion(workflow)
    };
    
    return cohesion;
  }
  
  generateDecompositionOptions(workflow) {
    const options = [];
    
    // Option 1: Decompose by business capability
    options.push(this.decomposeByBusinessCapability(workflow));
    
    // Option 2: Decompose by data model
    options.push(this.decomposeByDataModel(workflow));
    
    // Option 3: Decompose by scalability requirements
    options.push(this.decomposeByScalability(workflow));
    
    // Option 4: Decompose by team structure
    options.push(this.decomposeByTeamStructure(workflow));
    
    return options;
  }
  
  decomposeByBusinessCapability(workflow) {
    const capabilities = this.identifyBusinessCapabilities(workflow);
    
    return {
      strategy: 'business-capability',
      services: capabilities.map(capability => ({
        name: `${capability.name}-service`,
        capability: capability.name,
        nodes: capability.nodes,
        responsibilities: capability.responsibilities,
        boundaries: capability.boundaries
      })),
      benefits: [
        'Aligned with business functions',
        'Clear ownership',
        'Independent evolution'
      ],
      challenges: [
        'Potential data duplication',
        'Cross-capability transactions'
      ]
    };
  }
  
  decomposeByDataModel(workflow) {
    const dataModels = this.identifyDataModels(workflow);
    
    return {
      strategy: 'data-model',
      services: dataModels.map(model => ({
        name: `${model.name}-service`,
        dataModel: model.name,
        nodes: model.nodes,
        dataOwnership: model.entities,
        operations: model.operations
      })),
      benefits: [
        'Clear data ownership',
        'Reduced data coupling',
        'Consistency within service'
      ],
      challenges: [
        'Cross-service queries',
        'Referential integrity'
      ]
    };
  }
  
  decomposeByScalability(workflow) {
    const scalabilityGroups = this.identifyScalabilityGroups(workflow);
    
    return {
      strategy: 'scalability',
      services: scalabilityGroups.map(group => ({
        name: `${group.name}-service`,
        scalabilityProfile: group.profile,
        nodes: group.nodes,
        performanceRequirements: group.requirements,
        resourceNeeds: group.resources
      })),
      benefits: [
        'Optimized resource allocation',
        'Independent scaling',
        'Performance isolation'
      ],
      challenges: [
        'Complex coordination',
        'Potential over-engineering'
      ]
    };
  }
  
  identifyBusinessCapabilities(workflow) {
    const capabilities = [];
    
    // Group nodes by business function
    const businessFunctions = new Map();
    
    workflow.nodes.forEach(node => {
      const businessFunction = this.extractBusinessFunction(node);
      
      if (!businessFunctions.has(businessFunction)) {
        businessFunctions.set(businessFunction, {
          name: businessFunction,
          nodes: [],
          responsibilities: [],
          boundaries: []
        });
      }
      
      businessFunctions.get(businessFunction).nodes.push(node);
    });
    
    return Array.from(businessFunctions.values());
  }
  
  extractBusinessFunction(node) {
    // Extract business function from node configuration
    if (node.businessFunction) return node.businessFunction;
    if (node.domain) return node.domain;
    
    // Infer from node type and name
    const name = node.name.toLowerCase();
    
    if (name.includes('user') || name.includes('account')) return 'user-management';
    if (name.includes('order') || name.includes('payment')) return 'order-management';
    if (name.includes('inventory') || name.includes('product')) return 'inventory-management';
    if (name.includes('notification') || name.includes('email')) return 'notification';
    
    return 'general';
  }
  
  calculateDataCoupling(workflow) {
    // Calculate coupling based on shared data structures
    let couplingScore = 0;
    const totalPairs = workflow.nodes.length * (workflow.nodes.length - 1) / 2;
    
    for (let i = 0; i < workflow.nodes.length; i++) {
      for (let j = i + 1; j < workflow.nodes.length; j++) {
        const node1 = workflow.nodes[i];
        const node2 = workflow.nodes[j];
        
        const sharedData = this.calculateSharedData(node1, node2);
        if (sharedData > 0) couplingScore++;
      }
    }
    
    return totalPairs > 0 ? couplingScore / totalPairs : 0;
  }
  
  calculateSharedData(node1, node2) {
    const data1 = new Set([
      ...(node1.inputData ? Object.keys(node1.inputData) : []),
      ...(node1.outputData ? Object.keys(node1.outputData) : [])
    ]);
    
    const data2 = new Set([
      ...(node2.inputData ? Object.keys(node2.inputData) : []),
      ...(node2.outputData ? Object.keys(node2.outputData) : [])
    ]);
    
    const intersection = new Set([...data1].filter(x => data2.has(x)));
    return intersection.size;
  }
  
  generateDecompositionRecommendations(workflow) {
    const recommendations = [];
    const structure = this.analyzeCurrentStructure(workflow);
    
    if (structure.coupling.dataCoupling > 0.7) {
      recommendations.push({
        priority: 'HIGH',
        type: 'COUPLING_REDUCTION',
        description: 'High data coupling detected',
        action: 'Consider decomposing by data model to reduce coupling',
        strategy: 'data-model'
      });
    }
    
    if (structure.totalNodes > 20) {
      recommendations.push({
        priority: 'MEDIUM',
        type: 'COMPLEXITY_REDUCTION',
        description: 'Large workflow detected',
        action: 'Break down into smaller services by business capability',
        strategy: 'business-capability'
      });
    }
    
    const businessLogicNodes = structure.nodeTypes.businessLogic.length;
    const integrationNodes = structure.nodeTypes.integrations.length;
    
    if (integrationNodes > businessLogicNodes) {
      recommendations.push({
        priority: 'MEDIUM',
        type: 'SEPARATION_OF_CONCERNS',
        description: 'Integration-heavy workflow',
        action: 'Separate integration logic from business logic',
        strategy: 'layered-architecture'
      });
    }
    
    return recommendations;
  }
}

// Usage example
const decompositionFramework = new ServiceDecompositionFramework();

// Define decomposition strategies
decompositionFramework.defineDecompositionStrategy('business-capability', {
  criteria: ['Business function alignment', 'Team ownership', 'Change frequency'],
  implementation: 'Group nodes by business domain',
  benefits: ['Clear boundaries', 'Independent teams', 'Business alignment'],
  challenges: ['Data consistency', 'Cross-service transactions']
});

// Analyze a monolithic workflow
const monolithWorkflow = {
  name: 'e-commerce-order-processing',
  nodes: [
    { id: 'trigger', type: 'webhook', name: 'order-webhook' },
    { id: 'validate', type: 'function', name: 'validate-order', businessFunction: 'order-management' },
    { id: 'inventory', type: 'database', name: 'check-inventory', businessFunction: 'inventory-management' },
    { id: 'payment', type: 'http', name: 'process-payment', businessFunction: 'payment-processing' },
    { id: 'user', type: 'database', name: 'update-user-profile', businessFunction: 'user-management' },
    { id: 'notification', type: 'email', name: 'send-confirmation', businessFunction: 'notification' }
  ]
};

const decompositionAnalysis = decompositionFramework.analyzeServiceBoundaries(monolithWorkflow);
console.log('Decomposition Analysis:', decompositionAnalysis);
```

### Service Communication Patterns
```javascript
// Service communication patterns for distributed N8N workflows
class ServiceCommunicationManager {
  constructor(config = {}) {
    this.config = {
      defaultTimeout: config.defaultTimeout || 30000,
      retryPolicy: config.retryPolicy || { maxRetries: 3, backoff: 'exponential' },
      circuitBreaker: config.circuitBreaker || { threshold: 5, timeout: 60000 },
      ...config
    };
    
    this.communicationPatterns = new Map();
    this.serviceRegistry = new Map();
    this.messageQueues = new Map();
    this.eventBus = new EventBus();
  }
  
  registerCommunicationPattern(name, pattern) {
    this.communicationPatterns.set(name, {
      name: name,
      type: pattern.type, // sync, async, event-driven
      protocol: pattern.protocol, // http, message-queue, event-stream
      reliability: pattern.reliability, // at-least-once, exactly-once, at-most-once
      implementation: pattern.implementation
    });
  }
  
  registerService(serviceName, serviceConfig) {
    this.serviceRegistry.set(serviceName, {
      name: serviceName,
      endpoint: serviceConfig.endpoint,
      version: serviceConfig.version,
      capabilities: serviceConfig.capabilities || [],
      communicationPatterns: serviceConfig.patterns || ['request-response'],
      healthCheck: serviceConfig.healthCheck,
      sla: serviceConfig.sla || {}
    });
  }
  
  // Synchronous Request-Response Pattern
  async requestResponse(fromService, toService, request, options = {}) {
    const pattern = this.communicationPatterns.get('request-response');
    const targetService = this.serviceRegistry.get(toService);
    
    if (!targetService) {
      throw new Error(`Service not found: ${toService}`);
    }
    
    const requestConfig = {
      method: options.method || 'POST',
      timeout: options.timeout || this.config.defaultTimeout,
      headers: {
        'Content-Type': 'application/json',
        'X-Source-Service': fromService,
        'X-Request-ID': this.generateRequestId(),
        ...options.headers
      },
      body: JSON.stringify(request)
    };
    
    try {
      const response = await this.executeWithRetry(
        () => this.makeHttpRequest(targetService.endpoint, requestConfig),
        this.config.retryPolicy
      );
      
      return {
        success: true,
        data: response.data,
        metadata: {
          requestId: requestConfig.headers['X-Request-ID'],
          responseTime: response.responseTime,
          fromService: fromService,
          toService: toService
        }
      };
      
    } catch (error) {
      return {
        success: false,
        error: error.message,
        metadata: {
          requestId: requestConfig.headers['X-Request-ID'],
          fromService: fromService,
          toService: toService
        }
      };
    }
  }
  
  // Asynchronous Fire-and-Forget Pattern
  async fireAndForget(fromService, toService, message, options = {}) {
    const queueName = options.queue || `${fromService}-to-${toService}`;
    
    if (!this.messageQueues.has(queueName)) {
      this.messageQueues.set(queueName, new MessageQueue(queueName));
    }
    
    const messageQueue = this.messageQueues.get(queueName);
    
    const messageConfig = {
      id: this.generateMessageId(),
      timestamp: new Date().toISOString(),
      fromService: fromService,
      toService: toService,
      payload: message,
      priority: options.priority || 'normal',
      ttl: options.ttl || 3600000 // 1 hour default TTL
    };
    
    await messageQueue.enqueue(messageConfig);
    
    return {
      messageId: messageConfig.id,
      queueName: queueName,
      enqueuedAt: messageConfig.timestamp
    };
  }
  
  // Event-Driven Communication Pattern
  async publishEvent(fromService, eventType, eventData, options = {}) {
    const event = {
      id: this.generateEventId(),
      type: eventType,
      source: fromService,
      timestamp: new Date().toISOString(),
      data: eventData,
      version: options.version || '1.0',
      correlationId: options.correlationId
    };
    
    await this.eventBus.publish(event);
    
    return {
      eventId: event.id,
      published: true,
      timestamp: event.timestamp
    };
  }
  
  // Subscribe to events
  subscribeToEvent(serviceName, eventType, handler) {
    this.eventBus.subscribe(eventType, async (event) => {
      try {
        await handler(event);
      } catch (error) {
        console.error(`Error handling event ${event.id} in service ${serviceName}:`, error);
        
        // Publish error event
        await this.publishEvent(serviceName, 'EventProcessingFailed', {
          originalEvent: event,
          error: error.message,
          serviceName: serviceName
        });
      }
    });
  }
  
  // Saga Pattern for Distributed Transactions
  async executeSaga(sagaDefinition) {
    const sagaInstance = {
      id: this.generateSagaId(),
      definition: sagaDefinition,
      currentStep: 0,
      completedSteps: [],
      compensationStack: [],
      status: 'RUNNING',
      startedAt: new Date().toISOString()
    };
    
    try {
      for (let i = 0; i < sagaDefinition.steps.length; i++) {
        const step = sagaDefinition.steps[i];
        
        try {
          const result = await this.executeSagaStep(step, sagaInstance);
          
          sagaInstance.completedSteps.push({
            step: i,
            result: result,
            completedAt: new Date().toISOString()
          });
          
          // Add compensation action to stack
          if (step.compensation) {
            sagaInstance.compensationStack.push(step.compensation);
          }
          
          sagaInstance.currentStep = i + 1;
          
        } catch (error) {
          // Step failed, trigger compensation
          sagaInstance.status = 'COMPENSATING';
          await this.compensateSaga(sagaInstance);
          
          sagaInstance.status = 'FAILED';
          sagaInstance.failedAt = new Date().toISOString();
          sagaInstance.error = error.message;
          
          throw error;
        }
      }
      
      sagaInstance.status = 'COMPLETED';
      sagaInstance.completedAt = new Date().toISOString();
      
      return sagaInstance;
      
    } catch (error) {
      return sagaInstance;
    }
  }
  
  async executeSagaStep(step, sagaInstance) {
    const stepConfig = {
      sagaId: sagaInstance.id,
      stepName: step.name,
      input: step.input,
      timeout: step.timeout || this.config.defaultTimeout
    };
    
    switch (step.type) {
      case 'service-call':
        return await this.requestResponse(
          sagaInstance.definition.orchestrator,
          step.service,
          stepConfig,
          { timeout: step.timeout }
        );
        
      case 'event-publish':
        return await this.publishEvent(
          sagaInstance.definition.orchestrator,
          step.eventType,
          stepConfig
        );
        
      case 'delay':
        await new Promise(resolve => setTimeout(resolve, step.duration));
        return { delayed: step.duration };
        
      default:
        throw new Error(`Unknown saga step type: ${step.type}`);
    }
  }
  
  async compensateSaga(sagaInstance) {
    // Execute compensation actions in reverse order
    for (let i = sagaInstance.compensationStack.length - 1; i >= 0; i--) {
      const compensation = sagaInstance.compensationStack[i];
      
      try {
        await this.executeCompensationAction(compensation, sagaInstance);
      } catch (error) {
        console.error(`Compensation failed for saga ${sagaInstance.id}:`, error);
        // Continue with other compensations even if one fails
      }
    }
  }
  
  async executeCompensationAction(compensation, sagaInstance) {
    const compensationConfig = {
      sagaId: sagaInstance.id,
      compensationFor: compensation.step,
      input: compensation.input
    };
    
    switch (compensation.type) {
      case 'service-call':
        return await this.requestResponse(
          sagaInstance.definition.orchestrator,
          compensation.service,
          compensationConfig
        );
        
      case 'event-publish':
        return await this.publishEvent(
          sagaInstance.definition.orchestrator,
          compensation.eventType,
          compensationConfig
        );
        
      default:
        throw new Error(`Unknown compensation type: ${compensation.type}`);
    }
  }
  
  async makeHttpRequest(endpoint, config) {
    const startTime = Date.now();
    
    const response = await fetch(endpoint, {
      method: config.method,
      headers: config.headers,
      body: config.body,
      signal: AbortSignal.timeout(config.timeout)
    });
    
    const responseTime = Date.now() - startTime;
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    
    const data = await response.json();
    
    return {
      data: data,
      responseTime: responseTime,
      status: response.status
    };
  }
  
  async executeWithRetry(operation, retryPolicy) {
    let lastError = null;
    
    for (let attempt = 0; attempt <= retryPolicy.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt < retryPolicy.maxRetries) {
          const delay = this.calculateRetryDelay(attempt, retryPolicy.backoff);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }
    
    throw lastError;
  }
  
  calculateRetryDelay(attempt, backoffType) {
    switch (backoffType) {
      case 'exponential':
        return Math.min(1000 * Math.pow(2, attempt), 30000);
      case 'linear':
        return 1000 * (attempt + 1);
      case 'fixed':
        return 1000;
      default:
        return 1000;
    }
  }
  
  generateRequestId() {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  generateMessageId() {
    return `msg_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  generateEventId() {
    return `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  generateSagaId() {
    return `saga_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Supporting classes
class MessageQueue {
  constructor(name) {
    this.name = name;
    this.messages = [];
    this.subscribers = [];
  }
  
  async enqueue(message) {
    this.messages.push(message);
    await this.processMessages();
  }
  
  async processMessages() {
    while (this.messages.length > 0 && this.subscribers.length > 0) {
      const message = this.messages.shift();
      const subscriber = this.subscribers.shift();
      
      try {
        await subscriber.handler(message);
        this.subscribers.push(subscriber); // Re-add subscriber for next message
      } catch (error) {
        console.error(`Message processing failed:`, error);
        // Optionally implement dead letter queue
      }
    }
  }
  
  subscribe(handler) {
    this.subscribers.push({ handler });
  }
}

class EventBus {
  constructor() {
    this.subscriptions = new Map();
  }
  
  subscribe(eventType, handler) {
    if (!this.subscriptions.has(eventType)) {
      this.subscriptions.set(eventType, []);
    }
    this.subscriptions.get(eventType).push(handler);
  }
  
  async publish(event) {
    const handlers = this.subscriptions.get(event.type) || [];
    
    await Promise.all(handlers.map(async (handler) => {
      try {
        await handler(event);
      } catch (error) {
        console.error(`Event handler failed for ${event.type}:`, error);
      }
    }));
  }
}
```

## Practice Exercises

### Exercise 1: Enterprise Service Architecture
Design and implement a complete enterprise service architecture that:
1. Decomposes a monolithic workflow into microservices
2. Implements proper service communication patterns
3. Includes monitoring and observability
4. Provides automated testing and deployment

### Exercise 2: Multi-Tenant SaaS Platform
Build a multi-tenant N8N automation platform that:
1. Isolates tenant data and workflows
2. Provides tenant-specific customizations
3. Implements proper security and compliance
4. Scales efficiently across tenants

### Exercise 3: Event-Driven Architecture
Create an event-driven system that:
1. Implements event sourcing patterns
2. Provides event replay capabilities
3. Handles eventual consistency
4. Includes comprehensive monitoring

## Next Steps

- Learn [Custom Node Development](./14-custom-nodes.md)
- Master [Database Operations](./15-database-operations.md)
- Explore [Monitoring & Maintenance](./16-monitoring-maintenance.md)

---

Enterprise patterns provide the foundation for building scalable, maintainable automation systems. Master these patterns to create professional-grade solutions!
