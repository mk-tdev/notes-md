# MCP Complete End-to-End Example (Python)

## Overview

This guide provides a **complete, working MCP application** in Python with:

- ✅ MCP Server with database tools and analysis tools
- ✅ MCP Client with server connection
- ✅ Claude AI integration for intelligent agent behavior
- ✅ Real database operations (SQLite)
- ✅ Complete error handling

## Project Structure

```
mcp-ai-app/
├── requirements.txt
├── database/
│   └── init.sql
├── server/
│   ├── server.py
│   ├── tools/
│   │   ├── database.py
│   │   └── analysis.py
│   └── database.db
├── client/
│   ├── client.py
│   ├── assistant.py
│   └── main.py
└── README.md
```

## Step 1: Setup and Dependencies

### Create requirements.txt

```txt
mcp==0.6.0
anthropic==0.34.0
sqlite3
asyncio
python-dotenv==1.0.0
```

### Install dependencies

```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

## Step 2: Database Setup

### Create database/init.sql

```sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    price REAL NOT NULL,
    category TEXT NOT NULL,
    inventory INTEGER DEFAULT 0
);

CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    product_id INTEGER NOT NULL,
    quantity INTEGER NOT NULL,
    total_price REAL NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

INSERT OR IGNORE INTO users (name, email) VALUES
('Alice Johnson', 'alice@example.com'),
('Bob Smith', 'bob@example.com'),
('Charlie Davis', 'charlie@example.com');

INSERT OR IGNORE INTO products (name, price, category, inventory) VALUES
('Laptop', 999.99, 'Electronics', 5),
('Monitor', 299.99, 'Electronics', 10),
('Keyboard', 79.99, 'Peripherals', 25),
('Mouse', 49.99, 'Peripherals', 50);

INSERT OR IGNORE INTO orders (user_id, product_id, quantity, total_price, status) VALUES
(1, 1, 1, 999.99, 'completed'),
(2, 2, 2, 599.98, 'completed'),
(3, 3, 1, 79.99, 'pending');
```

## Step 3: Create MCP Server

### Create server/tools/database.py

```python
# Database tool implementations
import sqlite3
import json
from typing import List, Dict, Any

class DatabaseTools:
    def __init__(self, db_path: str):
        self.db_path = db_path

    def connect(self):
        return sqlite3.connect(self.db_path)

    def query_database(self, sql: str) -> Dict[str, Any]:
        """Execute a SELECT query"""
        try:
            conn = self.connect()
            cursor = conn.cursor()

            # Prevent dangerous queries
            if not sql.strip().upper().startswith("SELECT"):
                return {
                    "error": "Only SELECT queries are allowed",
                    "success": False
                }

            cursor.execute(sql)
            rows = cursor.fetchall()
            columns = [description[0] for description in cursor.description]

            result = []
            for row in rows:
                result.append(dict(zip(columns, row)))

            conn.close()

            return {
                "success": True,
                "data": result,
                "rowCount": len(result)
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }

    def create_record(self, table: str, data: Dict[str, Any]) -> Dict[str, Any]:
        """Insert a record into a table"""
        try:
            if table not in ["users", "products", "orders"]:
                return {
                    "error": f"Table {table} not allowed",
                    "success": False
                }

            conn = self.connect()
            cursor = conn.cursor()

            columns = ", ".join(data.keys())
            placeholders = ", ".join("?" * len(data))
            values = list(data.values())

            sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"

            cursor.execute(sql, values)
            conn.commit()

            record_id = cursor.lastrowid

            conn.close()

            return {
                "success": True,
                "id": record_id,
                "message": f"Record created in {table}"
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }

    def list_users(self) -> Dict[str, Any]:
        """Get all users"""
        return self.query_database("SELECT id, name, email, created_at FROM users")

    def list_products(self) -> Dict[str, Any]:
        """Get all products"""
        return self.query_database(
            "SELECT id, name, price, category, inventory FROM products"
        )

    def list_orders(self) -> Dict[str, Any]:
        """Get all orders"""
        return self.query_database("""
            SELECT o.id, o.user_id, u.name as user_name, o.product_id,
                   p.name as product_name, o.quantity, o.total_price,
                   o.status, o.created_at
            FROM orders o
            JOIN users u ON o.user_id = u.id
            JOIN products p ON o.product_id = p.id
        """)

    def get_database_schema(self) -> Dict[str, Any]:
        """Get database schema information"""
        try:
            conn = self.connect()
            cursor = conn.cursor()

            cursor.execute("""
                SELECT name, sql FROM sqlite_master
                WHERE type='table'
            """)

            schema = {}
            for table_name, create_sql in cursor.fetchall():
                schema[table_name] = create_sql

            conn.close()

            return {
                "success": True,
                "schema": schema
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }
```

### Create server/tools/analysis.py

```python
# Analysis tool implementations
import sqlite3
from datetime import datetime
from typing import Dict, Any

