# MCP Quick Reference - Python

## Installation

```bash
pip install mcp anthropic
```

## Creating a Server

### Minimal Server

```python
from mcp.server import Server
import asyncio

server = Server("my-server")

@server.request_handler("tools/list")
async def list_tools():
    return {"tools": [...]}

@server.request_handler("tools/call")
async def call_tool(request):
    return {"content": [{"type": "text", "text": "result"}]}

async def main():
    from mcp.server.stdio import stdio_server
    async with stdio_server(server) as streams:
        await streams.wait_closed()

asyncio.run(main())
```

### Server with Tools

```python
@server.request_handler("tools/list")
async def list_tools():
    return {
        "tools": [
            {
                "name": "add",
                "description": "Add two numbers",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "a": {"type": "number"},
                        "b": {"type": "number"}
                    },
                    "required": ["a", "b"]
                }
            }
        ]
    }

@server.request_handler("tools/call")
async def call_tool(request):
    if request["name"] == "add":
        a = request["arguments"]["a"]
        b = request["arguments"]["b"]
        result = a + b
        return {"content": [{"type": "text", "text": str(result)}]}
```

### Server with Resources

```python
@server.request_handler("resources/list")
async def list_resources():
    return {
        "resources": [
            {
                "uri": "file:///data.json",
                "name": "Data File",
                "description": "JSON data"
            }
        ]
    }

@server.request_handler("resources/read")
async def read_resource(request):
    uri = request["uri"]
    if uri == "file:///data.json":
        return {
            "contents": [{
                "uri": uri,
                "mimeType": "application/json",
                "text": '{"key": "value"}'
            }]
        }
```

## Creating a Client

### Basic Client

```python
import asyncio
import json
import subprocess

class MCPClient:
    def __init__(self):
        self.process = None
        self.request_id = 0

    async def connect(self, command, args=[]):
        self.process = subprocess.Popen(
            [command] + args,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            text=True, bufsize=1
        )
        await self.send_request("initialize", {...})

    async def send_request(self, method, params):
        self.request_id += 1
        request = {
            "jsonrpc": "2.0",
            "id": self.request_id,
            "method": method,
            "params": params
        }
        self.process.stdin.write(json.dumps(request) + "\n")
        self.process.stdin.flush()

        response_line = self.process.stdout.readline()
        return json.loads(response_line).get("result")

    async def call_tool(self, name, args):
        return await self.send_request("tools/call", {
            "name": name,
            "arguments": args
        })
```

### Client Discovery

```python
async def discover_tools(client):
    response = await client.send_request("tools/list", {})
    return response.get("tools", [])

async def discover_resources(client):
    response = await client.send_request("resources/list", {})
    return response.get("resources", [])
```

## Integration with Claude

```python
from anthropic import Anthropic

client = Anthropic()

# Convert MCP tools to Claude format
claude_tools = [
    {
        "name": tool["name"],
        "description": tool["description"],
        "input_schema": tool["inputSchema"]
    }
    for tool in mcp_tools
]

# Get Claude response with tools
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=4096,
    tools=claude_tools,
    messages=[{"role": "user", "content": "What's the sum of 5 and 3?"}]
)

# Process tool calls
if response.stop_reason == "tool_use":
    for block in response.content:
        if block.type == "tool_use":
            result = await mcp_client.call_tool(block.name, block.input)
```

## Common Patterns

### Error Handling

```python
try:
    result = await client.call_tool("my_tool", {"arg": "value"})
except Exception as e:
    print(f"Tool failed: {e}")
    # Implement fallback
```

### Retry Logic

```python
async def call_with_retry(client, name, args, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await client.call_tool(name, args)
        except Exception as e:
            if attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)
            else:
                raise e
```

### Caching

```python
from functools import lru_cache

class CachingClient:
    def __init__(self, client):
        self.client = client
        self.cache = {}

    async def cached_call(self, name, args):
        key = f"{name}:{json.dumps(args)}"
        if key not in self.cache:
            self.cache[key] = await self.client.call_tool(name, args)
        return self.cache[key]
```

