# MCP Patterns and Best Practices

## Table of Contents

1. [Server Design Patterns](#server-design-patterns)
2. [Client Design Patterns](#client-design-patterns)
3. [Security Best Practices](#security-best-practices)
4. [Performance Optimization](#performance-optimization)
5. [Testing and Debugging](#testing-and-debugging)
6. [Deployment Strategies](#deployment-strategies)

## Server Design Patterns

### 1. Adapter Pattern

**Purpose**: Wrap existing services to expose as MCP tools

```javascript
// Email service adapter
class EmailServiceAdapter {
  constructor(emailProvider) {
    this.emailProvider = emailProvider;
  }

  getTool() {
    return {
      name: "send_email",
      description: "Send email through the email service",
      inputSchema: {
        type: "object",
        properties: {
          to: { type: "string" },
          subject: { type: "string" },
          body: { type: "string" },
          htmlBody: { type: "string" },
        },
        required: ["to", "subject", "body"],
      },
    };
  }

  async execute(args) {
    // Adapt MCP parameters to provider API
    const result = await this.emailProvider.send({
      recipients: [args.to],
      subject: args.subject,
      plaintext: args.body,
      html: args.htmlBody,
    });

    return {
      success: result.status === "sent",
      messageId: result.id,
    };
  }
}

// Usage in server
const emailAdapter = new EmailServiceAdapter(emailProvider);

server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "send_email") {
    const result = await emailAdapter.execute(args);
    return {
      content: [{ type: "text", text: JSON.stringify(result) }],
      isError: false,
    };
  }
});
```

### 2. Factory Pattern

**Purpose**: Create tools dynamically based on configuration

```javascript
class ToolFactory {
  static createDatabaseTool(dbConnection) {
    return {
      name: "query_database",
      description: "Execute database queries",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string" },
          parameters: { type: "array" },
        },
        required: ["query"],
      },
    };
  }

  static createAPICalls(endpoint) {
    return {
      name: `api_${endpoint.name}`,
      description: `Call ${endpoint.name} API`,
      inputSchema: endpoint.schema,
    };
  }

  static createToolsFromConfig(config) {
    const tools = [];

    if (config.database) {
      tools.push(this.createDatabaseTool(config.database));
    }

    config.apis?.forEach((api) => {
      tools.push(this.createAPICalls(api));
    });

    return tools;
  }
}

// Usage
const tools = ToolFactory.createToolsFromConfig({
  database: dbConnection,
  apis: [
    { name: "users", schema: {} },
    { name: "products", schema: {} },
  ],
});
```

### 3. Middleware Pattern

**Purpose**: Add cross-cutting concerns like logging, auth, rate limiting

```javascript
class ToolMiddleware {
  constructor(tool, middlewares = []) {
    this.tool = tool;
    this.middlewares = middlewares;
  }

  async execute(args) {
    let result = { args };

    // Execute middleware chain
    for (const middleware of this.middlewares) {
      result = await middleware(result);

      if (result.error) {
        return { error: result.error };
      }
    }

    // Execute actual tool
    return await this.tool.execute(args);
  }
}

// Middleware definitions
const authMiddleware = async (context) => {
  if (!context.token) {
    return { error: "Unauthorized" };
  }
  return context;
};

const loggingMiddleware = async (context) => {
  console.log("Executing with args:", context.args);
  return context;
};

const rateLimitMiddleware = async (context) => {
  // Check rate limit
  if (isRateLimited(context.args)) {
    return { error: "Rate limited" };
  }
  return context;
};

// Apply middleware
const protectedTool = new ToolMiddleware(queryTool, [
  authMiddleware,
  loggingMiddleware,
  rateLimitMiddleware,
]);
```

### 4. Resource Streaming Pattern

**Purpose**: Handle large resources efficiently

```javascript
class StreamingResource {
  static createLargeDatasetResource() {
    return {
      uri: "data://large_dataset",
      name: "Large Dataset",
      mimeType: "application/x-ndjson", // Newline-delimited JSON
      streaming: true,
    };
  }

  static async *streamLargeDataset() {
    const chunkSize = 1000;
    let offset = 0;

    while (true) {
      const rows = await database.query(
        `SELECT * FROM large_table LIMIT ${chunkSize} OFFSET ${offset}`,
      );

      if (rows.length === 0) break;

      // Yield rows as NDJSON (one JSON object per line)
      for (const row of rows) {
        yield JSON.stringify(row) + "\n";
      }

      offset += chunkSize;
    }
  }

  static async readStream(chunkCallback) {
    for await (const chunk of this.streamLargeDataset()) {
      chunkCallback(chunk);
    }
  }
}
```

## Client Design Patterns

### 1. Strategy Pattern

**Purpose**: Switch implementations based on context

```javascript
class ToolExecutionStrategy {
  async execute(toolName, args) {
    throw new Error("Must implement execute");
  }
}

class DirectExecutionStrategy extends ToolExecutionStrategy {
  constructor(client) {
    super();
    this.client = client;
  }

  async execute(toolName, args) {
    return await this.client.callTool(toolName, args);
  }
}

class CachedExecutionStrategy extends ToolExecutionStrategy {
  constructor(client) {
    super();
    this.client = client;
    this.cache = new Map();
  }

  async execute(toolName, args) {
    const key = `${toolName}:${JSON.stringify(args)}`;

    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    const result = await this.client.callTool(toolName, args);
    this.cache.set(key, result);

    return result;
  }
}

class OfflineExecutionStrategy extends ToolExecutionStrategy {
  constructor(client, offlineData) {
    super();
    this.client = client;
    this.offlineData = offlineData;
  }

  async execute(toolName, args) {
    // Try offline first
    if (this.offlineData[toolName]) {
      return this.offlineData[toolName](args);
    }

    // Fall back to online
    return await this.client.callTool(toolName, args);
  }
}

// Usage
class SmartClient {
  constructor(client) {
    this.client = client;
    this.strategies = {
      direct: new DirectExecutionStrategy(client),
      cached: new CachedExecutionStrategy(client),
      offline: new OfflineExecutionStrategy(client, {}),
    };
    this.currentStrategy = "direct";
  }

  setStrategy(strategyName) {
    this.currentStrategy = strategyName;
  }

  async execute(toolName, args) {
    return await this.strategies[this.currentStrategy].execute(toolName, args);
  }
}
```

### 2. Observer Pattern

**Purpose**: Notify multiple listeners of tool execution events

```javascript
class ToolExecutor {
  constructor() {
    this.observers = [];
  }

  subscribe(observer) {
    this.observers.push(observer);
  }

  unsubscribe(observer) {
    this.observers = this.observers.filter((o) => o !== observer);
  }

  async callTool(name, args) {
    // Notify before execution
    for (const observer of this.observers) {
      observer.onBeforeToolCall?.({
        toolName: name,
        arguments: args,
        timestamp: Date.now(),
      });
    }

    const startTime = Date.now();
    let result, error;

    try {
      result = await this.client.callTool(name, args);
    } catch (e) {
      error = e;
    }

    // Notify after execution
    for (const observer of this.observers) {
      observer.onAfterToolCall?.({
        toolName: name,
        arguments: args,
        result,
        error,
        duration: Date.now() - startTime,
        timestamp: Date.now(),
      });
    }

    if (error) throw error;
    return result;
  }
}

// Observers
class LoggingObserver {
  onBeforeToolCall(event) {
    console.log(`[${event.timestamp}] Calling tool: ${event.toolName}`);
  }

  onAfterToolCall(event) {
    console.log(
      `[${event.timestamp}] Tool ${event.toolName} completed in ${event.duration}ms`,
    );
  }
}

class MetricsObserver {
  constructor() {
    this.metrics = new Map();
  }

  onBeforeToolCall(event) {
    if (!this.metrics.has(event.toolName)) {
      this.metrics.set(event.toolName, {
        calls: 0,
        totalTime: 0,
        errors: 0,
      });
    }
  }

  onAfterToolCall(event) {
    const metric = this.metrics.get(event.toolName);
    metric.calls++;
    metric.totalTime += event.duration;
    if (event.error) metric.errors++;
  }

  getMetrics(toolName) {
    const metric = this.metrics.get(toolName);
    return {
      ...metric,
      avgTime: metric.totalTime / metric.calls,
    };
  }
}

// Usage
const executor = new ToolExecutor();
executor.subscribe(new LoggingObserver());
executor.subscribe(new MetricsObserver());
```

### 3. Chain of Responsibility Pattern

**Purpose**: Pass request through chain of handlers

```javascript
class ValidationHandler {
  async handle(toolName, args) {
    // Validate arguments
    if (!this.isValid(toolName, args)) {
      throw new Error(`Invalid arguments for ${toolName}`);
    }

    return await this.next?.handle(toolName, args);
  }

  setNext(handler) {
    this.next = handler;
    return handler;
  }

  isValid(toolName, args) {
    // Implementation
    return true;
  }
}

class AuthenticationHandler {
  async handle(toolName, args) {
    if (!this.isAuthenticated()) {
      throw new Error("Not authenticated");
    }

    return await this.next?.handle(toolName, args);
  }

  setNext(handler) {
    this.next = handler;
    return handler;
  }

  isAuthenticated() {
    return !!this.token;
  }
}

class ExecutionHandler {
  constructor(client) {
    this.client = client;
  }

  async handle(toolName, args) {
    return await this.client.callTool(toolName, args);
  }

  setNext(handler) {
    // Last handler
    return this;
  }
}

// Build chain
const validation = new ValidationHandler();
const auth = new AuthenticationHandler();
const execution = new ExecutionHandler(client);

validation.setNext(auth).setNext(execution);

// Use
const result = await validation.handle("query", { sql: "..." });
```

## Security Best Practices

### 1. Input Validation

```javascript
class InputValidator {
  static validateSQLQuery(query) {
    // Disallow dangerous SQL patterns
    const dangerousPatterns = [
      /drop\s+table/i,
      /delete\s+from/i,
      /truncate/i,
      /alter\s+table/i,
    ];

    for (const pattern of dangerousPatterns) {
      if (pattern.test(query)) {
        throw new Error("Query contains disallowed operations");
      }
    }

    // Limit query length
    if (query.length > 10000) {
      throw new Error("Query too long");
    }

    return true;
  }

  static validateJSON(json) {
    try {
      JSON.parse(json);
      return true;
    } catch {
      throw new Error("Invalid JSON");
    }
  }

  static validateRange(value, min, max) {
    if (value < min || value > max) {
      throw new Error(`Value must be between ${min} and ${max}`);
    }
    return true;
  }
}

// Apply in tool execution
server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "query") {
    try {
      InputValidator.validateSQLQuery(args.query);
      // Execute query
    } catch (error) {
      return {
        content: [{ type: "text", text: error.message }],
        isError: true,
      };
    }
  }
});
```

### 2. Authentication and Authorization

```javascript
class AuthManager {
  constructor() {
    this.tokens = new Map();
  }

  validateToken(token) {
    if (!this.tokens.has(token)) {
      throw new Error("Invalid token");
    }

    const tokenData = this.tokens.get(token);

    if (tokenData.expiry < Date.now()) {
      this.tokens.delete(token);
      throw new Error("Token expired");
    }

    return tokenData;
  }

  canAccessTool(token, toolName) {
    const tokenData = this.validateToken(token);
    return tokenData.permissions?.includes(toolName) ?? false;
  }

  canAccessResource(token, resourceUri) {
    const tokenData = this.validateToken(token);
    return tokenData.resources?.includes(resourceUri) ?? false;
  }

  issueToken(permissions, resources, expiryHours = 24) {
    const token = require("crypto").randomBytes(32).toString("hex");

    this.tokens.set(token, {
      permissions,
      resources,
      expiry: Date.now() + expiryHours * 3600000,
      createdAt: Date.now(),
    });

    return token;
  }
}

// Apply authentication
const authManager = new AuthManager();

server.setRequestHandler("tools/call", async (request) => {
  const { token } = request.context;
  const { name, arguments: args } = request.params;

  // Check authentication
  try {
    const tokenData = authManager.validateToken(token);

    // Check authorization
    if (!authManager.canAccessTool(token, name)) {
      throw new Error(`Not authorized to use tool: ${name}`);
    }

    // Execute tool
  } catch (error) {
    return {
      content: [{ type: "text", text: error.message }],
      isError: true,
    };
  }
});
```

### 3. Rate Limiting

```javascript
class RateLimiter {
  constructor(maxRequests = 100, windowMs = 60000) {
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
    this.requests = new Map();
  }

  checkLimit(clientId) {
    const now = Date.now();
    const key = `${clientId}`;

    if (!this.requests.has(key)) {
      this.requests.set(key, []);
    }

    const times = this.requests.get(key);

    // Remove old entries
    while (times.length > 0 && times[0] < now - this.windowMs) {
      times.shift();
    }

    if (times.length >= this.maxRequests) {
      return false;
    }

    times.push(now);
    return true;
  }
}

// Apply rate limiting
const rateLimiter = new RateLimiter(100, 60000);

server.setRequestHandler("tools/call", async (request) => {
  const clientId = request.context.clientId;

  if (!rateLimiter.checkLimit(clientId)) {
    return {
      content: [{ type: "text", text: "Rate limit exceeded" }],
      isError: true,
    };
  }

  // Continue with tool execution
});
```

## Performance Optimization

### 1. Connection Pooling

```javascript
class ConnectionPool {
  constructor(size = 10) {
    this.size = size;
    this.available = [];
    this.inUse = new Set();
  }

  async acquire() {
    if (this.available.length > 0) {
      return this.available.pop();
    }

    return await new Promise((resolve) => {
      this.waiting = this.waiting || [];
      this.waiting.push(resolve);
    });
  }

  release(connection) {
    this.inUse.delete(connection);

    if (this.waiting?.length > 0) {
      const resolve = this.waiting.shift();
      resolve(connection);
    } else {
      this.available.push(connection);
    }
  }

  async exec(fn) {
    const conn = await this.acquire();
    try {
      return await fn(conn);
    } finally {
      this.release(conn);
    }
  }
}
```

### 2. Caching Strategy

```javascript
class CacheManager {
  constructor(ttl = 3600000) {
    this.cache = new Map();
    this.ttl = ttl;
  }

  set(key, value) {
    this.cache.set(key, {
      value,
      expiry: Date.now() + this.ttl,
    });
  }

  get(key) {
    const entry = this.cache.get(key);

    if (!entry) return null;

    if (entry.expiry < Date.now()) {
      this.cache.delete(key);
      return null;
    }

    return entry.value;
  }

  invalidate(pattern) {
    for (const key of this.cache.keys()) {
      if (pattern.test(key)) {
        this.cache.delete(key);
      }
    }
  }
}
```

## Testing and Debugging

### 1. Mock Server for Testing

```javascript
class MockMCPServer {
  constructor() {
    this.tools = [];
    this.resources = [];
    this.toolResults = new Map();
  }

  addTool(tool) {
    this.tools.push(tool);
  }

  addResource(resource, content) {
    this.resources.push(resource);
    this.resourceContent = this.resourceContent || {};
    this.resourceContent[resource.uri] = content;
  }

  setToolResult(toolName, args, result) {
    const key = `${toolName}:${JSON.stringify(args)}`;
    this.toolResults.set(key, result);
  }

  async handleRequest(method, params) {
    if (method === "tools/list") {
      return { tools: this.tools };
    }

    if (method === "tools/call") {
      const key = `${params.name}:${JSON.stringify(params.arguments)}`;
      const result = this.toolResults.get(key);

      if (!result) {
        throw new Error(`No mock result for ${params.name}`);
      }

      return { content: [{ type: "text", text: JSON.stringify(result) }] };
    }

    if (method === "resources/read") {
      return {
        contents: [
          {
            uri: params.uri,
            text: this.resourceContent[params.uri],
          },
        ],
      };
    }
  }
}

// Test usage
const mockServer = new MockMCPServer();

mockServer.addTool({
  name: "test_tool",
  description: "Test tool",
});

mockServer.setToolResult("test_tool", { arg: "value" }, { result: "success" });
```

### 2. Debug Logging

```javascript
class DebugLogger {
  static formatMessage(data) {
    return JSON.stringify(data, null, 2);
  }

  static logRequest(method, params) {
    console.log(
      `[MCP Request]\nMethod: ${method}\n${this.formatMessage(params)}`,
    );
  }

  static logResponse(method, result) {
    console.log(
      `[MCP Response]\nMethod: ${method}\n${this.formatMessage(result)}`,
    );
  }

  static logError(method, error) {
    console.error(`[MCP Error]\nMethod: ${method}\nError: ${error.message}`);
  }
}
```

## Deployment Strategies

### 1. Docker Container Deployment

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["node", "server.js"]
```

### 2. Environment Configuration

```javascript
const config = {
  development: {
    port: 3000,
    debug: true,
    database: "sqlite:memory",
  },
  production: {
    port: process.env.PORT || 8080,
    debug: false,
    database: process.env.DATABASE_URL,
  },
};

const environment = process.env.NODE_ENV || "development";
module.exports = config[environment];
```

---

**Next Steps**: Review [Complete End-to-End Example](06-mcp-complete-example.md)