class AnalysisTools:
    def __init__(self, db_path: str):
        self.db_path = db_path

    def connect(self):
        return sqlite3.connect(self.db_path)

    def analyze_sales(self) -> Dict[str, Any]:
        """Analyze sales data"""
        try:
            conn = self.connect()
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()

            # Total sales
            cursor.execute(
                "SELECT SUM(total_price) as total_sales FROM orders"
            )
            result = cursor.fetchone()
            total_sales = result["total_sales"] or 0

            # Orders by status
            cursor.execute(
                "SELECT status, COUNT(*) as count FROM orders GROUP BY status"
            )
            status_counts = {row["status"]: row["count"] for row in cursor}

            # Top products
            cursor.execute("""
                SELECT p.name, SUM(o.quantity) as quantity_sold
                FROM orders o
                JOIN products p ON o.product_id = p.id
                GROUP BY o.product_id
                ORDER BY quantity_sold DESC
                LIMIT 5
            """)
            top_products = [dict(row) for row in cursor]

            conn.close()

            return {
                "success": True,
                "total_sales": total_sales,
                "order_status": status_counts,
                "top_products": top_products
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }

    def inventory_report(self) -> Dict[str, Any]:
        """Generate inventory report"""
        try:
            conn = self.connect()
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()

            cursor.execute("""
                SELECT name, category, inventory, price
                FROM products
                ORDER BY inventory ASC
            """)

            inventory = [dict(row) for row in cursor]

            # Low stock items (< 10)
            low_stock = [item for item in inventory if item["inventory"] < 10]

            conn.close()

            return {
                "success": True,
                "total_items": len(inventory),
                "low_stock_count": len(low_stock),
                "inventory": inventory,
                "low_stock_alert": low_stock
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }

    def customer_analysis(self) -> Dict[str, Any]:
        """Analyze customer data"""
        try:
            conn = self.connect()
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()

            # Total customers
            cursor.execute("SELECT COUNT(*) as count FROM users")
            total_customers = cursor.fetchone()["count"]

            # Customer spending
            cursor.execute("""
                SELECT u.id, u.name, u.email,
                       COUNT(o.id) as order_count,
                       SUM(o.total_price) as total_spent
                FROM users u
                LEFT JOIN orders o ON u.id = o.user_id
                GROUP BY u.id
                ORDER BY total_spent DESC
            """)

            customer_stats = [dict(row) for row in cursor]

            conn.close()

            return {
                "success": True,
                "total_customers": total_customers,
                "customer_stats": customer_stats
            }
        except Exception as error:
            return {
                "error": str(error),
                "success": False
            }
```

### Create server/server.py

```python
# MCP Server implementation
import sqlite3
import os
from pathlib import Path
from mcp.server import Server
from mcp.types import (
    Tool, TextContent, Resource, Prompt, PromptArgument
)
from server.tools.database import DatabaseTools
from server.tools.analysis import AnalysisTools

