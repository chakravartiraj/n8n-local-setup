# Security Best Practices for N8N Workflows

Master essential security practices to build secure, compliant, and trustworthy automation workflows that protect sensitive data and systems.

## Table of Contents
- [Security Fundamentals](#security-fundamentals)
- [Credential Management](#credential-management)
- [Data Protection](#data-protection)
- [Network Security](#network-security)
- [Access Control](#access-control)
- [Audit and Logging](#audit-and-logging)
- [Compliance Frameworks](#compliance-frameworks)
- [Vulnerability Management](#vulnerability-management)
- [Secure Development Lifecycle](#secure-development-lifecycle)
- [Incident Response](#incident-response)

## Security Fundamentals

### Security-First Architecture
```javascript
// Security architecture principles for N8N workflows
const SecurityPrinciples = {
  defenseInDepth: {
    description: 'Multiple layers of security controls',
    implementation: [
      'Network security (firewalls, VPNs)',
      'Application security (authentication, authorization)',
      'Data security (encryption, tokenization)',
      'Infrastructure security (container security, host hardening)'
    ]
  },
  
  leastPrivilege: {
    description: 'Minimum necessary permissions',
    implementation: [
      'Role-based access control (RBAC)',
      'Time-limited access tokens',
      'Resource-specific permissions',
      'Regular access reviews'
    ]
  },
  
  zeroTrust: {
    description: 'Never trust, always verify',
    implementation: [
      'Verify every request',
      'Encrypt all communications',
      'Monitor all activities',
      'Assume breach mentality'
    ]
  }
};

// Security assessment framework
class SecurityAssessment {
  constructor() {
    this.vulnerabilities = [];
    this.riskLevels = {
      CRITICAL: { score: 10, impact: 'System compromise' },
      HIGH: { score: 7, impact: 'Data exposure' },
      MEDIUM: { score: 5, impact: 'Service disruption' },
      LOW: { score: 2, impact: 'Minor information leak' }
    };
  }
  
  assessWorkflow(workflow) {
    const assessment = {
      workflowId: workflow.id,
      timestamp: new Date().toISOString(),
      vulnerabilities: [],
      riskScore: 0,
      recommendations: []
    };
    
    // Check for common security issues
    assessment.vulnerabilities.push(...this.checkCredentialSecurity(workflow));
    assessment.vulnerabilities.push(...this.checkDataHandling(workflow));
    assessment.vulnerabilities.push(...this.checkNetworkSecurity(workflow));
    assessment.vulnerabilities.push(...this.checkAccessControls(workflow));
    
    // Calculate overall risk score
    assessment.riskScore = this.calculateRiskScore(assessment.vulnerabilities);
    
    // Generate recommendations
    assessment.recommendations = this.generateRecommendations(assessment.vulnerabilities);
    
    return assessment;
  }
  
  checkCredentialSecurity(workflow) {
    const issues = [];
    
    // Check for hardcoded credentials
    const hardcodedCreds = this.findHardcodedCredentials(workflow);
    if (hardcodedCreds.length > 0) {
      issues.push({
        type: 'HARDCODED_CREDENTIALS',
        severity: 'CRITICAL',
        description: 'Hardcoded credentials found in workflow',
        locations: hardcodedCreds,
        remediation: 'Use N8N credential system instead of hardcoded values'
      });
    }
    
    // Check for insecure credential storage
    const insecureStorage = this.findInsecureCredentialStorage(workflow);
    if (insecureStorage.length > 0) {
      issues.push({
        type: 'INSECURE_CREDENTIAL_STORAGE',
        severity: 'HIGH',
        description: 'Credentials stored insecurely',
        locations: insecureStorage,
        remediation: 'Encrypt credentials and use secure credential management'
      });
    }
    
    return issues;
  }
  
  checkDataHandling(workflow) {
    const issues = [];
    
    // Check for PII handling
    const piiHandling = this.findPIIHandling(workflow);
    if (piiHandling.length > 0) {
      issues.push({
        type: 'PII_HANDLING',
        severity: 'HIGH',
        description: 'Personal data handling without proper protection',
        locations: piiHandling,
        remediation: 'Implement data anonymization and encryption'
      });
    }
    
    // Check for data leakage
    const dataLeakage = this.findDataLeakageRisks(workflow);
    if (dataLeakage.length > 0) {
      issues.push({
        type: 'DATA_LEAKAGE',
        severity: 'MEDIUM',
        description: 'Potential data leakage in logs or outputs',
        locations: dataLeakage,
        remediation: 'Sanitize logs and implement data masking'
      });
    }
    
    return issues;
  }
  
  checkNetworkSecurity(workflow) {
    const issues = [];
    
    // Check for unencrypted communications
    const unencrypted = this.findUnencryptedCommunications(workflow);
    if (unencrypted.length > 0) {
      issues.push({
        type: 'UNENCRYPTED_COMMUNICATION',
        severity: 'HIGH',
        description: 'Unencrypted network communications',
        locations: unencrypted,
        remediation: 'Use HTTPS/TLS for all external communications'
      });
    }
    
    return issues;
  }
  
  findHardcodedCredentials(workflow) {
    const locations = [];
    const patterns = [
      /password\s*[:=]\s*['"][^'"]+['"]/gi,
      /api[_-]?key\s*[:=]\s*['"][^'"]+['"]/gi,
      /secret\s*[:=]\s*['"][^'"]+['"]/gi,
      /token\s*[:=]\s*['"][^'"]+['"]/gi
    ];
    
    // Check workflow configuration and nodes
    const workflowString = JSON.stringify(workflow);
    
    patterns.forEach(pattern => {
      const matches = workflowString.match(pattern);
      if (matches) {
        locations.push(...matches.map(match => ({
          pattern: pattern.source,
          match: match.substring(0, 50) + '...'
        })));
      }
    });
    
    return locations;
  }
  
  findPIIHandling(workflow) {
    const piiFields = [
      'email', 'phone', 'ssn', 'credit_card', 'passport',
      'address', 'name', 'date_of_birth', 'ip_address'
    ];
    
    const locations = [];
    const workflowString = JSON.stringify(workflow).toLowerCase();
    
    piiFields.forEach(field => {
      if (workflowString.includes(field)) {
        locations.push({
          field: field,
          description: `Potential PII field: ${field}`
        });
      }
    });
    
    return locations;
  }
  
  findUnencryptedCommunications(workflow) {
    const locations = [];
    
    // Check for HTTP (non-HTTPS) URLs
    const httpPattern = /http:\/\/[^\s'"]+/gi;
    const workflowString = JSON.stringify(workflow);
    const matches = workflowString.match(httpPattern);
    
    if (matches) {
      locations.push(...matches.map(url => ({
        url: url,
        description: 'Unencrypted HTTP communication'
      })));
    }
    
    return locations;
  }
  
  calculateRiskScore(vulnerabilities) {
    return vulnerabilities.reduce((total, vuln) => {
      return total + this.riskLevels[vuln.severity].score;
    }, 0);
  }
  
  generateRecommendations(vulnerabilities) {
    const recommendations = [];
    const vulnTypes = [...new Set(vulnerabilities.map(v => v.type))];
    
    if (vulnTypes.includes('HARDCODED_CREDENTIALS')) {
      recommendations.push({
        priority: 'CRITICAL',
        action: 'Implement secure credential management',
        details: 'Move all credentials to N8N credential system with encryption'
      });
    }
    
    if (vulnTypes.includes('PII_HANDLING')) {
      recommendations.push({
        priority: 'HIGH',
        action: 'Implement data protection controls',
        details: 'Add data encryption, anonymization, and access controls'
      });
    }
    
    if (vulnTypes.includes('UNENCRYPTED_COMMUNICATION')) {
      recommendations.push({
        priority: 'HIGH',
        action: 'Enable encryption for all communications',
        details: 'Use HTTPS/TLS for all external API calls and data transfers'
      });
    }
    
    return recommendations;
  }
}

// Usage
const securityAssessment = new SecurityAssessment();
const workflow = {
  id: 'user-workflow',
  nodes: [
    // workflow nodes here
  ]
};

const assessment = securityAssessment.assessWorkflow(workflow);
console.log('Security Assessment:', assessment);
```

### Threat Modeling
```javascript
// Threat modeling framework for N8N workflows
class ThreatModel {
  constructor() {
    this.threats = {
      spoofing: 'Impersonating users or systems',
      tampering: 'Modifying data or code',
      repudiation: 'Denying actions performed',
      informationDisclosure: 'Exposing information to unauthorized users',
      denialOfService: 'Making system unavailable',
      elevationOfPrivilege: 'Gaining unauthorized access'
    };
    
    this.attackVectors = [
      'External APIs', 'User inputs', 'Database connections',
      'File systems', 'Network communications', 'Third-party services'
    ];
  }
  
  analyzeWorkflow(workflow) {
    const model = {
      workflowId: workflow.id,
      dataFlow: this.mapDataFlow(workflow),
      trustBoundaries: this.identifyTrustBoundaries(workflow),
      threats: this.identifyThreats(workflow),
      mitigations: this.suggestMitigations(workflow)
    };
    
    return model;
  }
  
  mapDataFlow(workflow) {
    const dataFlow = {
      sources: [],
      processors: [],
      destinations: [],
      flows: []
    };
    
    workflow.nodes.forEach(node => {
      if (node.type === 'trigger' || node.type === 'webhook') {
        dataFlow.sources.push({
          id: node.id,
          type: node.type,
          trustLevel: this.assessTrustLevel(node)
        });
      } else if (node.type === 'http' || node.type === 'database') {
        dataFlow.destinations.push({
          id: node.id,
          type: node.type,
          trustLevel: this.assessTrustLevel(node)
        });
      } else {
        dataFlow.processors.push({
          id: node.id,
          type: node.type,
          dataOperations: this.identifyDataOperations(node)
        });
      }
    });
    
    return dataFlow;
  }
  
  identifyTrustBoundaries(workflow) {
    const boundaries = [];
    
    // Boundary between internal workflow and external services
    workflow.nodes.forEach(node => {
      if (node.type === 'http' || node.type === 'webhook') {
        boundaries.push({
          type: 'external_service',
          nodeId: node.id,
          description: 'Communication with external service',
          riskLevel: 'HIGH'
        });
      }
      
      if (node.type === 'database') {
        boundaries.push({
          type: 'data_store',
          nodeId: node.id,
          description: 'Access to data storage',
          riskLevel: 'MEDIUM'
        });
      }
    });
    
    return boundaries;
  }
  
  identifyThreats(workflow) {
    const threats = [];
    
    workflow.nodes.forEach(node => {
      // Spoofing threats
      if (node.type === 'webhook' || node.type === 'trigger') {
        threats.push({
          type: 'SPOOFING',
          nodeId: node.id,
          description: 'Unauthorized requests could trigger workflow',
          likelihood: 'MEDIUM',
          impact: 'HIGH'
        });
      }
      
      // Information disclosure threats
      if (node.type === 'http') {
        threats.push({
          type: 'INFORMATION_DISCLOSURE',
          nodeId: node.id,
          description: 'Sensitive data could be exposed in HTTP requests',
          likelihood: 'LOW',
          impact: 'HIGH'
        });
      }
      
      // Tampering threats
      if (node.type === 'database') {
        threats.push({
          type: 'TAMPERING',
          nodeId: node.id,
          description: 'Data could be modified without authorization',
          likelihood: 'LOW',
          impact: 'CRITICAL'
        });
      }
    });
    
    return threats;
  }
  
  suggestMitigations(workflow) {
    const mitigations = [];
    
    // Authentication and authorization
    mitigations.push({
      threat: 'SPOOFING',
      mitigation: 'Implement strong authentication',
      implementation: [
        'Use API keys or OAuth tokens',
        'Validate request signatures',
        'Implement rate limiting'
      ]
    });
    
    // Data protection
    mitigations.push({
      threat: 'INFORMATION_DISCLOSURE',
      mitigation: 'Encrypt sensitive data',
      implementation: [
        'Use HTTPS for all communications',
        'Encrypt data at rest',
        'Implement data masking in logs'
      ]
    });
    
    // Integrity protection
    mitigations.push({
      threat: 'TAMPERING',
      mitigation: 'Implement data integrity controls',
      implementation: [
        'Use database transactions',
        'Implement audit logging',
        'Validate all inputs'
      ]
    });
    
    return mitigations;
  }
  
  assessTrustLevel(node) {
    // Assess trust level based on node configuration
    if (node.authentication === 'oauth' || node.authentication === 'certificate') {
      return 'HIGH';
    } else if (node.authentication === 'apikey' || node.authentication === 'basic') {
      return 'MEDIUM';
    } else {
      return 'LOW';
    }
  }
  
  identifyDataOperations(node) {
    const operations = [];
    
    if (node.parameters && node.parameters.includes('transform')) {
      operations.push('DATA_TRANSFORMATION');
    }
    
    if (node.parameters && node.parameters.includes('filter')) {
      operations.push('DATA_FILTERING');
    }
    
    if (node.parameters && node.parameters.includes('validate')) {
      operations.push('DATA_VALIDATION');
    }
    
    return operations;
  }
}
```

## Credential Management

### Secure Credential Handling
```javascript
// Advanced credential management system
class SecureCredentialManager {
  constructor(config = {}) {
    this.config = {
      encryptionAlgorithm: config.encryptionAlgorithm || 'AES-256-GCM',
      keyRotationInterval: config.keyRotationInterval || 86400000, // 24 hours
      maxCredentialAge: config.maxCredentialAge || 604800000, // 7 days
      ...config
    };
    
    this.credentials = new Map();
    this.accessLog = [];
    this.rotationSchedule = new Map();
  }
  
  async storeCredential(id, credential, metadata = {}) {
    const encryptedCredential = await this.encryptCredential(credential);
    
    const credentialRecord = {
      id: id,
      encrypted: encryptedCredential,
      metadata: {
        ...metadata,
        createdAt: new Date().toISOString(),
        lastAccessed: null,
        accessCount: 0,
        expiresAt: this.calculateExpiration(metadata.ttl)
      }
    };
    
    this.credentials.set(id, credentialRecord);
    
    // Schedule rotation if needed
    if (metadata.rotationEnabled) {
      this.scheduleRotation(id, metadata.rotationInterval);
    }
    
    this.logAccess(id, 'STORE', 'SUCCESS');
  }
  
  async retrieveCredential(id, context = {}) {
    const record = this.credentials.get(id);
    
    if (!record) {
      this.logAccess(id, 'RETRIEVE', 'NOT_FOUND', context);
      throw new Error(`Credential not found: ${id}`);
    }
    
    // Check expiration
    if (record.metadata.expiresAt && new Date() > new Date(record.metadata.expiresAt)) {
      this.logAccess(id, 'RETRIEVE', 'EXPIRED', context);
      throw new Error(`Credential expired: ${id}`);
    }
    
    // Check access permissions
    if (!this.checkAccess(id, context)) {
      this.logAccess(id, 'RETRIEVE', 'ACCESS_DENIED', context);
      throw new Error(`Access denied for credential: ${id}`);
    }
    
    try {
      const decryptedCredential = await this.decryptCredential(record.encrypted);
      
      // Update access metadata
      record.metadata.lastAccessed = new Date().toISOString();
      record.metadata.accessCount++;
      
      this.logAccess(id, 'RETRIEVE', 'SUCCESS', context);
      
      return decryptedCredential;
      
    } catch (error) {
      this.logAccess(id, 'RETRIEVE', 'DECRYPT_FAILED', context);
      throw new Error(`Failed to decrypt credential: ${id}`);
    }
  }
  
  async rotateCredential(id, newCredential) {
    const record = this.credentials.get(id);
    
    if (!record) {
      throw new Error(`Credential not found for rotation: ${id}`);
    }
    
    // Store old credential for rollback
    const backupId = `${id}_backup_${Date.now()}`;
    await this.storeCredential(backupId, record, {
      ...record.metadata,
      isBackup: true,
      originalId: id
    });
    
    // Update with new credential
    await this.storeCredential(id, newCredential, {
      ...record.metadata,
      rotatedAt: new Date().toISOString(),
      previousBackup: backupId
    });
    
    this.logAccess(id, 'ROTATE', 'SUCCESS');
  }
  
  async encryptCredential(credential) {
    // In a real implementation, this would use a proper encryption library
    // For demo purposes, we'll simulate encryption
    const serialized = JSON.stringify(credential);
    const encrypted = Buffer.from(serialized).toString('base64');
    
    return {
      algorithm: this.config.encryptionAlgorithm,
      data: encrypted,
      timestamp: new Date().toISOString()
    };
  }
  
  async decryptCredential(encryptedCredential) {
    // In a real implementation, this would use proper decryption
    try {
      const decrypted = Buffer.from(encryptedCredential.data, 'base64').toString();
      return JSON.parse(decrypted);
    } catch (error) {
      throw new Error('Decryption failed');
    }
  }
  
  checkAccess(credentialId, context) {
    // Implement access control logic
    const record = this.credentials.get(credentialId);
    
    if (!record) return false;
    
    // Check workflow access
    if (record.metadata.allowedWorkflows) {
      if (!record.metadata.allowedWorkflows.includes(context.workflowId)) {
        return false;
      }
    }
    
    // Check user access
    if (record.metadata.allowedUsers) {
      if (!record.metadata.allowedUsers.includes(context.userId)) {
        return false;
      }
    }
    
    // Check time-based access
    if (record.metadata.accessHours) {
      const currentHour = new Date().getHours();
      if (!record.metadata.accessHours.includes(currentHour)) {
        return false;
      }
    }
    
    return true;
  }
  
  scheduleRotation(credentialId, interval) {
    const rotationTime = Date.now() + (interval || this.config.keyRotationInterval);
    
    this.rotationSchedule.set(credentialId, {
      nextRotation: rotationTime,
      interval: interval
    });
  }
  
  calculateExpiration(ttl) {
    if (!ttl) return null;
    return new Date(Date.now() + ttl).toISOString();
  }
  
  logAccess(credentialId, operation, result, context = {}) {
    this.accessLog.push({
      credentialId: credentialId,
      operation: operation,
      result: result,
      timestamp: new Date().toISOString(),
      context: {
        workflowId: context.workflowId,
        userId: context.userId,
        ipAddress: context.ipAddress,
        userAgent: context.userAgent
      }
    });
    
    // Keep only last 1000 log entries
    if (this.accessLog.length > 1000) {
      this.accessLog.shift();
    }
  }
  
  getCredentialAudit(credentialId) {
    return this.accessLog.filter(log => log.credentialId === credentialId);
  }
  
  generateSecurityReport() {
    const report = {
      totalCredentials: this.credentials.size,
      expiredCredentials: this.getExpiredCredentials().length,
      pendingRotations: this.getPendingRotations().length,
      accessViolations: this.getAccessViolations().length,
      recommendations: this.generateRecommendations()
    };
    
    return report;
  }
  
  getExpiredCredentials() {
    const expired = [];
    
    for (const [id, record] of this.credentials.entries()) {
      if (record.metadata.expiresAt && new Date() > new Date(record.metadata.expiresAt)) {
        expired.push(id);
      }
    }
    
    return expired;
  }
  
  getPendingRotations() {
    const pending = [];
    const now = Date.now();
    
    for (const [id, schedule] of this.rotationSchedule.entries()) {
      if (schedule.nextRotation <= now) {
        pending.push(id);
      }
    }
    
    return pending;
  }
  
  getAccessViolations() {
    return this.accessLog.filter(log => 
      log.result === 'ACCESS_DENIED' || 
      log.result === 'EXPIRED' || 
      log.result === 'NOT_FOUND'
    );
  }
  
  generateRecommendations() {
    const recommendations = [];
    
    if (this.getExpiredCredentials().length > 0) {
      recommendations.push({
        type: 'EXPIRED_CREDENTIALS',
        priority: 'HIGH',
        message: 'Remove or renew expired credentials',
        count: this.getExpiredCredentials().length
      });
    }
    
    if (this.getPendingRotations().length > 0) {
      recommendations.push({
        type: 'PENDING_ROTATIONS',
        priority: 'MEDIUM',
        message: 'Execute pending credential rotations',
        count: this.getPendingRotations().length
      });
    }
    
    if (this.getAccessViolations().length > 10) {
      recommendations.push({
        type: 'ACCESS_VIOLATIONS',
        priority: 'HIGH',
        message: 'Investigate unusual access patterns',
        count: this.getAccessViolations().length
      });
    }
    
    return recommendations;
  }
}

// Usage in N8N workflow
const credentialManager = new SecureCredentialManager({
  keyRotationInterval: 24 * 60 * 60 * 1000, // 24 hours
  maxCredentialAge: 7 * 24 * 60 * 60 * 1000  // 7 days
});

// Store a credential securely
await credentialManager.storeCredential('api_key_1', {
  apiKey: 'secret-api-key',
  endpoint: 'https://api.example.com'
}, {
  allowedWorkflows: ['workflow_1', 'workflow_2'],
  rotationEnabled: true,
  ttl: 7 * 24 * 60 * 60 * 1000 // 7 days
});

// Retrieve credential in workflow context
try {
  const credential = await credentialManager.retrieveCredential('api_key_1', {
    workflowId: $workflow.id,
    userId: $executionMode.userId
  });
  
  // Use credential safely
  const response = await fetch(credential.endpoint, {
    headers: {
      'Authorization': `Bearer ${credential.apiKey}`
    }
  });
  
} catch (error) {
  console.error('Credential access failed:', error.message);
}
```

### Token Management
```javascript
// JWT token management for secure API access
class TokenManager {
  constructor(config = {}) {
    this.config = {
      accessTokenTTL: config.accessTokenTTL || 900000,    // 15 minutes
      refreshTokenTTL: config.refreshTokenTTL || 604800000, // 7 days
      issuer: config.issuer || 'n8n-workflow',
      ...config
    };
    
    this.tokens = new Map();
    this.blacklist = new Set();
  }
  
  async generateToken(payload, options = {}) {
    const tokenId = this.generateTokenId();
    const now = Math.floor(Date.now() / 1000);
    
    const tokenPayload = {
      jti: tokenId,
      iss: this.config.issuer,
      sub: payload.subject,
      aud: payload.audience,
      iat: now,
      exp: now + Math.floor((options.ttl || this.config.accessTokenTTL) / 1000),
      ...payload.claims
    };
    
    // In a real implementation, use a proper JWT library
    const token = this.signToken(tokenPayload);
    
    // Store token metadata
    this.tokens.set(tokenId, {
      payload: tokenPayload,
      createdAt: new Date().toISOString(),
      lastUsed: null,
      usageCount: 0
    });
    
    return {
      token: token,
      tokenId: tokenId,
      expiresAt: new Date(tokenPayload.exp * 1000).toISOString()
    };
  }
  
  async validateToken(token) {
    try {
      const payload = this.verifyToken(token);
      
      // Check if token is blacklisted
      if (this.blacklist.has(payload.jti)) {
        throw new Error('Token has been revoked');
      }
      
      // Check expiration
      const now = Math.floor(Date.now() / 1000);
      if (payload.exp < now) {
        throw new Error('Token has expired');
      }
      
      // Update usage statistics
      const tokenData = this.tokens.get(payload.jti);
      if (tokenData) {
        tokenData.lastUsed = new Date().toISOString();
        tokenData.usageCount++;
      }
      
      return {
        valid: true,
        payload: payload,
        tokenId: payload.jti
      };
      
    } catch (error) {
      return {
        valid: false,
        error: error.message
      };
    }
  }
  
  async refreshToken(refreshToken) {
    const validation = await this.validateToken(refreshToken);
    
    if (!validation.valid) {
      throw new Error(`Invalid refresh token: ${validation.error}`);
    }
    
    // Revoke old tokens
    await this.revokeToken(validation.tokenId);
    
    // Generate new access token
    const newToken = await this.generateToken({
      subject: validation.payload.sub,
      audience: validation.payload.aud,
      claims: {
        roles: validation.payload.roles,
        permissions: validation.payload.permissions
      }
    });
    
    return newToken;
  }
  
  async revokeToken(tokenId) {
    this.blacklist.add(tokenId);
    
    // Remove from active tokens
    this.tokens.delete(tokenId);
    
    return {
      revoked: true,
      tokenId: tokenId,
      revokedAt: new Date().toISOString()
    };
  }
  
  signToken(payload) {
    // In a real implementation, use a proper signing algorithm
    // This is a simplified example
    const header = {
      alg: 'HS256',
      typ: 'JWT'
    };
    
    const encodedHeader = Buffer.from(JSON.stringify(header)).toString('base64');
    const encodedPayload = Buffer.from(JSON.stringify(payload)).toString('base64');
    const signature = this.generateSignature(`${encodedHeader}.${encodedPayload}`);
    
    return `${encodedHeader}.${encodedPayload}.${signature}`;
  }
  
  verifyToken(token) {
    const parts = token.split('.');
    if (parts.length !== 3) {
      throw new Error('Invalid token format');
    }
    
    const [header, payload, signature] = parts;
    
    // Verify signature
    const expectedSignature = this.generateSignature(`${header}.${payload}`);
    if (signature !== expectedSignature) {
      throw new Error('Invalid token signature');
    }
    
    // Decode payload
    const decodedPayload = JSON.parse(Buffer.from(payload, 'base64').toString());
    
    return decodedPayload;
  }
  
  generateSignature(data) {
    // Simplified signature generation (use HMAC in real implementation)
    return Buffer.from(data + 'secret-key').toString('base64');
  }
  
  generateTokenId() {
    return `tok_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
  
  cleanupExpiredTokens() {
    const now = Math.floor(Date.now() / 1000);
    const expiredTokens = [];
    
    for (const [tokenId, tokenData] of this.tokens.entries()) {
      if (tokenData.payload.exp < now) {
        expiredTokens.push(tokenId);
      }
    }
    
    expiredTokens.forEach(tokenId => {
      this.tokens.delete(tokenId);
      this.blacklist.add(tokenId);
    });
    
    return {
      cleanedUp: expiredTokens.length,
      expiredTokens: expiredTokens
    };
  }
  
  getTokenStatistics() {
    const now = Math.floor(Date.now() / 1000);
    const activeTokens = Array.from(this.tokens.values())
      .filter(token => token.payload.exp > now);
    
    return {
      totalTokens: this.tokens.size,
      activeTokens: activeTokens.length,
      blacklistedTokens: this.blacklist.size,
      averageUsage: activeTokens.length > 0 
        ? activeTokens.reduce((sum, token) => sum + token.usageCount, 0) / activeTokens.length 
        : 0
    };
  }
}

// Usage in N8N workflow for secure API access
const tokenManager = new TokenManager({
  accessTokenTTL: 15 * 60 * 1000,  // 15 minutes
  refreshTokenTTL: 7 * 24 * 60 * 60 * 1000  // 7 days
});

// Generate token for API access
const apiToken = await tokenManager.generateToken({
  subject: 'workflow_user',
  audience: 'api.example.com',
  claims: {
    workflowId: $workflow.id,
    permissions: ['read', 'write'],
    scope: 'workflow-execution'
  }
});

// Use token in HTTP requests
const processedItems = [];

for (const item of $input.all()) {
  try {
    // Validate token before use
    const validation = await tokenManager.validateToken(apiToken.token);
    
    if (!validation.valid) {
      throw new Error(`Token validation failed: ${validation.error}`);
    }
    
    // Make secure API request
    const response = await fetch(`https://api.example.com/data/${item.json.id}`, {
      headers: {
        'Authorization': `Bearer ${apiToken.token}`,
        'Content-Type': 'application/json'
      }
    });
    
    if (!response.ok) {
      throw new Error(`API request failed: ${response.status}`);
    }
    
    const data = await response.json();
    
    processedItems.push({
      json: {
        ...item.json,
        apiData: data,
        tokenUsed: validation.tokenId
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

// Add token statistics
processedItems.push({
  json: {
    type: 'token_stats',
    stats: tokenManager.getTokenStatistics()
  }
});

return processedItems;
```

## Data Protection

### Data Encryption and Anonymization
```javascript
// Comprehensive data protection system
class DataProtectionService {
  constructor(config = {}) {
    this.config = {
      encryptionAlgorithm: config.encryptionAlgorithm || 'AES-256-GCM',
      hashAlgorithm: config.hashAlgorithm || 'SHA-256',
      saltLength: config.saltLength || 32,
      ...config
    };
    
    this.encryptionKeys = new Map();
    this.dataClassifications = new Map();
  }
  
  classifyData(data) {
    const classification = {
      public: [],
      internal: [],
      confidential: [],
      restricted: []
    };
    
    for (const [key, value] of Object.entries(data)) {
      const dataType = this.identifyDataType(key, value);
      
      switch (dataType.sensitivity) {
        case 'PUBLIC':
          classification.public.push({ key, value, type: dataType.type });
          break;
        case 'INTERNAL':
          classification.internal.push({ key, value, type: dataType.type });
          break;
        case 'CONFIDENTIAL':
          classification.confidential.push({ key, value, type: dataType.type });
          break;
        case 'RESTRICTED':
          classification.restricted.push({ key, value, type: dataType.type });
          break;
      }
    }
    
    return classification;
  }
  
  identifyDataType(key, value) {
    const keyLower = key.toLowerCase();
    
    // PII identification patterns
    const piiPatterns = {
      email: {
        pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
        sensitivity: 'CONFIDENTIAL',
        type: 'EMAIL'
      },
      phone: {
        pattern: /^\+?[\d\s\-\(\)]{10,}$/,
        sensitivity: 'CONFIDENTIAL',
        type: 'PHONE'
      },
      ssn: {
        pattern: /^\d{3}-?\d{2}-?\d{4}$/,
        sensitivity: 'RESTRICTED',
        type: 'SSN'
      },
      creditCard: {
        pattern: /^\d{4}[\s\-]?\d{4}[\s\-]?\d{4}[\s\-]?\d{4}$/,
        sensitivity: 'RESTRICTED',
        type: 'CREDIT_CARD'
      },
      ipAddress: {
        pattern: /^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$/,
        sensitivity: 'INTERNAL',
        type: 'IP_ADDRESS'
      }
    };
    
    // Check field name patterns
    if (keyLower.includes('password') || keyLower.includes('secret')) {
      return { sensitivity: 'RESTRICTED', type: 'SECRET' };
    }
    
    if (keyLower.includes('name') || keyLower.includes('address')) {
      return { sensitivity: 'CONFIDENTIAL', type: 'PII' };
    }
    
    // Check value patterns
    if (typeof value === 'string') {
      for (const [type, config] of Object.entries(piiPatterns)) {
        if (config.pattern.test(value)) {
          return { sensitivity: config.sensitivity, type: config.type };
        }
      }
    }
    
    return { sensitivity: 'PUBLIC', type: 'GENERAL' };
  }
  
  async encryptSensitiveData(data) {
    const classification = this.classifyData(data);
    const encrypted = { ...data };
    
    // Encrypt confidential and restricted data
    const sensitiveFields = [
      ...classification.confidential,
      ...classification.restricted
    ];
    
    for (const field of sensitiveFields) {
      encrypted[field.key] = await this.encryptField(
        field.value, 
        field.type, 
        classification.restricted.includes(field) ? 'RESTRICTED' : 'CONFIDENTIAL'
      );
    }
    
    return {
      encrypted: encrypted,
      classification: classification,
      encryptedFields: sensitiveFields.map(f => f.key)
    };
  }
  
  async decryptSensitiveData(encryptedData, encryptedFields) {
    const decrypted = { ...encryptedData };
    
    for (const fieldKey of encryptedFields) {
      if (encryptedData[fieldKey] && encryptedData[fieldKey].encrypted) {
        decrypted[fieldKey] = await this.decryptField(encryptedData[fieldKey]);
      }
    }
    
    return decrypted;
  }
  
  async encryptField(value, dataType, sensitivity) {
    const key = await this.getEncryptionKey(sensitivity);
    
    // In a real implementation, use proper encryption library
    const encrypted = Buffer.from(JSON.stringify(value)).toString('base64');
    
    return {
      encrypted: true,
      algorithm: this.config.encryptionAlgorithm,
      data: encrypted,
      dataType: dataType,
      sensitivity: sensitivity,
      timestamp: new Date().toISOString()
    };
  }
  
  async decryptField(encryptedField) {
    // In a real implementation, use proper decryption
    try {
      const decrypted = Buffer.from(encryptedField.data, 'base64').toString();
      return JSON.parse(decrypted);
    } catch (error) {
      throw new Error('Decryption failed');
    }
  }
  
  anonymizeData(data, anonymizationLevel = 'STANDARD') {
    const classification = this.classifyData(data);
    const anonymized = { ...data };
    
    // Anonymize based on data type and level
    const fieldsToAnonymize = [
      ...classification.confidential,
      ...classification.restricted
    ];
    
    for (const field of fieldsToAnonymize) {
      anonymized[field.key] = this.anonymizeField(field.value, field.type, anonymizationLevel);
    }
    
    return {
      anonymized: anonymized,
      originalClassification: classification,
      anonymizationLevel: anonymizationLevel
    };
  }
  
  anonymizeField(value, dataType, level) {
    switch (dataType) {
      case 'EMAIL':
        return this.anonymizeEmail(value, level);
      case 'PHONE':
        return this.anonymizePhone(value, level);
      case 'SSN':
        return this.anonymizeSSN(value, level);
      case 'CREDIT_CARD':
        return this.anonymizeCreditCard(value, level);
      case 'PII':
        return this.anonymizePII(value, level);
      default:
        return this.hashValue(value);
    }
  }
  
  anonymizeEmail(email, level) {
    if (level === 'MINIMAL') {
      const [local, domain] = email.split('@');
      const maskedLocal = local.charAt(0) + '*'.repeat(local.length - 2) + local.charAt(local.length - 1);
      return `${maskedLocal}@${domain}`;
    } else {
      return this.hashValue(email);
    }
  }
  
  anonymizePhone(phone, level) {
    if (level === 'MINIMAL') {
      const digits = phone.replace(/\D/g, '');
      return `***-***-${digits.slice(-4)}`;
    } else {
      return this.hashValue(phone);
    }
  }
  
  anonymizeSSN(ssn, level) {
    if (level === 'MINIMAL') {
      return `***-**-${ssn.slice(-4)}`;
    } else {
      return this.hashValue(ssn);
    }
  }
  
  anonymizeCreditCard(card, level) {
    const digits = card.replace(/\D/g, '');
    return `****-****-****-${digits.slice(-4)}`;
  }
  
  anonymizePII(value, level) {
    if (typeof value === 'string') {
      if (level === 'MINIMAL') {
        return value.charAt(0) + '*'.repeat(Math.max(0, value.length - 2)) + 
               (value.length > 1 ? value.charAt(value.length - 1) : '');
      } else {
        return this.hashValue(value);
      }
    }
    return this.hashValue(value);
  }
  
  hashValue(value) {
    // In a real implementation, use proper hashing library
    const stringValue = JSON.stringify(value);
    return Buffer.from(stringValue).toString('base64').substring(0, 16) + '...';
  }
  
  async getEncryptionKey(sensitivity) {
    if (!this.encryptionKeys.has(sensitivity)) {
      // Generate or retrieve encryption key
      this.encryptionKeys.set(sensitivity, `key_${sensitivity}_${Date.now()}`);
    }
    
    return this.encryptionKeys.get(sensitivity);
  }
  
  redactData(data, redactionLevel = 'STANDARD') {
    const classification = this.classifyData(data);
    const redacted = { ...data };
    
    // Redact sensitive fields
    const fieldsToRedact = [
      ...classification.confidential,
      ...classification.restricted
    ];
    
    for (const field of fieldsToRedact) {
      redacted[field.key] = '[REDACTED]';
    }
    
    return {
      redacted: redacted,
      redactedFields: fieldsToRedact.map(f => f.key),
      redactionLevel: redactionLevel
    };
  }
  
  generateDataProcessingReport(data) {
    const classification = this.classifyData(data);
    
    return {
      timestamp: new Date().toISOString(),
      dataClassification: {
        public: classification.public.length,
        internal: classification.internal.length,
        confidential: classification.confidential.length,
        restricted: classification.restricted.length
      },
      riskAssessment: this.assessDataRisk(classification),
      recommendations: this.generateDataRecommendations(classification)
    };
  }
  
  assessDataRisk(classification) {
    let riskScore = 0;
    
    riskScore += classification.restricted.length * 10;  // High risk
    riskScore += classification.confidential.length * 5; // Medium risk
    riskScore += classification.internal.length * 2;     // Low risk
    
    if (riskScore > 50) return 'HIGH';
    if (riskScore > 20) return 'MEDIUM';
    return 'LOW';
  }
  
  generateDataRecommendations(classification) {
    const recommendations = [];
    
    if (classification.restricted.length > 0) {
      recommendations.push({
        priority: 'CRITICAL',
        action: 'Encrypt all restricted data fields',
        fields: classification.restricted.map(f => f.key)
      });
    }
    
    if (classification.confidential.length > 0) {
      recommendations.push({
        priority: 'HIGH',
        action: 'Implement access controls for confidential data',
        fields: classification.confidential.map(f => f.key)
      });
    }
    
    if (classification.internal.length > 5) {
      recommendations.push({
        priority: 'MEDIUM',
        action: 'Consider data minimization for internal fields',
        count: classification.internal.length
      });
    }
    
    return recommendations;
  }
}

// Usage in N8N workflow
const dataProtection = new DataProtectionService();

const processedItems = [];

for (const item of $input.all()) {
  try {
    // Classify and protect sensitive data
    const protectionResult = await dataProtection.encryptSensitiveData(item.json);
    
    // Generate data processing report
    const report = dataProtection.generateDataProcessingReport(item.json);
    
    // Create anonymized version for logging
    const anonymized = dataProtection.anonymizeData(item.json, 'STANDARD');
    
    processedItems.push({
      json: {
        id: item.json.id,
        protectedData: protectionResult.encrypted,
        encryptedFields: protectionResult.encryptedFields,
        dataClassification: protectionResult.classification,
        processingReport: report,
        anonymizedData: anonymized.anonymized
      }
    });
    
  } catch (error) {
    processedItems.push({
      json: {
        error: error.message,
        originalId: item.json.id,
        failed: true
      }
    });
  }
}

return processedItems;
```

## Practice Exercises

### Exercise 1: Security Assessment Tool
Build a comprehensive security assessment tool that:
1. Scans workflows for security vulnerabilities
2. Generates risk assessments and recommendations
3. Tracks security improvements over time
4. Integrates with compliance frameworks

### Exercise 2: Secure Data Pipeline
Create a secure data processing pipeline that:
1. Implements end-to-end encryption
2. Provides data anonymization capabilities
3. Maintains audit trails for all data access
4. Supports multiple compliance standards

### Exercise 3: Incident Response System
Develop an incident response workflow that:
1. Detects security anomalies automatically
2. Implements containment procedures
3. Coordinates response activities
4. Generates incident reports

## Next Steps

- Explore [Enterprise Patterns](./13-enterprise-patterns.md)
- Learn [Custom Node Development](./14-custom-nodes.md)
- Master [Database Operations](./15-database-operations.md)

---

Security is not optional in modern automation systems. Master these practices to build trustworthy workflows that protect your organization's most valuable assets!
