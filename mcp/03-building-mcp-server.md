# Building Custom MCP Servers

## Table of Contents

1. [Server Architecture](#server-architecture)
2. [Implementation Steps](#implementation-steps)
3. [Defining Tools](#defining-tools)
4. [Defining Resources](#defining-resources)
5. [Defining Prompts](#defining-prompts)
6. [Complete Example](#complete-example)

## Server Architecture

### What is an MCP Server?

An MCP Server is a **process that listens for MCP protocol messages** and responds with capabilities and results.

### Design Pattern

```
Input Stream (JSON-RPC messages)
    ↓
Message Parser
    ↓
Method Router
    ├─ initialize → Server Info
    ├─ tools/list → Tool Definitions
    ├─ tools/call → Tool Execution
    ├─ resources/list → Resource Definitions
    ├─ resources/read → Resource Content
    ├─ prompts/list → Prompt Definitions
    └─ prompts/get → Prompt Content
    ↓
Output Stream (JSON-RPC responses)
```

### Core Responsibilities

1. **Accept connections** via stdio, HTTP, SSE, etc.
2. **Parse JSON-RPC messages**
3. **Route messages** to appropriate handlers
4. **Execute tools** when requested
5. **Provide resources** when accessed
6. **Return responses** in JSON-RPC format
7. **Handle errors gracefully**

## Implementation Steps

### Step 1: Set Up Project Structure

```bash
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk

# Directory structure
my-mcp-server/
├── package.json
├── server.js          # Main server file
├── tools/             # Tool implementations
│   ├── database.js
│   └── api.js
├── resources/         # Resource definitions
│   ├── schemas/
│   └── documentation/
└── utils/
    ├── validators.js
    └── logger.js
```

### Step 2: Create Base Server

```javascript
// server.js
const { Server } = require("@modelcontextprotocol/sdk/server/index");
const {
  StdioServerTransport,
} = require("@modelcontextprotocol/sdk/server/stdio");

class MCPServer {
  constructor() {
    this.server = new Server({
      name: "MyMCPServer",
      version: "1.0.0",
    });

    this.setupHandlers();
  }

  setupHandlers() {
    // Handler for initialize
    this.server.setRequestHandler("initialize", async (request) => {
      return {
        protocolVersion: "2024-11-05",
        capabilities: {
          tools: {},
          resources: {},
          prompts: {},
        },
        serverInfo: {
          name: "MyMCPServer",
          version: "1.0.0",
        },
      };
    });

    // Handler for notifications/initialized
    this.server.setNotificationHandler(
      "notifications/initialized",
      async () => {
        console.log("Client initialized successfully");
      },
    );
  }

  async start() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
    console.log("MCP Server started and listening...");
  }
}

// Start server
const mcpServer = new MCPServer();
mcpServer.start().catch(console.error);
```

### Step 3: Add Tool Handlers

```javascript
// In setupHandlers() method

// List available tools
this.server.setRequestHandler("tools/list", async () => {
  return {
    tools: [
      {
        name: "query_database",
        description: "Execute SQL queries against the database",
        inputSchema: {
          type: "object",
          properties: {
            query: {
              type: "string",
              description: "SQL query to execute",
            },
            params: {
              type: "array",
              description: "Query parameters for parameterized queries",
              items: { type: "string" },
            },
          },
          required: ["query"],
        },
      },
      {
        name: "create_record",
        description: "Create a new record in the database",
        inputSchema: {
          type: "object",
          properties: {
            table: {
              type: "string",
              description: "Table name",
            },
            data: {
              type: "object",
              description: "Record data as key-value pairs",
            },
          },
          required: ["table", "data"],
        },
      },
    ],
  };
});

// Execute tools
this.server.setRequestHandler("tools/call", async (request) => {
  const { name, arguments: args } = request.params;

  try {
    let result;

    if (name === "query_database") {
      result = await this.executeQuery(args.query, args.params);
    } else if (name === "create_record") {
      result = await this.createRecord(args.table, args.data);
    } else {
      throw new Error(`Unknown tool: ${name}`);
    }

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(result, null, 2),
        },
      ],
      isError: false,
    };
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
});
```

### Step 4: Add Resource Handlers

```javascript
// List available resources
this.server.setRequestHandler("resources/list", async () => {
  return {
    resources: [
      {
        uri: "file:///schemas/database.json",
        name: "Database Schema",
        description: "Complete database schema with all tables and columns",
        mimeType: "application/json",
      },
      {
        uri: "file:///docs/api.md",
        name: "API Documentation",
        description: "API endpoint documentation and examples",
        mimeType: "text/markdown",
      },
    ],
  };
});

// Read resource content
this.server.setRequestHandler("resources/read", async (request) => {
  const { uri } = request.params;

  try {
    let content;

    if (uri === "file:///schemas/database.json") {
      content = await this.getSchemaContent();
    } else if (uri === "file:///docs/api.md") {
      content = await this.getAPIDocumentation();
    } else {
      throw new Error(`Unknown resource: ${uri}`);
    }

    return {
      contents: [
        {
          uri,
          mimeType: "application/json",
          text: content,
        },
      ],
    };
  } catch (error) {
    throw new Error(`Failed to read resource: ${error.message}`);
  }
});
```

## Defining Tools

### Tool Definition Structure

```javascript
{
  name: "tool_name",                    // Unique identifier
  description: "What the tool does",    // User-facing description
  inputSchema: {                        // JSON Schema for inputs
    type: "object",
    properties: {
      param1: {
        type: "string",
        description: "Description of param1"
      },
      param2: {
        type: "integer",
        description: "Description of param2",
        default: 10
      }
    },
    required: ["param1"]
  }
}
```

### Input Schema Examples

#### Simple String Parameter

```javascript
{
  name: "send_email",
  description: "Send an email to a recipient",
  inputSchema: {
    type: "object",
    properties: {
      to: {
        type: "string",
        description: "Recipient email address"
      },
      subject: {
        type: "string",
        description: "Email subject"
      },
      body: {
        type: "string",
        description: "Email body"
      }
    },
    required: ["to", "subject", "body"]
  }
}
```

#### Complex Nested Object

```javascript
{
  name: "create_product",
  description: "Create a new product with multiple attributes",
  inputSchema: {
    type: "object",
    properties: {
      name: {
        type: "string",
        description: "Product name"
      },
      pricing: {
        type: "object",
        description: "Pricing information",
        properties: {
          basePrice: {
            type: "number",
            description: "Base price in USD"
          },
          currency: {
            type: "string",
            enum: ["USD", "EUR", "GBP"],
            description: "Currency code"
          },
          discounts: {
            type: "array",
            description: "Available discounts",
            items: {
              type: "object",
              properties: {
                name: { type: "string" },
                percentage: { type: "number" }
              }
            }
          }
        },
        required: ["basePrice", "currency"]
      }
    },
    required: ["name", "pricing"]
  }
}
```

#### Array with Constraints

```javascript
{
  name: "batch_process",
  description: "Process multiple items in batch",
  inputSchema: {
    type: "object",
    properties: {
      items: {
        type: "array",
        description: "Items to process",
        items: {
          type: "string"
        },
        minItems: 1,
        maxItems: 100
      },
      processingOptions: {
        type: "object",
        properties: {
          parallel: {
            type: "boolean",
            description: "Process in parallel",
            default: false
          },
          timeout: {
            type: "integer",
            description: "Timeout in milliseconds",
            default: 30000
          }
        }
      }
    },
    required: ["items"]
  }
}
```

## Defining Resources

### Resource Definition Structure

```javascript
{
  uri: "resource://identifier",           // Unique resource URI
  name: "Display Name",                   // User-friendly name
  description: "What this resource is",   // Description
  mimeType: "application/json"            // Content type
}
```

### Resource Implementation Examples

#### Static Resource

```javascript
async getSchemaResource() {
  const schema = {
    tables: [
      {
        name: "users",
        columns: [
          { name: "id", type: "integer", pk: true },
          { name: "email", type: "string", unique: true },
          { name: "created_at", type: "timestamp" }
        ]
      }
    ]
  };

  return {
    uri: "file:///schemas/database.json",
    mimeType: "application/json",
    text: JSON.stringify(schema, null, 2)
  };
}
```

#### Dynamic Resource

```javascript
async getDocumentation(version) {
  // Generate documentation based on version
  const docs = await this.generateAPIDocs(version);

  return {
    uri: `file:///docs/api-v${version}.md`,
    mimeType: "text/markdown",
    text: docs
  };
}
```

#### File Resource

```javascript
const fs = require("fs").promises;

async readFileResource(filePath) {
  try {
    const content = await fs.readFile(filePath, "utf-8");

    return {
      uri: `file://${filePath}`,
      mimeType: "text/plain",
      text: content
    };
  } catch (error) {
    throw new Error(`Failed to read file: ${error.message}`);
  }
}
```

## Defining Prompts

### Prompt Definition Structure

```javascript
{
  name: "prompt_name",
  description: "What this prompt does",
  arguments: [
    {
      name: "argument_name",
      description: "Description of the argument",
      required: true
    }
  ]
}
```

### Prompt Implementation

```javascript
this.server.setRequestHandler("prompts/get", async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "code_review") {
    return {
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Please review the following ${args.language} code for:
1. Security vulnerabilities
2. Performance issues
3. Code quality
4. Best practices

Focus on severity level: ${args.severity || "all"}

Code:
${args.code}`,
          },
        },
      ],
    };
  }

  throw new Error(`Unknown prompt: ${name}`);
});
```

## Complete Example

### Full Database Tool Server

```javascript
// server.js
const { Server } = require("@modelcontextprotocol/sdk/server/index");
const {
  StdioServerTransport,
} = require("@modelcontextprotocol/sdk/server/stdio");
const sqlite3 = require("sqlite3").verbose();

class DatabaseMCPServer {
  constructor(dbPath) {
    this.dbPath = dbPath;
    this.db = null;

    this.server = new Server({
      name: "DatabaseMCPServer",
      version: "1.0.0",
    });

    this.setupHandlers();
  }

  setupHandlers() {
    this.server.setRequestHandler("initialize", async () => {
      return {
        protocolVersion: "2024-11-05",
        capabilities: {
          tools: {},
          resources: {},
        },
        serverInfo: {
          name: "DatabaseMCPServer",
          version: "1.0.0",
        },
      };
    });

    this.server.setRequestHandler("tools/list", async () => {
      return {
        tools: [
          {
            name: "query",
            description: "Execute a SELECT query on the database",
            inputSchema: {
              type: "object",
              properties: {
                sql: {
                  type: "string",
                  description: "SQL SELECT query",
                },
              },
              required: ["sql"],
            },
          },
          {
            name: "insert",
            description: "Insert records into the database",
            inputSchema: {
              type: "object",
              properties: {
                table: { type: "string" },
                data: { type: "object" },
              },
              required: ["table", "data"],
            },
          },
          {
            name: "schema",
            description: "Get table schemas",
            inputSchema: {
              type: "object",
              properties: {
                table: {
                  type: "string",
                  description: "Table name (optional, returns all if omitted)",
                },
              },
            },
          },
        ],
      };
    });

    this.server.setRequestHandler("tools/call", async (request) => {
      const { name, arguments: args } = request.params;

      try {
        let result;

        switch (name) {
          case "query":
            result = await this.executeQuery(args.sql);
            break;
          case "insert":
            result = await this.insertRecord(args.table, args.data);
            break;
          case "schema":
            result = await this.getSchema(args.table);
            break;
          default:
            throw new Error(`Unknown tool: ${name}`);
        }

        return {
          content: [
            {
              type: "text",
              text: JSON.stringify(result, null, 2),
            },
          ],
          isError: false,
        };
      } catch (error) {
        return {
          content: [
            {
              type: "text",
              text: `Error: ${error.message}`,
            },
          ],
          isError: true,
        };
      }
    });

    this.server.setRequestHandler("resources/list", async () => {
      return {
        resources: [
          {
            uri: "schema://database",
            name: "Database Schema",
            description: "Complete database schema",
            mimeType: "application/json",
          },
        ],
      };
    });

    this.server.setRequestHandler("resources/read", async (request) => {
      const { uri } = request.params;

      if (uri === "schema://database") {
        const schema = await this.getSchema();
        return {
          contents: [
            {
              uri,
              mimeType: "application/json",
              text: JSON.stringify(schema, null, 2),
            },
          ],
        };
      }

      throw new Error(`Unknown resource: ${uri}`);
    });
  }

  executeQuery(sql) {
    return new Promise((resolve, reject) => {
      this.db.all(sql, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  }

  insertRecord(table, data) {
    return new Promise((resolve, reject) => {
      const keys = Object.keys(data);
      const values = Object.values(data);
      const placeholders = keys.map(() => "?").join(", ");

      const sql = `INSERT INTO ${table} (${keys.join(", ")}) VALUES (${placeholders})`;

      this.db.run(sql, values, function (err) {
        if (err) reject(err);
        else resolve({ id: this.lastID, changes: this.changes });
      });
    });
  }

  getSchema(table) {
    return new Promise((resolve, reject) => {
      const sql = table
        ? `PRAGMA table_info(${table})`
        : `SELECT name FROM sqlite_master WHERE type='table'`;

      this.db.all(sql, (err, rows) => {
        if (err) reject(err);
        else resolve(rows);
      });
    });
  }

  async start() {
    // Open database
    this.db = new sqlite3.Database(this.dbPath);

    // Connect transport
    const transport = new StdioServerTransport();
    await this.server.connect(transport);

    console.error("Database MCP Server started");
  }
}

// Start the server
const server = new DatabaseMCPServer("./data.db");
server.start().catch(console.error);
```

### package.json

```json
{
  "name": "database-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for database access",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.1.0",
    "sqlite3": "^5.1.6"
  }
}
```

---

**Next Steps**: Learn how to [Build an MCP Client](04-building-mcp-client.md)
