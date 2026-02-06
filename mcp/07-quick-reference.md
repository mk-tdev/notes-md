# MCP Quick Reference & Summary

## Overview

Model Context Protocol (MCP) is a **standardized protocol for connecting AI models with external tools, databases, and services**. It provides a clean separation between your AI application and the capabilities it needs to access.

## Core Concepts Summary

### Three Roles

| Role       | Description         | Example                            |
| ---------- | ------------------- | ---------------------------------- |
| **Host**   | The AI application  | Claude Desktop, Custom Chat App    |
| **Client** | Protocol handler    | `@modelcontextprotocol/sdk` Client |
| **Server** | Capability provider | Database tool server, API server   |

### Main Capabilities

```
┌─────────────────────────────────────────────────┐
│                 MCP Server                       │
├─────────────────────────────────────────────────┤
│ Tools        → Functions AI can call             │
│ Resources    → Data AI can read                  │
│ Prompts      → Pre-built prompt templates        │
│ Sampling     → Server requests AI completions   │
└─────────────────────────────────────────────────┘
```

## Quick Start

### 1. Create an MCP Server (Node.js)

```javascript
const { Server } = require("@modelcontextprotocol/sdk/server/index");
const {
  StdioServerTransport,
} = require("@modelcontextprotocol/sdk/server/stdio");

const server = new Server({
  name: "MyServer",
  version: "1.0.0",
});

// Handle tool list requests
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "my_tool",
      description: "Does something useful",
      inputSchema: {
        type: "object",
        properties: {
          input: { type: "string" },
        },
        required: ["input"],
      },
    },
  ],
}));

// Handle tool execution
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "my_tool") {
    return {
      content: [
        {
          type: "text",
          text: "Result: " + request.params.arguments.input,
        },
      ],
      isError: false,
    };
  }
});

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 2. Create an MCP Client (Node.js)

```javascript
const { Client } = require("@modelcontextprotocol/sdk/client/index");
const {
  StdioClientTransport,
} = require("@modelcontextprotocol/sdk/client/stdio");

const client = new Client(
  {
    name: "MyApp",
    version: "1.0.0",
  },
  {
    capabilities: {},
  },
);

const transport = new StdioClientTransport({
  command: "node",
  args: ["server.js"],
});

await client.connect(transport);

// Call a tool
const response = await client.request({
  method: "tools/call",
  params: {
    name: "my_tool",
    arguments: { input: "hello" },
  },
});

console.log(response.content[0].text);
```

### 3. Connect to Claude AI

```javascript
const Anthropic = require("@anthropic-ai/sdk");
const mcpClient = ...; // Your MCP client

const anthropic = new Anthropic();

// Convert MCP tools to Claude format
const tools = (await mcpClient.request({
  method: "tools/list",
  params: {}
})).tools.map(t => ({
  name: t.name,
  description: t.description,
  input_schema: t.inputSchema
}));

// Chat with tool use
const response = await anthropic.messages.create({
  model: "claude-3-5-sonnet-20241022",
  max_tokens: 4096,
  tools,
  messages: [{
    role: "user",
    content: "Use a tool to help me"
  }]
});

// Handle tool calls from Claude
if (response.stop_reason === "tool_use") {
  for (const block of response.content) {
    if (block.type === "tool_use") {
      const result = await mcpClient.request({
        method: "tools/call",
        params: {
          name: block.name,
          arguments: block.input
        }
      });
      // Send result back to Claude
    }
  }
}
```

## Protocol Reference

### JSON-RPC Message Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

### Common Methods

#### Server Discovery

- `initialize` - Initial handshake and capability negotiation
- `notifications/initialized` - Client ready notification

#### Tools

- `tools/list` - Get available tools
- `tools/call` - Execute a tool

#### Resources

- `resources/list` - Get available resources
- `resources/read` - Read resource content

#### Prompts

- `prompts/list` - Get available prompts
- `prompts/get` - Get prompt with arguments

#### Sampling

- `completion/create` - Request AI completion (server → client)

## Common Patterns

### Pattern 1: Database Tool Server

**Use Case**: Expose database queries as tools

```javascript
// Define tool
{
  name: "query_db",
  description: "Execute SQL queries",
  inputSchema: {
    type: "object",
    properties: {
      sql: { type: "string" }
    },
    required: ["sql"]
  }
}

// Execute
const result = await database.query(args.sql);
return {
  content: [{ type: "text", text: JSON.stringify(result) }],
  isError: false
};
```

### Pattern 2: API Integration Server

**Use Case**: Expose REST APIs as tools

```javascript
// Define tool
{
  name: "fetch_user",
  description: "Get user information",
  inputSchema: {
    type: "object",
    properties: {
      userId: { type: "integer" }
    },
    required: ["userId"]
  }
}

