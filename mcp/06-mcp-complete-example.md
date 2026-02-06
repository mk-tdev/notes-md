# Complete End-to-End MCP Example: AI Chat with Database Access

## Project Overview

This example builds a complete AI chat application with MCP that:

- Connects Claude AI to a SQLite database
- Allows the AI to query and analyze data
- Demonstrates tool calling and resource access
- Includes error handling and security

## Directory Structure

```
ai-chat-with-mcp/
├── server/
│   ├── package.json
│   ├── server.js
│   ├── tools/
│   │   ├── database.js
│   │   └── analysis.js
│   └── resources/
│       └── schemas.json
├── client/
│   ├── package.json
│   ├── client.js
│   └── assistant.js
├── database/
│   └── init.sql
└── README.md
```

## Database Initialization

### database/init.sql

```sql
CREATE TABLE IF NOT EXISTS users (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  status TEXT DEFAULT 'active'
);

CREATE TABLE IF NOT EXISTS posts (
  id INTEGER PRIMARY KEY,
  user_id INTEGER NOT NULL,
  title TEXT NOT NULL,
  content TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE IF NOT EXISTS analytics (
  id INTEGER PRIMARY KEY,
  user_id INTEGER NOT NULL,
  page_views INTEGER DEFAULT 0,
  unique_sessions INTEGER DEFAULT 0,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Insert sample data
INSERT INTO users (name, email, status) VALUES
  ('Alice Johnson', 'alice@example.com', 'active'),
  ('Bob Smith', 'bob@example.com', 'active'),
  ('Carol White', 'carol@example.com', 'inactive'),
  ('David Brown', 'david@example.com', 'active'),
  ('Eve Davis', 'eve@example.com', 'active');

INSERT INTO posts (user_id, title, content) VALUES
  (1, 'Getting Started with AI', 'AI is transforming...'),
  (1, 'Best Practices in ML', 'When building ML systems...'),
  (2, 'Database Design 101', 'A good schema is essential...'),
  (3, 'Cloud Architecture', 'Building scalable systems...'),
  (4, 'API Design Patterns', 'RESTful APIs are the standard...');

INSERT INTO analytics (user_id, page_views, unique_sessions) VALUES
  (1, 1523, 342),
  (2, 892, 187),
  (3, 234, 45),
  (4, 3421, 654),
  (5, 2134, 412);
```

## MCP Server Implementation

### server/package.json

```json
{
  "name": "database-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for database access and analysis",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.1.0",
    "sqlite3": "^5.1.6",
    "sqlite": "^5.0.1"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

### server/server.js

Complete server implementation that:

- Manages database connections
- Provides multiple tools for querying data
- Exposes database schema as resources
- Handles tool execution with error handling

```javascript
const { Server } = require("@modelcontextprotocol/sdk/server/index");
const {
  StdioServerTransport,
} = require("@modelcontextprotocol/sdk/server/stdio");
const sqlite3 = require("sqlite3").verbose();
const DatabaseTools = require("./tools/database");
const AnalysisTools = require("./tools/analysis");
const fs = require("fs").promises;
const path = require("path");

class DatabaseMCPServer {
  constructor(dbPath = "./data.db") {
    this.dbPath = dbPath;
    this.db = null;
    this.databaseTools = null;
    this.analysisTools = null;

    this.server = new Server({
      name: "DatabaseMCPServer",
      version: "1.0.0",
    });

    this.setupHandlers();
  }

