# Building MCP Clients (Python)

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
├─ discover_tools()
├─ call_tool()
├─ read_resource()
├─ get_prompt()
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
- **Type Safety**: Optional type hints for better IDE support

## Client Types

### 1. Minimal Client

**Use Case**: Simple scripts or debugging

```python
import subprocess
import json

def create_minimal_client():
    def query(method, params):
        # Direct connection - for testing only
        proc = subprocess.Popen(
            ["python", "server.py"],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE
        )
        # Send message, get response
        return proc

    return {"query": query}
```

### 2. Production Client

**Use Case**: Enterprise applications, cloud deployments

- Full error handling
- Retry logic and backoff
- Connection pooling
- Comprehensive logging
- Health checks

### 3. Web Client

**Use Case**: Web-based AI assistants

- HTTP/SSE transport
- Request batching
- UI feedback and progress
- Offline support with local caching

## Basic Client Implementation

### Step 1: Create Client Class

```python
# client.py
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
                    "name": "MyAIApp",
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
            print(f"[Client] Discovered {len(self.resources)} resources")

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

                return response.get("content", [{}])[0].get("text")
            except Exception as error:
                last_error = error

                if attempt < self.options["retries"] - 1:
                    delay = 2 ** attempt
                    print(f"[Client] Retrying {name} after {delay}s...")
                    await asyncio.sleep(delay)

        raise last_error

    async def read_resource(self, uri: str):
        """Read a resource from the MCP server"""
        if not self.connected:
            raise RuntimeError("Not connected")

        if uri not in self.resources:
            raise ValueError(f"Resource not found: {uri}")

        try:
            response = await self.send_request(
                "resources/read",
                {"uri": uri}
            )

            return response.get("contents", [{}])[0].get("text")
        except Exception as error:
            raise RuntimeError(f"Failed to read resource: {str(error)}")

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

    async def list_tools(self):
        """Get list of available tools"""
        return [
            {
                "name": tool["name"],
                "description": tool.get("description", "")
            }
            for tool in self.tools.values()
        ]

    async def list_resources(self):
        """Get list of available resources"""
        return [
            {
                "uri": resource["uri"],
                "name": resource.get("name", ""),
                "description": resource.get("description", "")
            }
            for resource in self.resources.values()
        ]

    async def disconnect(self):
        """Disconnect from the MCP server"""
        if self.process:
            self.process.stdin.close()
            self.process.wait()
            self.connected = False
            print("[Client] Disconnected from MCP server")
```

### Step 2: Use the Client

```python
# app.py
import asyncio
from client import MCPClient

async def main():
    client = MCPClient({
        "timeout": 30,
        "retries": 3
    })

    try:
        # Connect to server
        await client.connect("python", ["./server.py"])

        # Discover capabilities
        capabilities = await client.discover_capabilities()

        print("Available tools:")
        tools = await client.list_tools()
        for tool in tools:
            print(f"  - {tool['name']}: {tool['description']}")

        # Call a tool
        result = await client.call_tool("query", {
            "sql": "SELECT * FROM users LIMIT 5"
        })

        print(f"Query result: {result}")

        # Read a resource
        schema = await client.read_resource("file:///schemas/database.json")
        print(f"Schema: {schema}")

    except Exception as error:
        print(f"Error: {str(error)}")
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(main())
```

## Advanced Client Features

### 1. Tool Result Caching

```python
from datetime import datetime, timedelta

class CachingClient(MCPClient):
    def __init__(self, options=None):
        super().__init__(options)
        self.cache = {}
        self.cache_expiry = (options or {}).get("cache_expiry", 300)  # 5 minutes

    async def call_tool(self, name, args):
        cache_key = f"{name}:{json.dumps(args)}"

        # Check cache
        if cache_key in self.cache:
            result, timestamp = self.cache[cache_key]
            if (datetime.now() - timestamp).total_seconds() < self.cache_expiry:
                print(f"[Client] Cache hit for {cache_key}")
                return result

        # Call tool
        result = await super().call_tool(name, args)

        # Store in cache
        self.cache[cache_key] = (result, datetime.now())

        return result

    def clear_cache(self):
        self.cache.clear()
```

### 2. Connection Pool

```python
import asyncio
from typing import Callable

class ClientPool:
    def __init__(self, size: int = 5):
        self.size = size
        self.clients = []
        self.available = asyncio.Queue()
        self.waiting = []

    async def initialize(self, command: str, args: list):
        for i in range(self.size):
            client = MCPClient()
            await client.connect(command, args)
            await client.discover_capabilities()

            self.clients.append(client)
            await self.available.put(client)

    async def acquire(self):
        return await self.available.get()

    async def release(self, client):
        await self.available.put(client)

    async def execute_with_client(self, fn: Callable):
        client = await self.acquire()
        try:
            return await fn(client)
        finally:
            await self.release(client)

    async def shutdown(self):
        for client in self.clients:
            await client.disconnect()
```

