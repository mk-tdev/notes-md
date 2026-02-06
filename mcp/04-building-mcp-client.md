# Building MCP Clients

## Table of Contents

1. [Client Architecture](#client-architecture)
2. [Client Types](#client-types)
3. [Basic Client Implementation](#basic-client-implementation)
4. [Advanced Client Features](#advanced-client-features)
5. [Integration with AI Models](#integration-with-ai-models)
6. [Error Handling and Resilience](#error-handling-and-resilience)

## Client Architecture

### What is an MCP Client?

An MCP Client is a **component that connects to MCP servers** and translates high-level requests into protocol messages.

### Client Responsibilities

```
High-Level Application
    ↓
Client API Layer
├─ connect()
├─ discoverTools()
├─ callTool()
├─ readResource()
├─ getPrompt()
└─ disconnect()
    ↓
Transport Layer
├─ stdio
├─ HTTP
└─ SSE
    ↓
MCP Server
```

### Key Characteristics

- **Connection Management**: Handles establishing and maintaining server connections
- **Message Serialization**: Converts to/from JSON-RPC format
- **Request Tracking**: Correlates requests with responses using IDs
- **Error Handling**: Gracefully handles timeouts and errors
- **Resource Caching**: Optionally caches tool/resource definitions
- **Type Safety**: Optional TypeScript support for type checking

## Client Types

### 1. Minimal Client

**Use Case**: Simple scripts or debugging

```javascript
// Minimal stdio-based client
const { exec } = require("child_process");

function createMinimalClient() {
  return {
    async query(method, params) {
      // Direct connection - for testing only
      const proc = exec("node server.js");
      // Send message, get response
    },
  };
}
```

### 2. Production Client

**Use Case**: Enterprise applications, cloud deployments

- Full error handling
- Retry logic and backoff
- Connection pooling
- Comprehensive logging
- Health checks

### 3. Web Client

**Use Case**: Browser-based AI assistants

- HTTP/SSE transport
- Request batching
- UI feedback and progress
- Offline support with local caching

## Basic Client Implementation

### Step 1: Create Client Class

```javascript
// client.js
const { Client } = require("@modelcontextprotocol/sdk/client/index");
const {
  StdioClientTransport,
} = require("@modelcontextprotocol/sdk/client/stdio");

class MCPClient {
  constructor(options = {}) {
    this.options = {
      timeout: 30000,
      retries: 3,
      ...options,
    };

    this.client = null;
    this.transport = null;
    this.connected = false;
    this.tools = {};
    this.resources = {};
  }

  async connect(command, args = []) {
    try {
      // Create transport
      this.transport = new StdioClientTransport({
        command,
        args,
      });

      // Create client
      this.client = new Client(
        {
          name: "MyAIApp",
          version: "1.0.0",
        },
        {
          capabilities: {
            sampling: {},
          },
        },
      );

      // Connect
      await this.client.connect(this.transport);

      // Send initialization notification
      await this.client.notification({
        method: "notifications/initialized",
        params: {},
      });

      this.connected = true;
      console.log("Connected to MCP server");
    } catch (error) {
      throw new Error(`Failed to connect: ${error.message}`);
    }
  }

  async discoverCapabilities() {
    if (!this.connected) throw new Error("Not connected");

    try {
      // Get tools
      const toolsResponse = await this.client.request(
        {
          method: "tools/list",
          params: {},
        },
        this.options.timeout,
      );

      this.tools = Object.fromEntries(
        toolsResponse.tools.map((t) => [t.name, t]),
      );

      // Get resources
      const resourcesResponse = await this.client.request(
        {
          method: "resources/list",
          params: {},
        },
        this.options.timeout,
      );

      this.resources = Object.fromEntries(
        resourcesResponse.resources.map((r) => [r.uri, r]),
      );

      console.log(`Discovered ${Object.keys(this.tools).length} tools`);
      console.log(`Discovered ${Object.keys(this.resources).length} resources`);

      return {
        tools: this.tools,
        resources: this.resources,
      };
    } catch (error) {
      throw new Error(`Failed to discover capabilities: ${error.message}`);
    }
  }

  async callTool(name, args) {
    if (!this.connected) throw new Error("Not connected");

    if (!this.tools[name]) {
      throw new Error(`Tool not found: ${name}`);
    }

    let lastError;

    for (let attempt = 0; attempt < this.options.retries; attempt++) {
      try {
        const response = await this.client.request(
          {
            method: "tools/call",
            params: {
              name,
              arguments: args,
            },
          },
          this.options.timeout,
        );

        if (response.isError) {
          throw new Error(response.content[0]?.text || "Tool returned error");
        }

        return response.content[0]?.text;
      } catch (error) {
        lastError = error;

        if (attempt < this.options.retries - 1) {
          const delay = Math.pow(2, attempt) * 1000;
          console.warn(`Retrying ${name} after ${delay}ms...`);
          await new Promise((resolve) => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError;
  }

  async readResource(uri) {
    if (!this.connected) throw new Error("Not connected");

    if (!this.resources[uri]) {
      throw new Error(`Resource not found: ${uri}`);
    }

    try {
      const response = await this.client.request(
        {
          method: "resources/read",
          params: { uri },
        },
        this.options.timeout,
      );

      return response.contents[0]?.text;
    } catch (error) {
      throw new Error(`Failed to read resource: ${error.message}`);
    }
  }

  async listTools() {
    return Object.values(this.tools).map((t) => ({
      name: t.name,
      description: t.description,
    }));
  }

  async listResources() {
    return Object.values(this.resources).map((r) => ({
      uri: r.uri,
      name: r.name,
      description: r.description,
    }));
  }

  async disconnect() {
    if (this.transport) {
      await this.transport.close();
      this.connected = false;
      console.log("Disconnected from MCP server");
    }
  }
}

module.exports = MCPClient;
```

### Step 2: Use the Client

```javascript
// app.js
const MCPClient = require("./client");

async function main() {
  const client = new MCPClient({
    timeout: 30000,
    retries: 3,
  });

  try {
    // Connect to server
    await client.connect("node", ["./server.js"]);

    // Discover what's available
    const capabilities = await client.discoverCapabilities();

    console.log("Available tools:");
    console.log(await client.listTools());

    // Call a tool
    const result = await client.callTool("query", {
      sql: "SELECT * FROM users LIMIT 5",
    });

    console.log("Query result:", result);

    // Read a resource
    const schema = await client.readResource("file:///schemas/database.json");
    console.log("Schema:", schema);
  } catch (error) {
    console.error("Error:", error.message);
  } finally {
    await client.disconnect();
  }
}

main();
```

## Advanced Client Features

### 1. Tool Result Caching

```javascript
class CachingClient extends MCPClient {
  constructor(options = {}) {
    super(options);
    this.cache = new Map();
    this.cacheExpiry = options.cacheExpiry || 300000; // 5 minutes
  }

  async callTool(name, args) {
    const cacheKey = `${name}:${JSON.stringify(args)}`;

    // Check cache
    if (this.cache.has(cacheKey)) {
      const { result, timestamp } = this.cache.get(cacheKey);

      if (Date.now() - timestamp < this.cacheExpiry) {
        console.log("Cache hit for", cacheKey);
        return result;
      }
    }

    // Call tool
    const result = await super.callTool(name, args);

    // Store in cache
    this.cache.set(cacheKey, {
      result,
      timestamp: Date.now(),
    });

    return result;
  }

  clearCache() {
    this.cache.clear();
  }
}
```

### 2. Connection Pool

```javascript
class ClientPool {
  constructor(size = 5) {
    this.size = size;
    this.clients = [];
    this.available = [];
    this.waiting = [];
  }

  async initialize(command, args) {
    for (let i = 0; i < this.size; i++) {
      const client = new MCPClient();
      await client.connect(command, args);
      await client.discoverCapabilities();

      this.clients.push(client);
      this.available.push(client);
    }
  }

  async acquire() {
    if (this.available.length > 0) {
      return this.available.pop();
    }

    return new Promise((resolve) => {
      this.waiting.push(resolve);
    });
  }

  release(client) {
    if (this.waiting.length > 0) {
      const resolve = this.waiting.shift();
      resolve(client);
    } else {
      this.available.push(client);
    }
  }

  async executeWithClient(fn) {
    const client = await this.acquire();
    try {
      return await fn(client);
    } finally {
      this.release(client);
    }
  }

  async shutdown() {
    for (const client of this.clients) {
      await client.disconnect();
    }
  }
}

// Usage
const pool = new ClientPool(5);
await pool.initialize("node", ["./server.js"]);

const result = await pool.executeWithClient(async (client) => {
  return await client.callTool("query", { sql: "SELECT 1" });
});

await pool.shutdown();
```

### 3. Request Batching

```javascript
class BatchingClient extends MCPClient {
  constructor(options = {}) {
    super(options);
    this.batchSize = options.batchSize || 10;
    this.batchDelay = options.batchDelay || 100;
    this.batch = [];
    this.batchTimer = null;
  }

  async callToolBatched(name, args) {
    return new Promise((resolve, reject) => {
      this.batch.push({ name, args, resolve, reject });

      if (this.batch.length >= this.batchSize) {
        this.processBatch();
      } else if (!this.batchTimer) {
        this.batchTimer = setTimeout(
          () => this.processBatch(),
          this.batchDelay,
        );
      }
    });
  }

  async processBatch() {
    if (this.batchTimer) {
      clearTimeout(this.batchTimer);
      this.batchTimer = null;
    }

    if (this.batch.length === 0) return;

    const batch = this.batch.splice(0, this.batchSize);

    try {
      const promises = batch.map(({ name, args }) =>
        super.callTool(name, args).catch((e) => ({ error: e })),
      );

      const results = await Promise.all(promises);

      batch.forEach((item, index) => {
        if (results[index].error) {
          item.reject(results[index].error);
        } else {
          item.resolve(results[index]);
        }
      });
    } catch (error) {
      batch.forEach((item) => item.reject(error));
    }
  }
}
```

## Integration with AI Models

### Integration with Claude API

```javascript
const Anthropic = require("@anthropic-ai/sdk");
const MCPClient = require("./client");

class AIAssistant {
  constructor() {
    this.anthropic = new Anthropic();
    this.mcpClient = new MCPClient();
    this.tools = [];
  }

  async initialize() {
    // Connect to MCP server
    await this.mcpClient.connect("node", ["./server.js"]);

    // Discover tools
    const capabilities = await this.mcpClient.discoverCapabilities();

    // Convert MCP tools to Claude tool format
    this.tools = Object.values(this.mcpClient.tools).map((tool) => ({
      name: tool.name,
      description: tool.description,
      input_schema: tool.inputSchema,
    }));
  }

  async chat(userMessage) {
    const messages = [{ role: "user", content: userMessage }];

    let response;

    // Agentic loop
    while (true) {
      response = await this.anthropic.messages.create({
        model: "claude-3-5-sonnet-20241022",
        max_tokens: 4096,
        tools: this.tools,
        messages,
      });

      // Check if Claude wants to use tools
      if (response.stop_reason === "tool_use") {
        const toolUses = response.content.filter(
          (block) => block.type === "tool_use",
        );

        // Execute each tool
        const toolResults = [];

        for (const toolUse of toolUses) {
          try {
            const result = await this.mcpClient.callTool(
              toolUse.name,
              toolUse.input,
            );

            toolResults.push({
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: result,
            });
          } catch (error) {
            toolResults.push({
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: `Error: ${error.message}`,
              is_error: true,
            });
          }
        }

        // Add assistant response and tool results to messages
        messages.push({ role: "assistant", content: response.content });
        messages.push({ role: "user", content: toolResults });
      } else {
        // Claude is done
        break;
      }
    }

    // Extract final text response
    const textContent = response.content.find((block) => block.type === "text");

    return textContent?.text || "No response";
  }

  async shutdown() {
    await this.mcpClient.disconnect();
  }
}

// Usage
async function main() {
  const assistant = new AIAssistant();
  await assistant.initialize();

  const response = await assistant.chat(
    "Show me the 5 most active users in the database",
  );

  console.log("Assistant:", response);

  await assistant.shutdown();
}

main().catch(console.error);
```

## Error Handling and Resilience

### Comprehensive Error Handling

```javascript
class RobustClient extends MCPClient {
  async callToolWithFallback(name, args, fallback) {
    try {
      return await this.callTool(name, args);
    } catch (error) {
      console.error(`Tool failed: ${error.message}`);

      // Try fallback if provided
      if (fallback) {
        console.log("Using fallback strategy...");
        return await fallback(name, args);
      }

      throw error;
    }
  }

  async withHealthCheck(fn) {
    try {
      // Quick health check
      await this.client.request(
        {
          method: "tools/list",
          params: {},
        },
        5000,
      );
    } catch (error) {
      console.error("Server appears to be unhealthy");
      throw new Error("Server health check failed");
    }

    return await fn();
  }

  async withCircuitBreaker(name, args, threshold = 5) {
    // Simple circuit breaker pattern
    if (!this.failures) this.failures = {};
    if (!this.failures[name]) this.failures[name] = 0;

    if (this.failures[name] >= threshold) {
      throw new Error(`Circuit breaker open for ${name}`);
    }

    try {
      const result = await this.callTool(name, args);
      this.failures[name] = 0; // Reset on success
      return result;
    } catch (error) {
      this.failures[name]++;
      throw error;
    }
  }
}
```

---

**Next Steps**: Learn about [Real-World Patterns and Best Practices](05-mcp-patterns.md)
