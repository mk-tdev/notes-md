# MCP Protocol Specification

## Table of Contents

1. [Message Format](#message-format)
2. [Message Types](#message-types)
3. [Protocol Flow](#protocol-flow)
4. [Error Handling](#error-handling)
5. [Transport Layer](#transport-layer)

## Message Format

All MCP communication uses **JSON-RPC 2.0** format.

### Basic Structure

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "method_name",
  "params": {
    "param1": "value1",
    "param2": "value2"
  }
}
```

### Fields

- **jsonrpc**: Protocol version (always "2.0")
- **id**: Unique identifier for request (integer or string)
  - Used to correlate responses with requests
  - Omitted for notifications (one-way messages)
- **method**: Name of the method to invoke
- **params**: Parameters as object or array

### Response Format

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "data": "response_data"
  }
}
```

### Error Response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Server error",
    "data": {
      "details": "Additional error information"
    }
  }
}
```

## Message Types

### 1. Initialization

#### Client → Server: `initialize`

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "sampling": {},
      "experimental": {}
    },
    "clientInfo": {
      "name": "MyAIApp",
      "version": "1.0.0"
    }
  }
}
```

#### Server → Client: `initialize` response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {},
      "prompts": {},
      "experimental": {}
    },
    "serverInfo": {
      "name": "MyMCPServer",
      "version": "1.0.0"
    }
  }
}
```

#### Client → Server: `notifications/initialized`

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized",
  "params": {}
}
```

### 2. Tool Discovery and Execution

#### Client → Server: `tools/list`

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

#### Server → Client: `tools/list` response

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "query_database",
        "description": "Execute SQL queries against the database",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {
              "type": "string",
              "description": "SQL query to execute"
            },
            "limit": {
              "type": "integer",
              "description": "Maximum rows to return",
              "default": 10
            }
          },
          "required": ["query"]
        }
      }
    ]
  }
}
```

#### Client → Server: `tools/call`

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": {
      "query": "SELECT * FROM users WHERE status = 'active'",
      "limit": 5
    }
  }
}
```

#### Server → Client: `tools/call` response

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Query Results:\nid | name | email\n1 | John | john@example.com\n2 | Jane | jane@example.com"
      }
    ],
    "isError": false
  }
}
```

### 3. Resource Access

#### Client → Server: `resources/list`

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/list",
  "params": {}
}
```

#### Server → Client: `resources/list` response

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "resources": [
      {
        "uri": "file:///schemas/users.json",
        "name": "Users Schema",
        "description": "Database schema for users table",
        "mimeType": "application/json"
      }
    ]
  }
}
```

#### Client → Server: `resources/read`

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "method": "resources/read",
  "params": {
    "uri": "file:///schemas/users.json"
  }
}
```

#### Server → Client: `resources/read` response

```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "contents": [
      {
        "uri": "file:///schemas/users.json",
        "mimeType": "application/json",
        "text": "{\"type\": \"object\", \"properties\": {...}}"
      }
    ]
  }
}
```

### 4. Prompts

#### Client → Server: `prompts/list`

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "method": "prompts/list",
  "params": {}
}
```

#### Server → Client: `prompts/list` response

```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "result": {
    "prompts": [
      {
        "name": "analyze_code",
        "description": "Analyze code for security issues",
        "arguments": [
          {
            "name": "language",
            "description": "Programming language",
            "required": true
          },
          {
            "name": "severity_level",
            "description": "Minimum severity to report",
            "required": false
          }
        ]
      }
    ]
  }
}
```

#### Client → Server: `prompts/get`

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "prompts/get",
  "params": {
    "name": "analyze_code",
    "arguments": {
      "language": "python",
      "severity_level": "high"
    }
  }
}
```

