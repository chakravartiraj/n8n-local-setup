# Custom Node Development for N8N

Master the art of creating custom nodes to extend N8N's capabilities and build specialized automation components for your unique requirements.

## Table of Contents
- [Node Development Fundamentals](#node-development-fundamentals)
- [Node Architecture](#node-architecture)
- [Building Your First Node](#building-your-first-node)
- [Advanced Node Features](#advanced-node-features)
- [Testing Custom Nodes](#testing-custom-nodes)
- [Node Configuration](#node-configuration)
- [Error Handling in Nodes](#error-handling-in-nodes)
- [Performance Optimization](#performance-optimization)
- [Publishing and Distribution](#publishing-and-distribution)
- [Community Best Practices](#community-best-practices)

## Node Development Fundamentals

### Understanding N8N Node Architecture
```typescript
// Core node interface and structure
import {
  INodeType,
  INodeTypeDescription,
  IExecuteFunctions,
  INodeExecutionData,
  INodeParameters,
  NodeParameterValue,
  NodeOperationError,
} from 'n8n-workflow';

// Basic node structure
export class CustomNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Custom Node',
    name: 'customNode',
    icon: 'file:customNode.svg',
    group: ['input'],
    version: 1,
    subtitle: '={{$parameter["operation"] + ": " + $parameter["resource"]}}',
    description: 'Custom node for specialized operations',
    defaults: {
      name: 'Custom Node',
    },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [
      {
        name: 'customApi',
        required: true,
      },
    ],
    properties: [
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          {
            name: 'User',
            value: 'user',
          },
          {
            name: 'Project',
            value: 'project',
          },
        ],
        default: 'user',
      },
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: {
          show: {
            resource: ['user'],
          },
        },
        options: [
          {
            name: 'Create',
            value: 'create',
            description: 'Create a new user',
            action: 'Create a user',
          },
          {
            name: 'Get',
            value: 'get',
            description: 'Get user information',
            action: 'Get a user',
          },
          {
            name: 'Update',
            value: 'update',
            description: 'Update user information',
            action: 'Update a user',
          },
          {
            name: 'Delete',
            value: 'delete',
            description: 'Delete a user',
            action: 'Delete a user',
          },
        ],
        default: 'get',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    
    const resource = this.getNodeParameter('resource', 0) as string;
    const operation = this.getNodeParameter('operation', 0) as string;

    for (let i = 0; i < items.length; i++) {
      try {
        let responseData;

        if (resource === 'user') {
          if (operation === 'create') {
            responseData = await this.createUser(i);
          } else if (operation === 'get') {
            responseData = await this.getUser(i);
          } else if (operation === 'update') {
            responseData = await this.updateUser(i);
          } else if (operation === 'delete') {
            responseData = await this.deleteUser(i);
          }
        }

        if (Array.isArray(responseData)) {
          returnData.push(...responseData);
        } else {
          returnData.push({
            json: responseData,
            pairedItem: {
              item: i,
            },
          });
        }

      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: error.message,
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }

  private async createUser(itemIndex: number): Promise<any> {
    const name = this.getNodeParameter('name', itemIndex) as string;
    const email = this.getNodeParameter('email', itemIndex) as string;
    
    // Implementation for creating user
    const credentials = await this.getCredentials('customApi');
    
    const response = await this.helpers.request({
      method: 'POST',
      url: `${credentials.baseUrl}/users`,
      headers: {
        'Authorization': `Bearer ${credentials.token}`,
        'Content-Type': 'application/json',
      },
      body: {
        name,
        email,
      },
      json: true,
    });

    return response;
  }

  private async getUser(itemIndex: number): Promise<any> {
    const userId = this.getNodeParameter('userId', itemIndex) as string;
    
    const credentials = await this.getCredentials('customApi');
    
    const response = await this.helpers.request({
      method: 'GET',
      url: `${credentials.baseUrl}/users/${userId}`,
      headers: {
        'Authorization': `Bearer ${credentials.token}`,
      },
      json: true,
    });

    return response;
  }

  private async updateUser(itemIndex: number): Promise<any> {
    const userId = this.getNodeParameter('userId', itemIndex) as string;
    const updateData = this.getNodeParameter('updateData', itemIndex) as object;
    
    const credentials = await this.getCredentials('customApi');
    
    const response = await this.helpers.request({
      method: 'PUT',
      url: `${credentials.baseUrl}/users/${userId}`,
      headers: {
        'Authorization': `Bearer ${credentials.token}`,
        'Content-Type': 'application/json',
      },
      body: updateData,
      json: true,
    });

    return response;
  }

  private async deleteUser(itemIndex: number): Promise<any> {
    const userId = this.getNodeParameter('userId', itemIndex) as string;
    
    const credentials = await this.getCredentials('customApi');
    
    await this.helpers.request({
      method: 'DELETE',
      url: `${credentials.baseUrl}/users/${userId}`,
      headers: {
        'Authorization': `Bearer ${credentials.token}`,
      },
    });

    return { success: true, deletedUserId: userId };
  }
}
```

### Node Development Environment Setup
```bash
# Create new node development environment
mkdir n8n-custom-nodes
cd n8n-custom-nodes

# Initialize package.json
npm init -y

# Install N8N development dependencies
npm install --save-dev n8n-workflow n8n-core typescript @types/node

# Install additional development tools
npm install --save-dev jest @types/jest ts-jest eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin

# Create TypeScript configuration
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2019",
    "module": "commonjs",
    "lib": ["ES2019"],
    "declaration": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts"]
}
EOF

# Create project structure
mkdir -p src/nodes src/credentials src/test

# Create build script
npm pkg set scripts.build="tsc"
npm pkg set scripts.test="jest"
npm pkg set scripts.lint="eslint src/**/*.ts"
npm pkg set scripts.dev="tsc --watch"
```

### Advanced Node Parameter System
```typescript
// Advanced parameter configuration with dynamic options
import {
  INodeProperties,
  ILoadOptionsFunctions,
  INodePropertyOptions,
  NodePropertyTypes,
} from 'n8n-workflow';

export class AdvancedCustomNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Advanced Custom Node',
    name: 'advancedCustomNode',
    icon: 'file:advancedCustomNode.svg',
    group: ['transform'],
    version: 1,
    description: 'Advanced node with dynamic parameters',
    defaults: {
      name: 'Advanced Custom Node',
    },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [
      {
        name: 'customApi',
        required: true,
      },
    ],
    properties: [
      // Resource selection with dynamic loading
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        typeOptions: {
          loadOptionsMethod: 'getResources',
        },
        default: '',
        description: 'Select the resource to work with',
      },
      
      // Conditional fields based on resource
      {
        displayName: 'Project ID',
        name: 'projectId',
        type: 'options',
        typeOptions: {
          loadOptionsMethod: 'getProjects',
        },
        displayOptions: {
          show: {
            resource: ['project'],
          },
        },
        default: '',
        description: 'Select the project to work with',
      },
      
      // Collection parameter for multiple items
      {
        displayName: 'Fields',
        name: 'fields',
        type: 'collection',
        placeholder: 'Add Field',
        default: {},
        options: [
          {
            displayName: 'Name',
            name: 'name',
            type: 'string',
            default: '',
            description: 'Name of the item',
          },
          {
            displayName: 'Description',
            name: 'description',
            type: 'string',
            typeOptions: {
              rows: 4,
            },
            default: '',
            description: 'Description of the item',
          },
          {
            displayName: 'Tags',
            name: 'tags',
            type: 'string',
            typeOptions: {
              multipleValues: true,
              multipleValueButtonText: 'Add Tag',
            },
            default: [],
            description: 'Tags associated with the item',
          },
          {
            displayName: 'Priority',
            name: 'priority',
            type: 'options',
            options: [
              {
                name: 'Low',
                value: 'low',
              },
              {
                name: 'Medium',
                value: 'medium',
              },
              {
                name: 'High',
                value: 'high',
              },
            ],
            default: 'medium',
            description: 'Priority level',
          },
          {
            displayName: 'Due Date',
            name: 'dueDate',
            type: 'dateTime',
            default: '',
            description: 'Due date for the item',
          },
        ],
      },
      
      // Fixed collection for multiple similar items
      {
        displayName: 'Custom Properties',
        name: 'customProperties',
        type: 'fixedCollection',
        placeholder: 'Add Property',
        typeOptions: {
          multipleValues: true,
        },
        default: {},
        options: [
          {
            name: 'property',
            displayName: 'Property',
            values: [
              {
                displayName: 'Key',
                name: 'key',
                type: 'string',
                default: '',
                description: 'Property key',
              },
              {
                displayName: 'Value',
                name: 'value',
                type: 'string',
                default: '',
                description: 'Property value',
              },
              {
                displayName: 'Type',
                name: 'type',
                type: 'options',
                options: [
                  {
                    name: 'String',
                    value: 'string',
                  },
                  {
                    name: 'Number',
                    value: 'number',
                  },
                  {
                    name: 'Boolean',
                    value: 'boolean',
                  },
                ],
                default: 'string',
                description: 'Property type',
              },
            ],
          },
        ],
      },
      
      // JSON editor for complex data
      {
        displayName: 'Advanced Configuration',
        name: 'advancedConfig',
        type: 'json',
        typeOptions: {
          rows: 10,
        },
        default: '{\n  "option1": "value1",\n  "option2": "value2"\n}',
        description: 'Advanced configuration in JSON format',
      },
      
      // File input
      {
        displayName: 'Input File',
        name: 'inputFile',
        type: 'string',
        typeOptions: {
          password: false,
        },
        displayOptions: {
          show: {
            resource: ['file'],
          },
        },
        default: '',
        description: 'Path to input file',
      },
      
      // Conditional parameter with expressions
      {
        displayName: 'Condition',
        name: 'condition',
        type: 'string',
        default: '={{$json.status === "active"}}',
        description: 'Condition to evaluate (use expressions)',
      },
    ],
  };

  methods = {
    loadOptions: {
      // Load available resources dynamically
      async getResources(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
        const credentials = await this.getCredentials('customApi');
        
        try {
          const response = await this.helpers.request({
            method: 'GET',
            url: `${credentials.baseUrl}/resources`,
            headers: {
              'Authorization': `Bearer ${credentials.token}`,
            },
            json: true,
          });

          return response.map((resource: any) => ({
            name: resource.name,
            value: resource.id,
            description: resource.description,
          }));
        } catch (error) {
          throw new NodeOperationError(
            this.getNode(),
            `Failed to load resources: ${error.message}`,
          );
        }
      },

      // Load projects based on selected resource
      async getProjects(this: ILoadOptionsFunctions): Promise<INodePropertyOptions[]> {
        const credentials = await this.getCredentials('customApi');
        const resourceId = this.getCurrentNodeParameter('resource') as string;
        
        if (!resourceId) {
          return [];
        }

        try {
          const response = await this.helpers.request({
            method: 'GET',
            url: `${credentials.baseUrl}/resources/${resourceId}/projects`,
            headers: {
              'Authorization': `Bearer ${credentials.token}`,
            },
            json: true,
          });

          return response.map((project: any) => ({
            name: project.name,
            value: project.id,
            description: project.description,
          }));
        } catch (error) {
          throw new NodeOperationError(
            this.getNode(),
            `Failed to load projects: ${error.message}`,
          );
        }
      },
    },
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      try {
        const resource = this.getNodeParameter('resource', i) as string;
        const fields = this.getNodeParameter('fields', i) as any;
        const customProperties = this.getNodeParameter('customProperties', i) as any;
        const advancedConfig = this.getNodeParameter('advancedConfig', i) as object;
        const condition = this.getNodeParameter('condition', i) as string;

        // Evaluate condition
        const shouldProcess = this.evaluateExpression(condition, {
          $json: items[i].json,
          $binary: items[i].binary,
          $input: items[i],
        });

        if (!shouldProcess) {
          // Skip this item
          returnData.push({
            json: {
              skipped: true,
              reason: 'Condition not met',
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }

        // Process custom properties
        const processedProperties = this.processCustomProperties(customProperties);
        
        // Merge all data
        const processedData = {
          ...items[i].json,
          resource: resource,
          fields: fields,
          customProperties: processedProperties,
          advancedConfig: advancedConfig,
          processedAt: new Date().toISOString(),
        };

        returnData.push({
          json: processedData,
          pairedItem: {
            item: i,
          },
        });

      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: error.message,
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }

  private processCustomProperties(customProperties: any): Record<string, any> {
    const processed: Record<string, any> = {};
    
    if (customProperties.property && Array.isArray(customProperties.property)) {
      for (const prop of customProperties.property) {
        const { key, value, type } = prop;
        
        switch (type) {
          case 'number':
            processed[key] = parseFloat(value);
            break;
          case 'boolean':
            processed[key] = value.toLowerCase() === 'true';
            break;
          default:
            processed[key] = value;
        }
      }
    }
    
    return processed;
  }
}
```

### Credential Management for Custom Nodes
```typescript
// Custom credential type definition
import {
  IAuthenticateGeneric,
  ICredentialType,
  INodeProperties,
  ICredentialTestRequest,
} from 'n8n-workflow';

export class CustomApiCredentials implements ICredentialType {
  name = 'customApi';
  displayName = 'Custom API';
  documentationUrl = 'https://docs.example.com/api';
  properties: INodeProperties[] = [
    {
      displayName: 'Base URL',
      name: 'baseUrl',
      type: 'string',
      default: 'https://api.example.com',
      description: 'Base URL of the API',
    },
    {
      displayName: 'API Token',
      name: 'token',
      type: 'string',
      typeOptions: {
        password: true,
      },
      default: '',
      description: 'API authentication token',
    },
    {
      displayName: 'API Version',
      name: 'version',
      type: 'options',
      options: [
        {
          name: 'v1',
          value: 'v1',
        },
        {
          name: 'v2',
          value: 'v2',
        },
      ],
      default: 'v1',
      description: 'API version to use',
    },
  ];

  // Authentication method
  authenticate: IAuthenticateGeneric = {
    type: 'generic',
    properties: {
      headers: {
        'Authorization': '=Bearer {{$credentials.token}}',
        'X-API-Version': '={{$credentials.version}}',
      },
    },
  };

  // Test the credentials
  test: ICredentialTestRequest = {
    request: {
      baseURL: '={{$credentials.baseUrl}}',
      url: '/auth/test',
      method: 'GET',
    },
  };
}

// OAuth2 credential example
export class CustomOAuth2Credentials implements ICredentialType {
  name = 'customOAuth2';
  displayName = 'Custom OAuth2';
  documentationUrl = 'https://docs.example.com/oauth';
  extends = ['oAuth2Api'];
  properties: INodeProperties[] = [
    {
      displayName: 'Grant Type',
      name: 'grantType',
      type: 'hidden',
      default: 'authorizationCode',
    },
    {
      displayName: 'Authorization URL',
      name: 'authUrl',
      type: 'hidden',
      default: 'https://api.example.com/oauth/authorize',
    },
    {
      displayName: 'Access Token URL',
      name: 'accessTokenUrl',
      type: 'hidden',
      default: 'https://api.example.com/oauth/token',
    },
    {
      displayName: 'Scope',
      name: 'scope',
      type: 'hidden',
      default: 'read write',
    },
    {
      displayName: 'Auth URI Query Parameters',
      name: 'authQueryParameters',
      type: 'hidden',
      default: 'response_type=code',
    },
    {
      displayName: 'Authentication',
      name: 'authentication',
      type: 'hidden',
      default: 'body',
    },
  ];
}
```

### Advanced Node Features Implementation
```typescript
// Node with streaming support and binary data handling
import {
  IBinaryData,
  IBinaryKeyData,
  IDataObject,
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  NodeOperationError,
} from 'n8n-workflow';

export class StreamingNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Streaming Node',
    name: 'streamingNode',
    icon: 'file:streamingNode.svg',
    group: ['transform'],
    version: 1,
    description: 'Node with streaming and binary data support',
    defaults: {
      name: 'Streaming Node',
    },
    inputs: ['main'],
    outputs: ['main'],
    properties: [
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        options: [
          {
            name: 'Stream Process',
            value: 'streamProcess',
            description: 'Process data in streaming mode',
          },
          {
            name: 'Binary Transform',
            value: 'binaryTransform',
            description: 'Transform binary data',
          },
          {
            name: 'Batch Process',
            value: 'batchProcess',
            description: 'Process data in batches',
          },
        ],
        default: 'streamProcess',
      },
      {
        displayName: 'Batch Size',
        name: 'batchSize',
        type: 'number',
        displayOptions: {
          show: {
            operation: ['batchProcess'],
          },
        },
        default: 100,
        description: 'Number of items to process in each batch',
      },
      {
        displayName: 'Binary Property',
        name: 'binaryProperty',
        type: 'string',
        displayOptions: {
          show: {
            operation: ['binaryTransform'],
          },
        },
        default: 'data',
        description: 'Name of the binary property to transform',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const operation = this.getNodeParameter('operation', 0) as string;

    switch (operation) {
      case 'streamProcess':
        return await this.streamProcess(items);
      case 'binaryTransform':
        return await this.binaryTransform(items);
      case 'batchProcess':
        return await this.batchProcess(items);
      default:
        throw new NodeOperationError(this.getNode(), `Unknown operation: ${operation}`);
    }
  }

  private async streamProcess(items: INodeExecutionData[]): Promise<INodeExecutionData[][]> {
    const returnData: INodeExecutionData[] = [];
    
    // Process items in streaming fashion
    for (let i = 0; i < items.length; i++) {
      const item = items[i];
      
      try {
        // Simulate streaming processing
        const processedData = await this.processStreamItem(item.json, i);
        
        returnData.push({
          json: processedData,
          pairedItem: {
            item: i,
          },
        });

        // Yield control periodically to prevent blocking
        if (i % 10 === 0) {
          await new Promise(resolve => setImmediate(resolve));
        }

      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: error.message,
              originalData: item.json,
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }

  private async binaryTransform(items: INodeExecutionData[]): Promise<INodeExecutionData[][]> {
    const binaryProperty = this.getNodeParameter('binaryProperty', 0) as string;
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      const item = items[i];
      
      if (!item.binary || !item.binary[binaryProperty]) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: `Binary property '${binaryProperty}' not found`,
              originalData: item.json,
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }
        throw new NodeOperationError(
          this.getNode(),
          `Binary property '${binaryProperty}' not found`,
          { itemIndex: i }
        );
      }

      try {
        const binaryData = item.binary[binaryProperty];
        const transformedBinary = await this.transformBinaryData(binaryData);
        
        const newBinary: IBinaryKeyData = {};
        newBinary[binaryProperty] = transformedBinary;

        returnData.push({
          json: {
            ...item.json,
            transformed: true,
            originalSize: binaryData.fileSize,
            newSize: transformedBinary.fileSize,
          },
          binary: newBinary,
          pairedItem: {
            item: i,
          },
        });

      } catch (error) {
        if (this.continueOnFail()) {
          returnData.push({
            json: {
              error: error.message,
              originalData: item.json,
            },
            pairedItem: {
              item: i,
            },
          });
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }

  private async batchProcess(items: INodeExecutionData[]): Promise<INodeExecutionData[][]> {
    const batchSize = this.getNodeParameter('batchSize', 0) as number;
    const returnData: INodeExecutionData[] = [];

    // Process items in batches
    for (let i = 0; i < items.length; i += batchSize) {
      const batch = items.slice(i, i + batchSize);
      
      try {
        const batchResults = await this.processBatch(batch, i);
        returnData.push(...batchResults);

        // Add delay between batches to prevent overwhelming the system
        if (i + batchSize < items.length) {
          await new Promise(resolve => setTimeout(resolve, 100));
        }

      } catch (error) {
        if (this.continueOnFail()) {
          // Add error entries for the entire batch
          for (let j = 0; j < batch.length; j++) {
            returnData.push({
              json: {
                error: error.message,
                batchIndex: Math.floor(i / batchSize),
                originalData: batch[j].json,
              },
              pairedItem: {
                item: i + j,
              },
            });
          }
          continue;
        }
        throw error;
      }
    }

    return [returnData];
  }

  private async processStreamItem(data: IDataObject, index: number): Promise<IDataObject> {
    // Simulate stream processing
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({
          ...data,
          processed: true,
          processedAt: new Date().toISOString(),
          index: index,
          streamId: `stream_${Date.now()}_${index}`,
        });
      }, 10);
    });
  }

  private async transformBinaryData(binaryData: IBinaryData): Promise<IBinaryData> {
    // Get binary data buffer
    const binaryBuffer = await this.helpers.getBinaryDataBuffer(
      binaryData.id as string
    );

    // Simulate binary transformation (e.g., image processing, text extraction)
    const transformedBuffer = Buffer.from(binaryBuffer.toString('base64'), 'base64');
    
    // Create new binary data
    const newBinaryData = await this.helpers.prepareBinaryData(
      transformedBuffer,
      `transformed_${binaryData.fileName}`,
      binaryData.mimeType
    );

    return newBinaryData;
  }

  private async processBatch(
    batch: INodeExecutionData[], 
    startIndex: number
  ): Promise<INodeExecutionData[]> {
    const batchResults: INodeExecutionData[] = [];

    // Simulate batch processing (e.g., bulk API call)
    const batchData = batch.map(item => item.json);
    
    const processedBatch = await this.simulateBatchAPI(batchData);

    // Map results back to individual items
    for (let i = 0; i < batch.length; i++) {
      batchResults.push({
        json: {
          ...batch[i].json,
          batchResult: processedBatch[i],
          batchId: `batch_${Date.now()}_${Math.floor(startIndex / batch.length)}`,
          processedAt: new Date().toISOString(),
        },
        pairedItem: {
          item: startIndex + i,
        },
      });
    }

    return batchResults;
  }

  private async simulateBatchAPI(batchData: IDataObject[]): Promise<any[]> {
    // Simulate batch API processing
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve(batchData.map((item, index) => ({
          id: `processed_${index}`,
          status: 'success',
          originalId: item.id,
        })));
      }, 100);
    });
  }
}
```

### Testing Framework for Custom Nodes
```typescript
// Comprehensive testing framework for custom nodes
import { IExecuteFunctions, INodeExecutionData } from 'n8n-workflow';
import { CustomNode } from '../nodes/CustomNode';

// Mock execution context
class MockExecuteFunctions implements Partial<IExecuteFunctions> {
  private inputData: INodeExecutionData[] = [];
  private parameters: Record<string, any> = {};
  private credentials: Record<string, any> = {};
  private nodeParameters: Record<string, any> = {};

  constructor(
    inputData: INodeExecutionData[] = [],
    parameters: Record<string, any> = {},
    credentials: Record<string, any> = {}
  ) {
    this.inputData = inputData;
    this.parameters = parameters;
    this.credentials = credentials;
  }

  getInputData(): INodeExecutionData[] {
    return this.inputData;
  }

  getNodeParameter(parameterName: string, itemIndex: number, fallbackValue?: any): any {
    return this.parameters[parameterName] ?? fallbackValue;
  }

  async getCredentials(type: string): Promise<any> {
    return this.credentials[type] || {};
  }

  continueOnFail(): boolean {
    return this.parameters.continueOnFail || false;
  }

  getNode() {
    return {
      name: 'Test Node',
      type: 'test-node',
      typeVersion: 1,
    };
  }

  helpers = {
    request: jest.fn(),
    prepareBinaryData: jest.fn(),
    getBinaryDataBuffer: jest.fn(),
  };
}

// Test suite for custom nodes
describe('CustomNode', () => {
  let customNode: CustomNode;
  let mockExecuteFunctions: MockExecuteFunctions;

  beforeEach(() => {
    customNode = new CustomNode();
    mockExecuteFunctions = new MockExecuteFunctions();
  });

  describe('execute method', () => {
    test('should create user successfully', async () => {
      const inputData = [
        {
          json: {
            name: 'John Doe',
            email: 'john@example.com',
          },
        },
      ];

      const parameters = {
        resource: 'user',
        operation: 'create',
        name: 'John Doe',
        email: 'john@example.com',
      };

      const credentials = {
        customApi: {
          baseUrl: 'https://api.example.com',
          token: 'test-token',
        },
      };

      mockExecuteFunctions = new MockExecuteFunctions(inputData, parameters, credentials);

      // Mock successful API response
      mockExecuteFunctions.helpers.request.mockResolvedValue({
        id: '123',
        name: 'John Doe',
        email: 'john@example.com',
        createdAt: '2023-01-01T00:00:00Z',
      });

      const result = await customNode.execute.call(mockExecuteFunctions);

      expect(result).toHaveLength(1);
      expect(result[0]).toHaveLength(1);
      expect(result[0][0].json).toEqual({
        id: '123',
        name: 'John Doe',
        email: 'john@example.com',
        createdAt: '2023-01-01T00:00:00Z',
      });

      expect(mockExecuteFunctions.helpers.request).toHaveBeenCalledWith({
        method: 'POST',
        url: 'https://api.example.com/users',
        headers: {
          'Authorization': 'Bearer test-token',
          'Content-Type': 'application/json',
        },
        body: {
          name: 'John Doe',
          email: 'john@example.com',
        },
        json: true,
      });
    });

    test('should handle API errors gracefully', async () => {
      const inputData = [
        {
          json: {
            name: 'John Doe',
            email: 'invalid-email',
          },
        },
      ];

      const parameters = {
        resource: 'user',
        operation: 'create',
        name: 'John Doe',
        email: 'invalid-email',
        continueOnFail: true,
      };

      const credentials = {
        customApi: {
          baseUrl: 'https://api.example.com',
          token: 'test-token',
        },
      };

      mockExecuteFunctions = new MockExecuteFunctions(inputData, parameters, credentials);

      // Mock API error
      mockExecuteFunctions.helpers.request.mockRejectedValue(
        new Error('Invalid email format')
      );

      const result = await customNode.execute.call(mockExecuteFunctions);

      expect(result).toHaveLength(1);
      expect(result[0]).toHaveLength(1);
      expect(result[0][0].json).toEqual({
        error: 'Invalid email format',
      });
    });

    test('should process multiple items', async () => {
      const inputData = [
        { json: { name: 'User 1', email: 'user1@example.com' } },
        { json: { name: 'User 2', email: 'user2@example.com' } },
        { json: { name: 'User 3', email: 'user3@example.com' } },
      ];

      const parameters = {
        resource: 'user',
        operation: 'create',
      };

      const credentials = {
        customApi: {
          baseUrl: 'https://api.example.com',
          token: 'test-token',
        },
      };

      mockExecuteFunctions = new MockExecuteFunctions(inputData, parameters, credentials);

      // Mock successful API responses
      mockExecuteFunctions.helpers.request
        .mockResolvedValueOnce({ id: '1', name: 'User 1', email: 'user1@example.com' })
        .mockResolvedValueOnce({ id: '2', name: 'User 2', email: 'user2@example.com' })
        .mockResolvedValueOnce({ id: '3', name: 'User 3', email: 'user3@example.com' });

      const result = await customNode.execute.call(mockExecuteFunctions);

      expect(result).toHaveLength(1);
      expect(result[0]).toHaveLength(3);
      
      expect(result[0][0].json.id).toBe('1');
      expect(result[0][1].json.id).toBe('2');
      expect(result[0][2].json.id).toBe('3');

      expect(mockExecuteFunctions.helpers.request).toHaveBeenCalledTimes(3);
    });
  });

  describe('parameter validation', () => {
    test('should validate required parameters', () => {
      const description = customNode.description;
      
      expect(description.properties).toBeDefined();
      expect(description.properties.length).toBeGreaterThan(0);
      
      const resourceParam = description.properties.find(p => p.name === 'resource');
      expect(resourceParam).toBeDefined();
      expect(resourceParam.type).toBe('options');
      
      const operationParam = description.properties.find(p => p.name === 'operation');
      expect(operationParam).toBeDefined();
      expect(operationParam.type).toBe('options');
    });

    test('should have proper display options', () => {
      const description = customNode.description;
      const operationParam = description.properties.find(p => p.name === 'operation');
      
      expect(operationParam.displayOptions).toBeDefined();
      expect(operationParam.displayOptions.show).toBeDefined();
      expect(operationParam.displayOptions.show.resource).toEqual(['user']);
    });
  });
});

// Integration tests
describe('CustomNode Integration', () => {
  test('should integrate with actual N8N workflow', async () => {
    // This would be an integration test that runs the node
    // in a real N8N environment
    
    const workflowData = {
      nodes: [
        {
          name: 'Custom Node',
          type: 'customNode',
          parameters: {
            resource: 'user',
            operation: 'create',
            name: 'Integration Test User',
            email: 'integration@example.com',
          },
          credentials: {
            customApi: 'test-credentials',
          },
        },
      ],
      connections: {},
    };

    // Execute workflow and verify results
    // This would require setting up a test N8N instance
  });
});

// Performance tests
describe('CustomNode Performance', () => {
  test('should handle large datasets efficiently', async () => {
    const largeInputData = Array.from({ length: 1000 }, (_, i) => ({
      json: {
        name: `User ${i}`,
        email: `user${i}@example.com`,
      },
    }));

    const mockExecuteFunctions = new MockExecuteFunctions(
      largeInputData,
      { resource: 'user', operation: 'create' },
      { customApi: { baseUrl: 'https://api.example.com', token: 'test-token' } }
    );

    const customNode = new CustomNode();

    // Mock API responses
    mockExecuteFunctions.helpers.request.mockImplementation(() =>
      Promise.resolve({ id: Math.random().toString(36), success: true })
    );

    const startTime = Date.now();
    const result = await customNode.execute.call(mockExecuteFunctions);
    const endTime = Date.now();

    const executionTime = endTime - startTime;
    
    expect(result[0]).toHaveLength(1000);
    expect(executionTime).toBeLessThan(5000); // Should complete within 5 seconds
  });

  test('should have acceptable memory usage', async () => {
    const memoryBefore = process.memoryUsage().heapUsed;
    
    const largeInputData = Array.from({ length: 10000 }, (_, i) => ({
      json: {
        name: `User ${i}`,
        email: `user${i}@example.com`,
        data: new Array(100).fill('x').join(''), // Some data to make it larger
      },
    }));

    const mockExecuteFunctions = new MockExecuteFunctions(
      largeInputData,
      { resource: 'user', operation: 'create' },
      { customApi: { baseUrl: 'https://api.example.com', token: 'test-token' } }
    );

    const customNode = new CustomNode();

    mockExecuteFunctions.helpers.request.mockImplementation(() =>
      Promise.resolve({ id: Math.random().toString(36), success: true })
    );

    await customNode.execute.call(mockExecuteFunctions);

    const memoryAfter = process.memoryUsage().heapUsed;
    const memoryDiff = memoryAfter - memoryBefore;
    
    // Memory usage should be reasonable (less than 100MB for this test)
    expect(memoryDiff).toBeLessThan(100 * 1024 * 1024);
  });
});
```

## Practice Exercises

### Exercise 1: Build a Custom Database Node
Create a custom node that:
1. Connects to multiple database types
2. Supports complex queries with parameterization
3. Implements connection pooling
4. Includes comprehensive error handling

### Exercise 2: Create a Machine Learning Node
Develop a node that:
1. Integrates with ML APIs (OpenAI, Hugging Face)
2. Supports different model types and configurations
3. Handles large text inputs efficiently
4. Implements caching for API responses

### Exercise 3: Build a File Processing Node
Design a node that:
1. Processes various file formats (CSV, JSON, XML, PDF)
2. Supports streaming for large files
3. Includes data validation and transformation
4. Handles binary data efficiently

## Next Steps

- Master [Database Operations](./15-database-operations.md)
- Explore [Monitoring & Maintenance](./16-monitoring-maintenance.md)
- Contribute to the N8N community

---

Custom node development opens unlimited possibilities for extending N8N. Master these skills to create powerful, reusable automation components!
