# Building Custom MCP Servers (Python)

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
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install mcp

# Directory structure
my-mcp-server/
├── requirements.txt       # Python dependencies
├── server.py              # Main server file
├── tools/                 # Tool implementations
│   ├── database.py
│   └── api.py
├── resources/             # Resource definitions
│   ├── schemas/
│   └── documentation/
└── utils/
    ├── validators.py
    └── logger.py
```

### Step 2: Create Base Server

```python
# server.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server

class MCPServer:
    def __init__(self):
        self.server = Server("MyMCPServer", "1.0.0")
        self.setup_handlers()

    def setup_handlers(self):
        # Handler for initialize
        @self.server.request_handler("initialize")
        async def handle_initialize(request):
            return {
                "protocolVersion": "2024-11-05",
                "capabilities": {
                    "tools": {},
                    "resources": {},
                    "prompts": {},
                },
                "serverInfo": {
                    "name": "MyMCPServer",
                    "version": "1.0.0",
                },
            }

        # Handler for notifications/initialized
        @self.server.notification_handler("notifications/initialized")
        async def handle_initialized():
            print("[Server] Client initialized")

    async def start(self):
        async with stdio_server(self.server) as streams:
            await streams[0].wait_closed()
            print("[Server] MCP Server started and listening...")

async def main():
    server = MCPServer()
    await server.start()

if __name__ == "__main__":
    asyncio.run(main())
```

### Step 3: Add Tool Handlers

```python
# In setup_handlers() method

@self.server.request_handler("tools/list")
async def handle_tools_list():
    return {
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
                        "params": {
                            "type": "array",
                            "description": "Query parameters"
                        }
                    },
                    "required": ["query"]
                }
            },
            {
                "name": "create_record",
                "description": "Create a new record in the database",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "table": {
                            "type": "string",
                            "description": "Table name"
                        },
                        "data": {
                            "type": "object",
                            "description": "Record data as key-value pairs"
                        }
                    },
                    "required": ["table", "data"]
                }
            }
        ]
    }

@self.server.request_handler("tools/call")
async def handle_tools_call(request):
    name = request.get("params", {}).get("name")
    args = request.get("params", {}).get("arguments", {})

    try:
        if name == "query_database":
            result = await self.execute_query(
                args.get("query"),
                args.get("params")
            )
        elif name == "create_record":
            result = await self.create_record(
                args.get("table"),
                args.get("data")
            )
        else:
            raise ValueError(f"Unknown tool: {name}")

        return {
            "content": [
                {
                    "type": "text",
                    "text": json.dumps(result, indent=2)
                }
            ],
            "isError": False
        }
    except Exception as error:
        return {
            "content": [
                {
                    "type": "text",
                    "text": f"Error: {str(error)}"
                }
            ],
            "isError": True
        }

async def execute_query(self, query, params=None):
    # Implementation for database query
    pass

async def create_record(self, table, data):
    # Implementation for record creation
    pass
```

### Step 4: Add Resource Handlers

```python
@self.server.request_handler("resources/list")
async def handle_resources_list():
    return {
        "resources": [
            {
                "uri": "file:///schemas/database.json",
                "name": "Database Schema",
                "description": "Complete database schema with all tables",
                "mimeType": "application/json"
            },
            {
                "uri": "file:///docs/api.md",
                "name": "API Documentation",
                "description": "API endpoint documentation",
                "mimeType": "text/markdown"
            }
        ]
    }

@self.server.request_handler("resources/read")
async def handle_resources_read(request):
    uri = request.get("params", {}).get("uri")

    try:
        if uri == "file:///schemas/database.json":
            content = await self.get_schema_content()
        elif uri == "file:///docs/api.md":
            content = await self.get_api_documentation()
        else:
            raise ValueError(f"Unknown resource: {uri}")

        return {
            "contents": [
                {
                    "uri": uri,
                    "mimeType": "application/json",
                    "text": content
                }
            ]
        }
    except Exception as error:
        raise ValueError(f"Failed to read resource: {str(error)}")