#### Server → Client: `prompts/get` response

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Analyze the following code for security issues, focusing on high severity items..."
        }
      }
    ]
  }
}
```

### 5. Sampling (Server requesting AI completion)

#### Server → Client: `completion/create` (Request)

```json
{
  "jsonrpc": "2.0",
  "id": 100,
  "method": "completion/create",
  "params": {
    "model": "claude-3-5-sonnet",
    "systemPrompt": "You are a data analyst",
    "messages": [
      {
        "role": "user",
        "content": "Analyze this data..."
      }
    ],
    "maxTokens": 1000
  }
}
```

#### Client → Server: `completion/create` response

```json
{
  "jsonrpc": "2.0",
  "id": 100,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Based on the data analysis..."
      }
    ],
    "model": "claude-3-5-sonnet",
    "stopReason": "end_turn",
    "usage": {
      "inputTokens": 500,
      "outputTokens": 200
    }
  }
}
```

## Protocol Flow

### Typical Session Lifecycle

```
1. INITIALIZATION PHASE
   ├─ Client: initialize
   ├─ Server: initialize (response)
   └─ Client: notifications/initialized

2. DISCOVERY PHASE
   ├─ Client: tools/list
   ├─ Client: resources/list
   ├─ Client: prompts/list
   └─ Server: responds with available capabilities

3. OPERATIONAL PHASE
   ├─ Client: tools/call (multiple times)
   ├─ Client: resources/read (multiple times)
   ├─ Client: prompts/get (multiple times)
   ├─ Server: completion/create (sampling requests)
   └─ ... repeat as needed

4. TERMINATION PHASE
   └─ Client: closes connection
```

### Example Session

```
Client initiates connection to Server
    ↓
Client sends: {method: "initialize", id: 1}
    ↓
Server responds: {id: 1, result: {capabilities: {...}}}
    ↓
Client sends: {method: "notifications/initialized"}
    ↓
Client sends: {method: "tools/list", id: 2}
    ↓
Server responds: {id: 2, result: {tools: [...]}}
    ↓
Client sends: {method: "tools/call", id: 3, params: {name: "...", arguments: {...}}}
    ↓
Server responds: {id: 3, result: {content: [...]}}
    ↓
... more operations ...
```

## Error Handling

### Standard Error Codes

| Code             | Message          | Meaning                   |
| ---------------- | ---------------- | ------------------------- |
| -32700           | Parse error      | Invalid JSON was received |
| -32600           | Invalid Request  | Request is malformed      |
| -32601           | Method not found | Method does not exist     |
| -32602           | Invalid params   | Parameters are invalid    |
| -32603           | Internal error   | Server encountered error  |
| -32000 to -32099 | Server error     | Custom server errors      |

### Error Response Example

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "expected": "object with 'query' property",
      "received": "object missing 'query' property"
    }
  }
}
```

### Tool Execution Error

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Error: Database connection failed - Connection timeout after 30s"
      }
    ],
    "isError": true
  }
}
```

## Transport Layer

### Supported Transports

#### 1. Standard Input/Output (stdio)

- **Use Case**: Local development, same-machine communication
- **Advantages**: Simple, no network overhead
- **Disadvantages**: Process must run locally
- **Port**: N/A (pipes)

#### 2. Server-Sent Events (SSE)

- **Use Case**: Browser-based clients, real-time updates
- **Advantages**: Built on HTTP, works with any web server
- **Disadvantages**: Server push only, client requests via separate channel
- **Port**: Configurable (typically 8080)

#### 3. HTTP

- **Use Case**: RESTful integration, cloud deployments
- **Advantages**: Standard HTTP, easy to deploy
- **Disadvantages**: Stateless, polling required
- **Port**: Configurable (typically 3000-8080)

### Stdio Transport Example

**Starting server**:

```bash
node server.js
```

**Connection in Node.js**:

```javascript
const {
  StdioClientTransport,
} = require("@modelcontextprotocol/sdk/client/stdio");
const { Client } = require("@modelcontextprotocol/sdk/client/index");

const transport = new StdioClientTransport({
  command: "node",
  args: ["server.js"],
});

const client = new Client(
  {
    name: "MyApp",
    version: "1.0.0",
  },
  {
    capabilities: {},
  },
);

await client.connect(transport);
```

### HTTP Transport Example

**Server setup**:

```javascript
const express = require("express");
const app = express();
const { createMCPServer } = require("./mcp-server");

const mcpServer = createMCPServer();

app.post("/mcp", express.json(), (req, res) => {
  mcpServer.handleMessage(req.body, (response) => {
    res.json(response);
  });
});

app.listen(3000);
```

---

**Next Steps**: Learn how to [Build an MCP Server](03-building-mcp-server.md)