// Execute
const user = await fetch(`/api/users/${args.userId}`);
return {
  content: [{ type: "text", text: JSON.stringify(user) }],
  isError: false
};
```

### Pattern 3: Caching Client

**Use Case**: Cache tool results to improve performance

```javascript
class CachingClient extends MCPClient {
  async callTool(name, args) {
    const key = `${name}:${JSON.stringify(args)}`;

    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    const result = await super.callTool(name, args);
    this.cache.set(key, result);

    return result;
  }
}
```

### Pattern 4: Retry Logic

**Use Case**: Handle transient failures

```javascript
async callToolWithRetry(name, args, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await this.callTool(name, args);
    } catch (error) {
      if (i < maxRetries - 1) {
        const delay = Math.pow(2, i) * 1000;
        await new Promise(r => setTimeout(r, delay));
      } else {
        throw error;
      }
    }
  }
}
```

## Best Practices Checklist

### Server Development

- ✅ **Validate inputs** - Always validate tool parameters
- ✅ **Clear descriptions** - Write descriptive tool descriptions
- ✅ **Error handling** - Return meaningful error messages
- ✅ **Timeouts** - Set reasonable execution timeouts
- ✅ **Logging** - Log important events for debugging
- ✅ **Security** - Validate tokens, implement rate limiting
- ✅ **Testing** - Test tools independently before deployment

### Client Development

- ✅ **Connection pooling** - Reuse connections efficiently
- ✅ **Caching** - Cache tool definitions and resources
- ✅ **Retry logic** - Handle transient failures gracefully
- ✅ **Timeouts** - Set reasonable request timeouts
- ✅ **Monitoring** - Track tool usage and errors
- ✅ **Error handling** - Provide helpful error messages to users

### Integration with AI

- ✅ **Tool descriptions** - Write descriptions for AI understanding
- ✅ **Input validation** - Let AI know constraints upfront
- ✅ **Error feedback** - Return errors so AI can retry differently
- ✅ **Resource context** - Use resources to provide background info
- ✅ **Sampling** - Use server sampling for complex decision-making

## Common Mistakes to Avoid

### ❌ Missing Input Validation

```javascript
// Bad
const result = await db.query(args.sql);

// Good
if (!args.sql || typeof args.sql !== "string") {
  throw new Error("sql must be a non-empty string");
}
const result = await db.query(args.sql);
```

### ❌ Overly Complex Schemas

```javascript
// Bad - AI struggles with understanding
{
  name: "complex_tool",
  inputSchema: {
    type: "object",
    properties: {
      data: {
        type: "array",
        items: {
          type: "object",
          properties: { /* 20 nested properties */ }
        }
      }
    }
  }
}

// Good - Simple, focused
{
  name: "process_record",
  inputSchema: {
    type: "object",
    properties: {
      recordId: { type: "integer" },
      operation: { type: "string", enum: ["create", "update", "delete"] }
    },
    required: ["recordId", "operation"]
  }
}
```

### ❌ Not Handling Tool Errors

```javascript
// Bad
return {
  content: [{ type: "text", text: JSON.stringify(result) }],
};

// Good
return {
  content: [
    {
      type: "text",
      text: error ? `Error: ${error.message}` : JSON.stringify(result),
    },
  ],
  isError: !!error,
};
```

### ❌ Missing Timeouts

```javascript
// Bad
const result = await someAsyncOperation();

// Good
const result = await Promise.race([
  someAsyncOperation(),
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error("Timeout")), 5000),
  ),
]);
```

## Security Considerations

### 1. Input Validation

```javascript
function validateQuery(sql) {
  const dangerous = [/DROP/i, /DELETE/i, /TRUNCATE/i];
  if (dangerous.some((p) => p.test(sql))) {
    throw new Error("Disallowed SQL operation");
  }
  if (sql.length > 10000) {
    throw new Error("Query too long");
  }
  return true;
}
```

### 2. Authentication

```javascript
function validateToken(token, authManager) {
  if (!authManager.isValid(token)) {
    throw new Error("Unauthorized");
  }
  return authManager.getPermissions(token);
}
```

### 3. Rate Limiting

```javascript
class RateLimiter {
  check(clientId) {
    const count = this.getRecentCount(clientId);
    if (count > this.maxRequests) {
      throw new Error("Rate limit exceeded");
    }
  }
}
```

### 4. Output Sanitization

```javascript
function sanitizeOutput(data) {
  // Remove sensitive fields
  if (data.password) delete data.password;
  if (data.apiKey) delete data.apiKey;
  return data;
}
```

## Debugging Tips

### Enable Verbose Logging

```javascript
const {
  StdioServerTransport,
} = require("@modelcontextprotocol/sdk/server/stdio");

// Log all messages
server.setRequestHandler("tools/call", async (request) => {
  console.error("[DEBUG] Tool call:", JSON.stringify(request, null, 2));
  // ... handle request
});
```

### Use Mock Servers for Testing

```javascript
class MockServer {
  async handleRequest(method, params) {
    const responses = {
      "tools/list": { tools: [...] },
      "tools/call": { content: [...], isError: false }
    };
    return responses[method];
  }
}
```

### Inspect Messages in Flight

```javascript
class DebuggingTransport {
  async send(message) {
    console.error("[SEND]", JSON.stringify(message, null, 2));
    return this.transport.send(message);
  }

  async receive() {
    const message = await this.transport.receive();
    console.error("[RECV]", JSON.stringify(message, null, 2));
    return message;
  }
}
```

## Resources

### Official Documentation

- [Anthropic MCP Documentation](https://modelcontextprotocol.io)
- [MCP SDK Reference](https://github.com/anthropics/model-context-protocol)
- [Claude API Documentation](https://docs.anthropic.com)

### Learn More

- Review [Complete Example Project](06-mcp-complete-example.md)
- Study [Design Patterns](05-mcp-patterns.md)
- Read [Protocol Specification](02-mcp-protocol.md)

### Community Resources

- MCP GitHub Discussions
- Anthropic Claude Community Forum
- MCP Server Examples Repository

## Key Takeaways

1. **MCP enables loose coupling** between AI and external systems
2. **Tools are functions** that AI can decide to call
3. **Resources provide context** for AI reasoning
4. **Prompts encapsulate expertise** and best practices
5. **Sampling allows servers** to request AI completions
6. **JSON-RPC provides** a simple, standard protocol
7. **Error handling** makes systems robust
8. **Security** protects your systems from misuse

---

**Start building with MCP today!** Choose an area from the table of contents to dive deeper.