```

## Defining Tools

### Tool Definition Structure

```python
{
    "name": "tool_name",
    "description": "What the tool does",
    "inputSchema": {
        "type": "object",
        "properties": {
            "param1": {
                "type": "string",
                "description": "Description of param1"
            },
            "param2": {
                "type": "integer",
                "description": "Description of param2",
                "default": 10
            }
        },
        "required": ["param1"]
    }
}
```

### Input Schema Examples

#### Simple String Parameter

```python
{
    "name": "send_email",
    "description": "Send an email to a recipient",
    "inputSchema": {
        "type": "object",
        "properties": {
            "to": {
                "type": "string",
                "description": "Recipient email address"
            },
            "subject": {
                "type": "string",
                "description": "Email subject"
            },
            "body": {
                "type": "string",
                "description": "Email body"
            }
        },
        "required": ["to", "subject", "body"]
    }
}
```

#### Complex Nested Object

```python
{
    "name": "create_product",
    "description": "Create a new product with multiple attributes",
    "inputSchema": {
        "type": "object",
        "properties": {
            "name": {
                "type": "string",
                "description": "Product name"
            },
            "pricing": {
                "type": "object",
                "description": "Pricing information",
                "properties": {
                    "basePrice": {
                        "type": "number",
                        "description": "Base price in USD"
                    },
                    "currency": {
                        "type": "string",
                        "enum": ["USD", "EUR", "GBP"],
                        "description": "Currency code"
                    },
                    "discounts": {
                        "type": "array",
                        "description": "Available discounts",
                        "items": {
                            "type": "object",
                            "properties": {
                                "name": {"type": "string"},
                                "percentage": {"type": "number"}
                            }
                        }
                    }
                },
                "required": ["basePrice", "currency"]
            }
        },
        "required": ["name", "pricing"]
    }
}
```

#### Array with Constraints

```python
{
    "name": "batch_process",
    "description": "Process multiple items in batch",
    "inputSchema": {
        "type": "object",
        "properties": {
            "items": {
                "type": "array",
                "description": "Items to process",
                "items": {"type": "string"},
                "minItems": 1,
                "maxItems": 100
            },
            "processingOptions": {
                "type": "object",
                "properties": {
                    "parallel": {
                        "type": "boolean",
                        "description": "Process in parallel",
                        "default": False
                    },
                    "timeout": {
                        "type": "integer",
                        "description": "Timeout in milliseconds",
                        "default": 30000
                    }
                }
            }
        },
        "required": ["items"]
    }
}
```

## Defining Resources

### Resource Definition Structure

```python
{
    "uri": "resource://identifier",
    "name": "Display Name",
    "description": "What this resource is",
    "mimeType": "application/json"
}
```

### Resource Implementation Examples

#### Static Resource

```python
async def get_schema_resource(self):
    schema = {
        "tables": [
            {
                "name": "users",
                "columns": [
                    {"name": "id", "type": "integer", "pk": True},
                    {"name": "email", "type": "string", "unique": True},
                    {"name": "created_at", "type": "timestamp"}
                ]
            }
        ]
    }

    return {
        "uri": "file:///schemas/database.json",
        "mimeType": "application/json",
        "text": json.dumps(schema, indent=2)
    }
```

#### Dynamic Resource

```python
async def get_documentation(self, version):
    # Generate documentation based on version
    docs = await self.generate_api_docs(version)

    return {
        "uri": f"file:///docs/api-v{version}.md",
        "mimeType": "text/markdown",
        "text": docs
    }
```

#### File Resource

```python
async def read_file_resource(self, file_path):
    try:
        with open(file_path, 'r') as f:
            content = f.read()

        return {
            "uri": f"file://{file_path}",
            "mimeType": "text/plain",
            "text": content
        }
    except Exception as error:
        raise ValueError(f"Failed to read file: {str(error)}")
```

## Defining Prompts

### Prompt Definition Structure

```python
{
    "name": "prompt_name",
    "description": "What this prompt does",
    "arguments": [
        {
            "name": "argument_name",
            "description": "Description of the argument",
            "required": True
        }
    ]
}
```

### Prompt Implementation

```python
@self.server.request_handler("prompts/get")
async def handle_prompts_get(request):
    name = request.get("params", {}).get("name")
    args = request.get("params", {}).get("arguments", {})

    if name == "code_review":
        return {
            "messages": [
                {
                    "role": "user",
                    "content": {
                        "type": "text",
                        "text": f"""Please review the following {args.get('language')} code for:
1. Security vulnerabilities
2. Performance issues
3. Code quality
4. Best practices

Focus on severity level: {args.get('severity', 'all')}

Code:
{args.get('code')}"""
                    }
                }
            ]
        }

    raise ValueError(f"Unknown prompt: {name}")
```

## Complete Example

### Full Database Tool Server

```python
# server.py
import asyncio
import json
import sqlite3
from datetime import datetime
from mcp.server import Server
from mcp.server.stdio import stdio_server