### 3. Request Batching

```python
class BatchingClient(MCPClient):
    def __init__(self, options=None):
        super().__init__(options)
        self.batch = []
        self.batch_size = (options or {}).get("batch_size", 10)
        self.batch_delay = (options or {}).get("batch_delay", 0.1)
        self.batch_timer = None

    async def call_tool_batched(self, name, args):
        future = asyncio.Future()
        self.batch.append((name, args, future))

        if len(self.batch) >= self.batch_size:
            await self.process_batch()
        elif not self.batch_timer:
            self.batch_timer = asyncio.create_task(
                self._batch_timeout()
            )

        return await future

    async def _batch_timeout(self):
        await asyncio.sleep(self.batch_delay)
        if self.batch:
            await self.process_batch()

    async def process_batch(self):
        if self.batch_timer:
            self.batch_timer.cancel()
            self.batch_timer = None

        if not self.batch:
            return

        batch = self.batch[:]
        self.batch = []

        tasks = [
            super().call_tool(name, args)
            for name, args, _ in batch
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)

        for i, (_, _, future) in enumerate(batch):
            if isinstance(results[i], Exception):
                future.set_exception(results[i])
            else:
                future.set_result(results[i])
```

## Integration with AI Models

### Integration with Claude API

```python
import anthropic

class AIAssistant:
    def __init__(self):
        self.anthropic_client = anthropic.Anthropic()
        self.mcp_client = MCPClient()
        self.tools = []

    async def initialize(self):
        # Connect to MCP server
        await self.mcp_client.connect("python", ["./server.py"])

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

    async def chat(self, user_message: str):
        messages = [
            {"role": "user", "content": user_message}
        ]

        print(f"\nYou: {user_message}\n")

        # Agentic loop
        while True:
            response = self.anthropic_client.messages.create(
                model="claude-3-5-sonnet-20241022",
                max_tokens=4096,
                tools=self.tools,
                messages=messages
            )

            # Check if Claude wants to use tools
            if response.stop_reason == "tool_use":
                tool_uses = [
                    block for block in response.content
                    if block.type == "tool_use"
                ]

                # Add assistant response
                messages.append({
                    "role": "assistant",
                    "content": response.content
                })

                # Execute tools
                tool_results = []

                for tool_use in tool_uses:
                    print(f"[Calling tool: {tool_use.name}]")

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

                        print(f"[Tool result received]\n")
                    except Exception as error:
                        print(f"[Tool error: {str(error)}]\n")

                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": tool_use.id,
                            "content": f"Error: {str(error)}",
                            "is_error": True
                        })

                # Add tool results
                messages.append({
                    "role": "user",
                    "content": tool_results
                })
            else:
                # Claude is done
                break

        # Extract final response
        text_content = next(
            (block for block in response.content if block.type == "text"),
            None
        )

        if text_content:
            print(f"Assistant: {text_content.text}\n")
            messages.append({
                "role": "assistant",
                "content": text_content.text
            })

        return text_content.text if text_content else "No response"

    async def shutdown(self):
        await self.mcp_client.disconnect()
```

## Error Handling and Resilience

### Comprehensive Error Handling

```python
class RobustClient(MCPClient):
    async def call_tool_with_fallback(self, name, args, fallback=None):
        try:
            return await self.call_tool(name, args)
        except Exception as error:
            print(f"[Client] Tool failed: {str(error)}")

            if fallback:
                print(f"[Client] Using fallback strategy...")
                return await fallback(name, args)

            raise error

    async def with_health_check(self, fn):
        try:
            # Quick health check
            await asyncio.wait_for(
                self.send_request("tools/list", {}),
                timeout=5
            )
        except asyncio.TimeoutError:
            print("[Client] Server appears to be unhealthy")
            raise RuntimeError("Server health check failed")

        return await fn()

    async def with_circuit_breaker(self, name, args, threshold=5):
        if not hasattr(self, '_failures'):
            self._failures = {}

        if name not in self._failures:
            self._failures[name] = 0

        if self._failures[name] >= threshold:
            raise RuntimeError(f"Circuit breaker open for {name}")

        try:
            result = await self.call_tool(name, args)
            self._failures[name] = 0  # Reset on success
            return result
        except Exception as error:
            self._failures[name] += 1
            raise error
```

---

**Next Steps**: Learn about [Real-World Patterns and Best Practices](05-mcp-patterns.md)