### Batch Processing

```python
async def batch_calls(client, calls):
    tasks = [
        client.call_tool(name, args)
        for name, args in calls
    ]
    return await asyncio.gather(*tasks)
```

## Debugging

### Enable Logging

```python
import logging

logging.basicConfig(level=logging.DEBUG)

# See all MCP protocol messages
logger = logging.getLogger("mcp")
logger.setLevel(logging.DEBUG)
```

### Inspect Server

```python
async def inspect_server(client):
    tools = await client.send_request("tools/list", {})
    resources = await client.send_request("resources/list", {})

    print("Tools:", [t["name"] for t in tools["tools"]])
    print("Resources:", [r["uri"] for r in resources["resources"]])
```

### Test Tool Call

```python
async def test_tool(client, name, args):
    try:
        result = await client.call_tool(name, args)
        print(f"✓ {name}: {result}")
    except Exception as e:
        print(f"✗ {name}: {e}")
```

## Protocol Messages

### Initialize

```python
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {"sampling": {}},
        "clientInfo": {"name": "MyApp", "version": "1.0"}
    }
}
```

### List Tools

```python
{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
}
```

### Call Tool

```python
{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
        "name": "my_tool",
        "arguments": {"arg1": "value1"}
    }
}
```

### Tool Response

```python
{
    "jsonrpc": "2.0",
    "id": 3,
    "result": {
        "content": [
            {"type": "text", "text": "Tool output"}
        ]
    }
}
```

## Environment Setup

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install MCP
pip install mcp

# Install dependencies
pip install anthropic python-dotenv

# Set API key
export ANTHROPIC_API_KEY="sk-..."  # Windows: set ANTHROPIC_API_KEY=sk-...
```

## File Layouts

### Server Project

```
my-server/
├── requirements.txt
├── server.py
├── tools/
│   ├── __init__.py
│   ├── database.py
│   └── api.py
└── README.md
```

### Client Project

```
my-client/
├── requirements.txt
├── client.py
├── assistant.py
└── main.py
```

### Full Stack Project

```
my-app/
├── requirements.txt
├── server/
│   └── server.py
├── client/
│   └── client.py
└── database/
    └── init.sql
```

## Tips and Tricks

**1. Use Type Hints**

```python
async def call_tool(self, name: str, args: dict[str, Any]) -> str:
    ...
```

**2. Async/Await Everywhere**

```python
# Good
await client.call_tool(...)

# Bad - won't work
client.call_tool(...)  # Returns coroutine, not result
```

**3. Handle Disconnections**

```python
try:
    await client.connect(...)
finally:
    await client.disconnect()
```

**4. Monitor Agentic Loops**

```python
iteration = 0
while response.stop_reason != "end_turn":
    iteration += 1
    if iteration > 10:
        break  # Prevent infinite loops
```

**5. Log Everything**

```python
print(f"[Tool Call] {name} with {args}")
print(f"[Tool Result] {result}")
print(f"[Error] {error}")
```

## Common Issues

| Issue                      | Solution                             |
| -------------------------- | ------------------------------------ |
| `ModuleNotFoundError: mcp` | `pip install mcp`                    |
| `asyncio` errors           | Use `async def` and `await`          |
| Socket connection failed   | Ensure server is running             |
| Tool not found             | Check tool name spelling             |
| Timeout errors             | Increase timeout value               |
| Memory leaks               | Call `disconnect()` in finally block |

## Further Reading

- [Building MCP Servers](03-building-mcp-server-python.md)
- [Building MCP Clients](04-building-mcp-client-python.md)
- [Patterns & Best Practices](05-mcp-patterns.md)
- [Complete Example](06-mcp-complete-example-python.md)

---

Generated for MCP Protocol 2024-11-05 | Python 3.8+