class DatabaseMCPServer:
    def __init__(self, db_path="./data.db"):
        self.db_path = db_path
        self.db_connection = None
        self.server = Server("DatabaseMCPServer", "1.0.0")
        self.setup_handlers()

    def setup_handlers(self):
        @self.server.request_handler("initialize")
        async def handle_initialize(request):
            return {
                "protocolVersion": "2024-11-05",
                "capabilities": {
                    "tools": {},
                    "resources": {}
                },
                "serverInfo": {
                    "name": "DatabaseMCPServer",
                    "version": "1.0.0"
                }
            }

        @self.server.request_handler("tools/list")
        async def handle_tools_list():
            return {
                "tools": [
                    {
                        "name": "query",
                        "description": "Execute a SELECT query on the database",
                        "inputSchema": {
                            "type": "object",
                            "properties": {
                                "sql": {
                                    "type": "string",
                                    "description": "SQL SELECT query"
                                }
                            },
                            "required": ["sql"]
                        }
                    },
                    {
                        "name": "insert",
                        "description": "Insert records into the database",
                        "inputSchema": {
                            "type": "object",
                            "properties": {
                                "table": {"type": "string"},
                                "data": {"type": "object"}
                            },
                            "required": ["table", "data"]
                        }
                    },
                    {
                        "name": "schema",
                        "description": "Get table schemas",
                        "inputSchema": {
                            "type": "object",
                            "properties": {
                                "table": {
                                    "type": "string",
                                    "description": "Table name (optional)"
                                }
                            }
                        }
                    }
                ]
            }

        @self.server.request_handler("tools/call")
        async def handle_tools_call(request):
            name = request.get("params", {}).get("name")
            args = request.get("params", {}).get("arguments", {})

            try:
                if name == "query":
                    result = await self.execute_query(args.get("sql"))
                elif name == "insert":
                    result = await self.insert_record(args.get("table"), args.get("data"))
                elif name == "schema":
                    result = await self.get_schema(args.get("table"))
                else:
                    raise ValueError(f"Unknown tool: {name}")

                return {
                    "content": [{
                        "type": "text",
                        "text": json.dumps(result, indent=2)
                    }],
                    "isError": False
                }
            except Exception as error:
                return {
                    "content": [{
                        "type": "text",
                        "text": f"Error: {str(error)}"
                    }],
                    "isError": True
                }

        @self.server.request_handler("resources/list")
        async def handle_resources_list():
            return {
                "resources": [{
                    "uri": "schema://database",
                    "name": "Database Schema",
                    "description": "Complete database schema",
                    "mimeType": "application/json"
                }]
            }

        @self.server.request_handler("resources/read")
        async def handle_resources_read(request):
            uri = request.get("params", {}).get("uri")

            if uri == "schema://database":
                schema = await self.get_schema()
                return {
                    "contents": [{
                        "uri": uri,
                        "mimeType": "application/json",
                        "text": json.dumps(schema, indent=2)
                    }]
                }

            raise ValueError(f"Unknown resource: {uri}")

    async def execute_query(self, sql):
        cursor = self.db_connection.cursor()
        cursor.execute(sql)
        columns = [description[0] for description in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]

    async def insert_record(self, table, data):
        keys = list(data.keys())
        values = list(data.values())
        placeholders = ",".join(["?" for _ in keys])
        sql = f"INSERT INTO {table} ({','.join(keys)}) VALUES ({placeholders})"

        cursor = self.db_connection.cursor()
        cursor.execute(sql, values)
        self.db_connection.commit()

        return {"id": cursor.lastrowid, "changes": cursor.rowcount}

    async def get_schema(self, table=None):
        cursor = self.db_connection.cursor()

        if table:
            cursor.execute(f"PRAGMA table_info({table})")
        else:
            cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")

        return cursor.fetchall()

    async def initialize(self):
        self.db_connection = sqlite3.connect(self.db_path)
        print(f"[Server] Database connected: {self.db_path}")

    async def start(self):
        await self.initialize()
        async with stdio_server(self.server) as streams:
            await streams[0].wait_closed()
            print("[Server] Database MCP Server started")

async def main():
    server = DatabaseMCPServer()
    await server.start()

if __name__ == "__main__":
    asyncio.run(main())
```

### requirements.txt

```
mcp==0.1.0
```

---

**Next Steps**: Learn how to [Build an MCP Client (Python)](04-building-mcp-client-python.md)
