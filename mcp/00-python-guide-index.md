# MCP Learning Guide - Python Edition

## Welcome! 👋

This is a **comprehensive guide to Model Context Protocol (MCP)** with **Python examples**. Whether you're building AI applications, integrating tools with Claude, or exploring the MCP ecosystem, this guide has you covered.

## Quick Start (5 minutes)

**New to MCP?** Start here:

1. Read [MCP Overview](01-mcp-overview.md) - 10 min conceptual foundation
2. Look at [Protocol Basics](02-mcp-protocol.md) - 15 min spec understanding
3. Jump to [Building a Server](03-building-mcp-server-python.md) - 30 min hands-on

**Want a complete working app?** Go straight to:

- [Complete End-to-End Example](06-mcp-complete-example-python.md) - Full working Python app with Claude

## Learning Paths

### 🚀 Path 1: Building AI Agents (Most Popular)

Learn to build AI assistants that can use tools and access data:

1. **Foundations** (30 min)
   - [MCP Overview](01-mcp-overview.md) - Understand what MCP is
   - [Protocol Basics](02-mcp-protocol.md) - How it works under the hood

2. **Building Tools** (45 min)
   - [Building MCP Servers (Python)](03-building-mcp-server-python.md) - Create tool providers
   - [Tools Deep Dive](03-building-mcp-server-python.md#tools) - Tool definitions and handlers

3. **Using Tools with Claude** (60 min)
   - [Building MCP Clients (Python)](04-building-mcp-client-python.md) - Connect to tools
   - [AI Integration Examples](06-mcp-complete-example-python.md#step-4-create-mcp-client) - Claude integration

4. **Production Ready** (90 min)
   - [Real-World Patterns](05-mcp-patterns.md) - Design patterns and best practices
   - [Complete Example](06-mcp-complete-example-python.md) - Full working application

---

### 📚 Path 2: Building Tool Providers

Learn to expose your systems through MCP:

1. **Understand MCP** (20 min)
   - [MCP Overview](01-mcp-overview.md)
   - [Core Concepts](01-mcp-overview.md#core-concepts)

2. **Build Your Server** (60 min)
   - [Server Architecture](03-building-mcp-server-python.md#server-architecture)
   - [Tool Implementation](03-building-mcp-server-python.md#tools)
   - [Resource Serving](03-building-mcp-server-python.md#resources)

3. **Advanced Features** (45 min)
   - [Prompts and Templating](03-building-mcp-server-python.md#prompts)
   - [Error Handling](03-building-mcp-server-python.md#error-handling)
   - [Security Patterns](05-mcp-patterns.md#security)

4. **Deploy & Scale** (60 min)
   - [Testing Strategies](05-mcp-patterns.md#testing)
   - [Performance Optimization](05-mcp-patterns.md#performance)
   - [Production Deployment](06-mcp-complete-example-python.md#production-considerations)

---

### 🔌 Path 3: Integrating MCP into Existing Apps

Learn to add MCP clients to your applications:

1. **Learn the Protocol** (20 min)
   - [MCP Overview](01-mcp-overview.md#client-server-communication)
   - [Protocol Reference](02-mcp-protocol.md)

2. **Create a Client** (45 min)
   - [Client Architecture](04-building-mcp-client-python.md#client-architecture)
   - [Connection Management](04-building-mcp-client-python.md#basic-client-implementation)
   - [Tool Discovery](04-building-mcp-client-python.md#basic-client-implementation)

3. **Connect to Claude** (30 min)
   - [Claude Integration](04-building-mcp-client-python.md#integration-with-ai-models)
   - [Agentic Patterns](06-mcp-complete-example-python.md#step-4-create-mcp-client)

4. **Advanced Patterns** (45 min)
   - [Caching and Pooling](04-building-mcp-client-python.md#advanced-client-features)
   - [Error Handling](04-building-mcp-client-python.md#error-handling-and-resilience)
   - [Real-World Examples](05-mcp-patterns.md)

---

## File Guide

| File                                                                   | Type           | Time   | Purpose                                     |
| ---------------------------------------------------------------------- | -------------- | ------ | ------------------------------------------- |
| [01-mcp-overview.md](01-mcp-overview.md)                               | 📖 Concepts    | 10 min | Understanding MCP architecture and concepts |
| [02-mcp-protocol.md](02-mcp-protocol.md)                               | 📋 Reference   | 15 min | Detailed protocol specification             |
| [03-building-mcp-server-python.md](03-building-mcp-server-python.md)   | 💻 Hands-On    | 45 min | Creating MCP servers in Python              |
| [04-building-mcp-client-python.md](04-building-mcp-client-python.md)   | 💻 Hands-On    | 45 min | Creating MCP clients in Python              |
| [05-mcp-patterns.md](05-mcp-patterns.md)                               | 🎯 Patterns    | 60 min | Design patterns and best practices          |
| [06-mcp-complete-example-python.md](06-mcp-complete-example-python.md) | 🚀 Project     | 90 min | Complete working AI application             |
| [07-quick-reference.md](07-quick-reference.md)                         | 📚 Cheat Sheet | 5 min  | Quick lookup reference                      |

## Key Concepts

### What is MCP?

**Model Context Protocol** is a **standard for AI applications to use external tools and data**.

```
Your AI App
    ↓
MCP Client (connects tools)
    ↓
MCP Server (provides tools)
    ↓
Your Database / APIs / Systems
```

### Core Features

- **Tools**: Functions your app can call
- **Resources**: Files and data the app can read
- **Prompts**: Pre-written instructions and templates
- **Sampling**: AI model function calling

### Why Use MCP?

1. **Standardization**: Same protocol for all tools
2. **Flexibility**: Mix and match tools easily
3. **Security**: Controlled access to systems
4. **Scalability**: Connect dozens of tools
5. **Interoperability**: Works with any MCP client

## Example: Your First Tool

```python
from mcp.server import Server

server = Server("my-app")

@server.request_handler("tools/list")
async def list_tools():
    return {
        "tools": [
            {
                "name": "get_weather",
                "description": "Get weather for a city",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "city": {"type": "string"}
                    }
                }
            }
        ]
    }

@server.request_handler("tools/call")
async def call_tool(request):
    if request["name"] == "get_weather":
        city = request["arguments"]["city"]
        return {
            "content": [{"type": "text", "text": f"Weather in {city}..."}]
        }
```

## Prerequisites

- **Python 3.8+**
- **Basic asyncio knowledge** (helpful but not required)
- **Anthropic API key** (for Claude integration examples)

### Installation

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install MCP
pip install mcp

# Install Anthropic SDK (for AI examples)
pip install anthropic
```

## Common Questions

### Q: Do I need to use Claude?

**A**: No! MCP works with any AI model. Claude examples are just convenient.

### Q: Can I use JavaScript instead?

**A**: Yes! See the [JavaScript Edition](00-index.md).

### Q: What Python versions are supported?

**A**: Python 3.8+ (async/await support required).

### Q: How do I deploy MCP servers?

**A**: See [Production Deployment](06-mcp-complete-example-python.md) section.

### Q: Can I use MCP without AI?

**A**: Absolutely! MCP is just a protocol for connecting tools.

## Troubleshooting

### Import errors with `mcp`?

```bash
pip install --upgrade mcp
```

### Asyncio errors?

Make sure you're using `async`/`await` properly. See examples in the guides.

### Claude API errors?

Set `ANTHROPIC_API_KEY` environment variable.

## Next Steps

1. **Start Learning**: Pick a learning path above
2. **Get Your Hands Dirty**: Follow [Complete Example](06-mcp-complete-example-python.md)
3. **Apply to Your Project**: Adapt examples to your use case
4. **Join Community**: [MCP GitHub](https://github.com/modelcontextprotocol)

## Resources

- 📖 [Official MCP Documentation](https://modelcontextprotocol.io)
- 🐍 [Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- 🤖 [Claude API Documentation](https://docs.anthropic.com)
- 💬 [Community GitHub Discussions](https://github.com/modelcontextprotocol)

## About This Guide

- **Created for**: Python developers building AI applications
- **Level**: Beginner to Advanced
- **Hands-on**: Every concept has working code examples
- **Practical**: Focused on real-world applications
- **Updated**: Current with MCP protocol version 2024-11-05

---

**Ready?** Start with [MCP Overview](01-mcp-overview.md) or jump straight to [Building a Server](03-building-mcp-server-python.md)!

Happy coding! 🚀