class DatabaseMCPServer:
    def __init__(self, db_path: str = "server/database.db"):
        self.db_path = db_path
        self.server = Server("database-mcp-server", "1.0.0")
        self.db_tools = DatabaseTools(db_path)
        self.analysis_tools = AnalysisTools(db_path)
        self.setup_handlers()

    def init_database(self):
        """Initialize database from SQL file"""
        sql_file = Path("database/init.sql")

        if sql_file.exists():
            conn = sqlite3.connect(self.db_path)
            with open(sql_file, 'r') as f:
                conn.executescript(f.read())
            conn.close()
            print(f"[Server] Database initialized: {self.db_path}")

    def setup_handlers(self):
        """Setup MCP request handlers"""

        @self.server.request_handler("initialize")
        async def handle_initialize(request):
            return {
                "protocolVersion": "2024-11-05",
                "capabilities": {
                    "tools": {},
                    "resources": {},
                    "prompts": {}
                },
                "serverInfo": {
                    "name": "database-mcp-server",
                    "version": "1.0.0"
                }
            }

        @self.server.request_handler("tools/list")
        async def handle_tools_list():
            return {
                "tools": [
                    {
                        "name": "query_database",
                        "description": "Execute a SELECT query against the database",
                        "inputSchema": {
                            "type": "object",
                            "properties": {
                                "sql": {
                                    "type": "string",
                                    "description": "SQL SELECT query to execute"
                                }
                            },
                            "required": ["sql"]
                        }
                    },
                    {
                        "name": "create_record",
                        "description": "Insert a record into a table",
                        "inputSchema": {
                            "type": "object",
                            "properties": {
                                "table": {
                                    "type": "string",
                                    "description": "Table name (users, products, or orders)"
                                },
                                "data": {
                                    "type": "object",
                                    "description": "Record data as key-value pairs"
                                }
                            },
                            "required": ["table", "data"]
                        }
                    },
                    {
                        "name": "list_users",
                        "description": "Get all users from the database",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    {
                        "name": "list_products",
                        "description": "Get all products from the database",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    {
                        "name": "list_orders",
                        "description": "Get all orders with user and product details",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    {
                        "name": "analyze_sales",
                        "description": "Get sales analysis and metrics",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    {
                        "name": "inventory_report",
                        "description": "Get inventory status and low stock alerts",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    {
                        "name": "customer_analysis",
                        "description": "Get customer statistics and spending analysis",
                        "inputSchema": {
                            "type": "object",
                            "properties": {}
                        }
                    }
                ]
            }

        @self.server.request_handler("tools/call")
        async def handle_tools_call(request):
            tool_name = request.get("name")
            arguments = request.get("arguments", {})

            try:
                result = None

                if tool_name == "query_database":
                    result = self.db_tools.query_database(arguments.get("sql", ""))
                elif tool_name == "create_record":
                    result = self.db_tools.create_record(
                        arguments.get("table"),
                        arguments.get("data")
                    )
                elif tool_name == "list_users":
                    result = self.db_tools.list_users()
                elif tool_name == "list_products":
                    result = self.db_tools.list_products()
                elif tool_name == "list_orders":
                    result = self.db_tools.list_orders()
                elif tool_name == "analyze_sales":
                    result = self.analysis_tools.analyze_sales()
                elif tool_name == "inventory_report":
                    result = self.analysis_tools.inventory_report()
                elif tool_name == "customer_analysis":
                    result = self.analysis_tools.customer_analysis()
                else:
                    result = {"error": f"Unknown tool: {tool_name}"}

                return {
                    "content": [{"type": "text", "text": str(result)}]
                }
            except Exception as error:
                return {
                    "content": [{"type": "text", "text": f"Error: {str(error)}"}],
                    "isError": True
                }

        @self.server.request_handler("resources/list")
        async def handle_resources_list():
            return {
                "resources": [
                    {
                        "uri": "database:///schema",
                        "name": "Database Schema",
                        "description": "Complete database schema",
                        "mimeType": "application/json"
                    }
                ]
            }

        @self.server.request_handler("resources/read")
        async def handle_resources_read(request):
            uri = request.get("uri")

            if uri == "database:///schema":
                schema = self.db_tools.get_database_schema()
                return {
                    "contents": [{
                        "uri": uri,
                        "mimeType": "application/json",
                        "text": str(schema)
                    }]
                }

            return {"contents": []}

        @self.server.notification_handler("notifications/initialized")
        async def handle_initialized(request):
            print("[Server] Client initialized")

    async def start(self):
        """Start the MCP server"""
        self.init_database()

        # For stdio transport
        import sys
        from mcp.server.stdio import stdio_server

        print("[Server] Starting MCP server...")

        async with stdio_server(self.server) as streams:
            print("[Server] MCP server running. Press Ctrl+C to stop.")
            await streams.wait_closed()

def main():
    import asyncio
    server = DatabaseMCPServer()
    asyncio.run(server.start())

if __name__ == "__main__":
    main()
```

## Step 4: Create MCP Client

### Create client/client.py

```python
# MCP Client implementation
import asyncio
import json
import subprocess
from typing import Dict, Any, Optional

class MCPClient:
    def __init__(self, options: Optional[Dict] = None):
        self.options = {
            "timeout": 30,
            "retries": 3,
            **(options or {})
        }

        self.process = None
        self.connected = False
        self.tools = {}
        self.resources = {}
        self.request_id = 0

    async def connect(self, command: str, args: list = []):
        """Connect to MCP server via stdio"""
        try:
            self.process = subprocess.Popen(
                [command] + args,
                stdin=subprocess.PIPE,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                bufsize=1
            )

            # Send initialize request
            await self.send_request("initialize", {
                "protocolVersion": "2024-11-05",
                "capabilities": {
                    "sampling": {}
                },
                "clientInfo": {
                    "name": "AIAssistant",
                    "version": "1.0.0"
                }
            })

            # Send initialized notification
            await self.send_notification("notifications/initialized", {})

            self.connected = True
            print("[Client] Connected to MCP server")
        except Exception as error:
            raise RuntimeError(f"Failed to connect: {str(error)}")

    async def discover_capabilities(self):
        """Discover available tools and resources"""
        if not self.connected:
            raise RuntimeError("Not connected")

        try:
            # Get tools
            tools_response = await self.send_request("tools/list", {})
            self.tools = {
                tool["name"]: tool
                for tool in tools_response.get("tools", [])
            }

            # Get resources
            resources_response = await self.send_request("resources/list", {})
            self.resources = {
                resource["uri"]: resource
                for resource in resources_response.get("resources", [])
            }

            print(f"[Client] Discovered {len(self.tools)} tools")

            return {
                "tools": self.tools,
                "resources": self.resources
            }
        except Exception as error:
            raise RuntimeError(f"Failed to discover capabilities: {str(error)}")

    async def call_tool(self, name: str, args: Dict[str, Any]):
        """Call a tool on the MCP server"""
        if not self.connected:
            raise RuntimeError("Not connected")

        if name not in self.tools:
            raise ValueError(f"Tool not found: {name}")

        last_error = None

        for attempt in range(self.options["retries"]):
            try:
                response = await self.send_request(
                    "tools/call",
                    {
                        "name": name,
                        "arguments": args
                    }
                )

                if response.get("isError"):
                    error_text = response.get("content", [{}])[0].get("text", "Tool error")
                    raise RuntimeError(error_text)

                return response.get("content", [{}])[0].get("text", "")
            except Exception as error:
                last_error = error

                if attempt < self.options["retries"] - 1:
                    delay = 2 ** attempt
                    print(f"[Client] Retrying {name} after {delay}s...")
                    await asyncio.sleep(delay)

        raise last_error

    async def send_request(self, method: str, params: Dict[str, Any]):
        """Send a JSON-RPC request"""
        self.request_id += 1

        request = {
            "jsonrpc": "2.0",
            "id": self.request_id,
            "method": method,
            "params": params
        }

        # Send to server
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()

        # Wait for response
        response_line = self.process.stdout.readline()
        response = json.loads(response_line)

        if "error" in response:
            raise RuntimeError(f"RPC Error: {response['error']}")

        return response.get("result", {})

    async def send_notification(self, method: str, params: Dict[str, Any]):
        """Send a JSON-RPC notification (no response expected)"""
        notification = {
            "jsonrpc": "2.0",
            "method": method,
            "params": params
        }

        self.process.stdin.write(json.dumps(notification) + "\n")
        self.process.stdin.flush()

    async def disconnect(self):
        """Disconnect from the MCP server"""
        if self.process:
            self.process.stdin.close()
            self.process.wait()
            self.connected = False
            print("[Client] Disconnected from MCP server")
```

### Create client/assistant.py

```python
# AI Assistant with Claude integration
import asyncio
import os
from anthropic import Anthropic
from client.client import MCPClient

class AIAssistant:
    def __init__(self):
        self.anthropic_client = Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.mcp_client = MCPClient()
        self.tools = []
        self.conversation_history = []

    async def initialize(self):
        """Initialize the assistant"""
        # Connect to MCP server
        await self.mcp_client.connect("python", ["server/server.py"])

        # Discover tools
        capabilities = await self.mcp_client.discover_capabilities()

        # Convert MCP tools to Claude tool format
        self.tools = [
            {
                "name": tool["name"],
                "description": tool["description"],
                "input_schema": tool["inputSchema"]
            }
            for tool in self.mcp_client.tools.values()
        ]

        print(f"[Assistant] Ready with {len(self.tools)} tools")

    async def chat(self, user_message: str):
        """Send a message and get a response"""
        print(f"\n👤 User: {user_message}\n")

        # Add to conversation
        self.conversation_history.append({
            "role": "user",
            "content": user_message
        })

        # Agentic loop
        max_iterations = 10
        iteration = 0

        while iteration < max_iterations:
            iteration += 1

            # Get Claude's response
            response = self.anthropic_client.messages.create(
                model="claude-3-5-sonnet-20241022",
                max_tokens=4096,
                tools=self.tools,
                messages=self.conversation_history
            )

            print(f"[Iteration {iteration}] Stop reason: {response.stop_reason}")

            # Check if Claude wants to use tools
            if response.stop_reason == "tool_use":
                tool_uses = [
                    block for block in response.content
                    if block.type == "tool_use"
                ]

                # Add assistant response to history
                self.conversation_history.append({
                    "role": "assistant",
                    "content": response.content
                })

                # Execute tools
                tool_results = []

                for tool_use in tool_uses:
                    print(f"🔧 Calling tool: {tool_use.name}")

                    try:
                        result = await self.mcp_client.call_tool(
                            tool_use.name,
                            tool_use.input
                        )

                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": tool_use.id,
                            "content": result
                        })

                        print(f"✓ Tool result received\n")
                    except Exception as error:
                        print(f"✗ Tool error: {str(error)}\n")

                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": tool_use.id,
                            "content": f"Error: {str(error)}",
                            "is_error": True
                        })

                # Add tool results to history
                self.conversation_history.append({
                    "role": "user",
                    "content": tool_results
                })
            else:
                # Claude is done - extract final response
                break

        # Extract final text response
        text_content = next(
            (block for block in response.content if block.type == "text"),
            None
        )

        if text_content:
            print(f"🤖 Assistant: {text_content.text}\n")
            self.conversation_history.append({
                "role": "assistant",
                "content": text_content.text
            })
            return text_content.text
        else:
            return "No response"

    async def shutdown(self):
        """Shutdown the assistant"""
        await self.mcp_client.disconnect()
```

### Create client/main.py

```python
# Main entry point
import asyncio
import os
from dotenv import load_dotenv
from client.assistant import AIAssistant

async def main():
    # Load environment variables
    load_dotenv()

    # Create assistant
    assistant = AIAssistant()

    try:
        # Initialize
        await assistant.initialize()

        # Example conversation
        await assistant.chat("What products do we have in stock?")

        await assistant.chat("Analyze our sales data and tell me what's our best selling product.")

        await assistant.chat("Create a new user record. Use the name 'David Wilson' and email 'david@example.com'.")

        await assistant.chat("Show me customer statistics and who spent the most.")

    except KeyboardInterrupt:
        print("\n[Main] Shutting down...")
    except Exception as error:
        print(f"[Main] Error: {str(error)}")
    finally:
        await assistant.shutdown()

if __name__ == "__main__":
    asyncio.run(main())
```

## Step 5: Run the Application

### 1. Setup environment

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set API key
export ANTHROPIC_API_KEY="your-api-key-here"
```

### 2. Initialize database

```bash
python -c "from server.server import DatabaseMCPServer; \
           import asyncio; \
           server = DatabaseMCPServer(); \
           server.init_database(); \
           print('Database initialized')"
```

### 3. Run the client

```bash
python client/main.py
```

### Expected Output

```
[Client] Connected to MCP server
[Client] Discovered 8 tools
[Assistant] Ready with 8 tools

👤 User: What products do we have in stock?

🔧 Calling tool: list_products
✓ Tool result received

🤖 Assistant: We have 4 products in stock:
1. Laptop - $999.99 (Electronics) - 5 units available
2. Monitor - $299.99 (Electronics) - 10 units available
3. Keyboard - $79.99 (Peripherals) - 25 units available
4. Mouse - $49.99 (Peripherals) - 50 units available

...
```

## Key Concepts Demonstrated

### 1. Tool Execution

- Client → Server tool calls via MCP protocol
- Argument validation
- Error handling and recovery

### 2. State Management

- Conversation history with Claude
- Tool state in client
- Database state in server

### 3. AI Agent Loop

- Tool discovery and planning
- Tool execution with results
- Iterative reasoning
- Final response generation

### 4. Database Integration

- Multi-table relationships
- Query execution
- Data insertion
- Analysis operations

## Next Steps

- Add persistence layer for conversation history
- Implement caching for frequently used queries
- Add authentication and authorization
- Deploy to production environment
- Scale with connection pooling
- Monitor tool execution metrics

---

**Review**: [MCP Patterns and Best Practices](05-mcp-patterns.md)