  setupHandlers() {
    this.server.setRequestHandler("initialize", async (request) => {
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

    this.server.setNotificationHandler(
      "notifications/initialized",
      async () => {
        console.error("[Server] Client initialized");
      },
    );

    this.server.setRequestHandler("tools/list", async () => {
      return {
        tools: [
          ...this.databaseTools.getTools(),
          ...this.analysisTools.getTools(),
        ],
      };
    });

    this.server.setRequestHandler("tools/call", async (request) => {
      const { name, arguments: args } = request.params;

      try {
        let result;

        if (this.databaseTools.hasTool(name)) {
          result = await this.databaseTools.execute(name, args, this.db);
        } else if (this.analysisTools.hasTool(name)) {
          result = await this.analysisTools.execute(name, args, this.db);
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

    this.server.setRequestHandler("resources/list", async () => {
      return {
        resources: [
          {
            uri: "schema://database",
            name: "Database Schema",
            description: "Complete database schema with tables and columns",
            mimeType: "application/json",
          },
          {
            uri: "summary://statistics",
            name: "Database Statistics",
            description: "Summary statistics about the database",
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

      if (uri === "summary://statistics") {
        const stats = await this.getStatistics();
        return {
          contents: [
            {
              uri,
              mimeType: "application/json",
              text: JSON.stringify(stats, null, 2),
            },
          ],
        };
      }

      throw new Error(`Unknown resource: ${uri}`);
    });
  }

  async getSchema() {
    return new Promise((resolve, reject) => {
      this.db.all(
        "SELECT name FROM sqlite_master WHERE type='table'",
        async (err, tables) => {
          if (err) {
            reject(err);
            return;
          }

          const schema = {};

          for (const table of tables) {
            await new Promise((res) => {
              this.db.all(
                `PRAGMA table_info(${table.name})`,
                (err, columns) => {
                  schema[table.name] = {
                    columns: columns.map((col) => ({
                      name: col.name,
                      type: col.type,
                      notnull: col.notnull === 1,
                      pk: col.pk === 1,
                    })),
                  };
                  res();
                },
              );
            });
          }

          resolve(schema);
        },
      );
    });
  }

  async getStatistics() {
    return new Promise((resolve) => {
      const stats = {};
      const tables = ["users", "posts", "analytics"];
      let completed = 0;

      tables.forEach((table) => {
        this.db.get(`SELECT COUNT(*) as count FROM ${table}`, (err, result) => {
          stats[table] = err ? 0 : result.count;
          completed++;

          if (completed === tables.length) {
            resolve({
              timestamp: new Date().toISOString(),
              tables: stats,
            });
          }
        });
      });
    });
  }

  async initialize() {
    this.db = new sqlite3.Database(this.dbPath);
    this.databaseTools = new DatabaseTools();
    this.analysisTools = new AnalysisTools();

    const exists = (await fs.stat(this.dbPath).catch(() => null)) !== null;
    if (!exists) {
      const schema = await fs.readFile(
        path.join(__dirname, "../database/init.sql"),
        "utf-8",
      );

      await new Promise((resolve, reject) => {
        this.db.exec(schema, (err) => {
          if (err) reject(err);
          else resolve();
        });
      });

      console.error("[Server] Database initialized");
    }
  }

  async start() {
    await this.initialize();

    const transport = new StdioServerTransport();
    await this.server.connect(transport);

    console.error("[Server] Started and listening");
  }
}

const server = new DatabaseMCPServer();
server.start().catch(console.error);
```

## Tool Implementations

### server/tools/database.js

```javascript
class DatabaseTools {
  getTools() {
    return [
      {
        name: "query_database",
        description: "Execute SELECT queries on the database",
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
        name: "get_users",
        description: "Get list of users with optional filtering",
        inputSchema: {
          type: "object",
          properties: {
            status: {
              type: "string",
              enum: ["active", "inactive"],
              description: "Filter by status",
            },
          },
        },
      },
      {
        name: "get_user_posts",
        description: "Get all posts by a specific user",
        inputSchema: {
          type: "object",
          properties: {
            userId: {
              type: "integer",
              description: "User ID",
            },
          },
          required: ["userId"],
        },
      },
    ];
  }

  hasTool(name) {
    return this.getTools().some((t) => t.name === name);
  }

  async execute(name, args, db) {
    switch (name) {
      case "query_database":
        return this.queryDatabase(args.sql, db);
      case "get_users":
        return this.getUsers(args.status, db);
      case "get_user_posts":
        return this.getUserPosts(args.userId, db);
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  }

  queryDatabase(sql, db) {
    return new Promise((resolve, reject) => {
      db.all(sql, (err, rows) => {
        if (err) reject(err);
        else resolve(rows || []);
      });
    });
  }

  getUsers(status, db) {
    return new Promise((resolve, reject) => {
      let sql = "SELECT * FROM users";
      const params = [];

      if (status) {
        sql += " WHERE status = ?";
        params.push(status);
      }

      db.all(sql, params, (err, rows) => {
        if (err) reject(err);
        else resolve(rows || []);
      });
    });
  }

  getUserPosts(userId, db) {
    return new Promise((resolve, reject) => {
      const sql = `
        SELECT p.*, u.name as author_name
        FROM posts p
        JOIN users u ON p.user_id = u.id
        WHERE p.user_id = ?
      `;

      db.all(sql, [userId], (err, rows) => {
        if (err) reject(err);
        else resolve(rows || []);
      });
    });
  }
}

module.exports = DatabaseTools;
```

### server/tools/analysis.js

```javascript
class AnalysisTools {
  getTools() {
    return [
      {
        name: "analyze_user_activity",
        description: "Analyze user activity and engagement metrics",
        inputSchema: {
          type: "object",
          properties: {
            userId: {
              type: "integer",
              description: "User ID to analyze",
            },
          },
          required: ["userId"],
        },
      },
      {
        name: "get_top_users",
        description: "Get top users by engagement",
        inputSchema: {
          type: "object",
          properties: {
            limit: {
              type: "integer",
              description: "Number of users to return",
              default: 5,
            },
          },
        },
      },
    ];
  }

  hasTool(name) {
    return this.getTools().some((t) => t.name === name);
  }

  async execute(name, args, db) {
    switch (name) {
      case "analyze_user_activity":
        return this.analyzeUserActivity(args.userId, db);
      case "get_top_users":
        return this.getTopUsers(args.limit || 5, db);
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  }

  async analyzeUserActivity(userId, db) {
    return new Promise((resolve, reject) => {
      const sql = `
        SELECT 
          u.id,
          u.name,
          u.email,
          COUNT(p.id) as post_count,
          a.page_views,
          a.unique_sessions
        FROM users u
        LEFT JOIN posts p ON u.id = p.user_id
        LEFT JOIN analytics a ON u.id = a.user_id
        WHERE u.id = ?
        GROUP BY u.id
      `;

      db.get(sql, [userId], (err, row) => {
        if (err) {
          reject(err);
        } else if (!row) {
          reject(new Error(`User ${userId} not found`));
        } else {
          resolve({
            userId: row.id,
            name: row.name,
            email: row.email,
            engagement: {
              posts: row.post_count,
              pageViews: row.page_views,
              sessions: row.unique_sessions,
              avgViewsPerSession: row.page_views / (row.unique_sessions || 1),
            },
          });
        }
      });
    });
  }

  getTopUsers(limit, db) {
    return new Promise((resolve, reject) => {
      const sql = `
        SELECT 
          u.id,
          u.name,
          COUNT(p.id) as post_count,
          a.page_views
        FROM users u
        LEFT JOIN posts p ON u.id = p.user_id
        LEFT JOIN analytics a ON u.id = a.user_id
        GROUP BY u.id
        ORDER BY a.page_views DESC, post_count DESC
        LIMIT ?
      `;

      db.all(sql, [limit], (err, rows) => {
        if (err) reject(err);
        else resolve(rows || []);
      });
    });
  }
}

module.exports = AnalysisTools;
```

## MCP Client Implementation

### client/client.js

```javascript
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
  }

  async connect(command, args = []) {
    try {
      this.transport = new StdioClientTransport({
        command,
        args,
      });

      this.client = new Client(
        {
          name: "AIChat",
          version: "1.0.0",
        },
        {
          capabilities: {
            sampling: {},
          },
        },
      );

      await this.client.connect(this.transport);

      await this.client.notification({
        method: "notifications/initialized",
        params: {},
      });

      this.connected = true;
      console.log("[Client] Connected to MCP server");
    } catch (error) {
      throw new Error(`Failed to connect: ${error.message}`);
    }
  }

  async discoverCapabilities() {
    if (!this.connected) throw new Error("Not connected");

    try {
      const response = await this.client.request(
        {
          method: "tools/list",
          params: {},
        },
        this.options.timeout,
      );

      this.tools = Object.fromEntries(response.tools.map((t) => [t.name, t]));

      console.log(
        `[Client] Discovered ${Object.keys(this.tools).length} tools`,
      );
      return this.tools;
    } catch (error) {
      throw new Error(`Failed to discover tools: ${error.message}`);
    }
  }

  async callTool(name, args) {
    if (!this.connected) throw new Error("Not connected");
    if (!this.tools[name]) throw new Error(`Tool not found: ${name}`);

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
        throw new Error(response.content[0]?.text || "Tool error");
      }

      return response.content[0]?.text;
    } catch (error) {
      throw new Error(`Tool execution failed: ${error.message}`);
    }
  }

  async readResource(uri) {
    if (!this.connected) throw new Error("Not connected");

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

  async getTools() {
    return Object.values(this.tools).map((t) => ({
      name: t.name,
      description: t.description,
      inputSchema: t.inputSchema,
    }));
  }

  async disconnect() {
    if (this.transport) {
      await this.transport.close();
      this.connected = false;
    }
  }
}

module.exports = MCPClient;
```

### client/assistant.js (AI Integration)

```javascript
const Anthropic = require("@anthropic-ai/sdk");
const MCPClient = require("./client");
const readline = require("readline");

class AIAssistant {
  constructor() {
    this.anthropic = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY,
    });
    this.mcpClient = new MCPClient();
    this.tools = [];
    this.conversationHistory = [];
  }

  async initialize() {
    console.log("[Assistant] Initializing...");

    await this.mcpClient.connect("node", ["./server/server.js"]);

    const tools = await this.mcpClient.discoverCapabilities();

    this.tools = Object.values(tools).map((tool) => ({
      name: tool.name,
      description: tool.description,
      input_schema: tool.inputSchema,
    }));

    console.log("[Assistant] Ready! Type your questions or commands.");
  }

  async chat(userMessage) {
    this.conversationHistory.push({
      role: "user",
      content: userMessage,
    });

    console.log(`\nYou: ${userMessage}\n`);

    let response;
    let iterations = 0;
    const maxIterations = 10;

    while (iterations < maxIterations) {
      iterations++;

      response = await this.anthropic.messages.create({
        model: "claude-3-5-sonnet-20241022",
        max_tokens: 4096,
        system: `You are a helpful database assistant. You have access to tools to:
- Query database tables
- Analyze user activity and engagement
- Retrieve user information

Use the tools to answer questions about the database. Format results clearly.`,
        tools: this.tools,
        messages: this.conversationHistory,
      });

      if (response.stop_reason === "tool_use") {
        const toolUses = response.content.filter(
          (block) => block.type === "tool_use",
        );

        this.conversationHistory.push({
          role: "assistant",
          content: response.content,
        });

        const toolResults = [];

        for (const toolUse of toolUses) {
          console.log(`[Calling tool: ${toolUse.name}]`);

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

            console.log(`[Tool result received]\n`);
          } catch (error) {
            console.error(`[Tool error: ${error.message}]\n`);

            toolResults.push({
              type: "tool_result",
              tool_use_id: toolUse.id,
              content: `Error: ${error.message}`,
              is_error: true,
            });
          }
        }

        this.conversationHistory.push({
          role: "user",
          content: toolResults,
        });
      } else {
        break;
      }
    }

    const textContent = response.content.find((block) => block.type === "text");

    if (textContent) {
      console.log(`Assistant: ${textContent.text}\n`);
      this.conversationHistory.push({
        role: "assistant",
        content: textContent.text,
      });
    }

    return textContent?.text;
  }

  async shutdown() {
    await this.mcpClient.disconnect();
  }
}

async function main() {
  const assistant = new AIAssistant();
  await assistant.initialize();

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const askQuestion = () => {
    rl.question("You: ", async (input) => {
      if (input.toLowerCase() === "exit") {
        console.log("Goodbye!");
        await assistant.shutdown();
        rl.close();
        process.exit(0);
      }

      try {
        await assistant.chat(input);
      } catch (error) {
        console.error(`Error: ${error.message}`);
      }

      askQuestion();
    });
  };

  askQuestion();
}

main().catch(console.error);
```

## Running the Complete Example

### Setup

```bash
# Install dependencies
cd server && npm install
cd ../client && npm install
cd ..

# Initialize database
sqlite3 data.db < database/init.sql

# Set Anthropic API key
export ANTHROPIC_API_KEY="sk-..."
```

### Run

```bash
# Start the AI assistant
node client/assistant.js
```

### Example Interactions

**Q**: "Who are the top users by engagement?"
**A**: Uses `get_top_users` to fetch data, analyzes results

**Q**: "Tell me about Alice's activity"
**A**: Uses `analyze_user_activity` with userId=1

**Q**: "Show me all posts in the database"
**A**: Uses `query_database` to execute SELECT query

## Key Concepts Demonstrated

1. **MCP Server**: Exposes database tools and schema as resources
2. **MCP Client**: Connects to server and manages tools
3. **Tool Discovery**: Dynamically learns available capabilities
4. **Tool Execution**: Calls tools with parameters
5. **Error Handling**: Gracefully handles failures
6. **AI Integration**: Combines Claude with MCP tools for agentic behavior
7. **Resource Access**: Reads database schema as MCP resource

---

**Next Steps**: Review security considerations in [Best Practices Guide](05-mcp-patterns.md#security-best-practices)
