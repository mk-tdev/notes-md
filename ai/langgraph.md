# LangGraph Reference Guide

## Overview

LangGraph is a library for building stateful, multi-actor applications using LLMs. It extends LangChain by providing a framework for creating graphs that represent complex workflows with state management, loops, and conditional logic.

## Key Concepts

### State

- Persistent data structure that flows through the graph
- Defined using Pydantic models or TypedDict
- Accumulated across multiple steps in the graph

```python
from typing_extensions import TypedDict

class AgentState(TypedDict):
    messages: list
    next: str
    iterations: int
```

### Nodes

- Functions that process the state
- Receive state as input, return updated state
- Can be sync or async

```python
def process_node(state: AgentState) -> AgentState:
    # Process state
    state["iterations"] += 1
    return state
```

### Edges

- Connections between nodes
- Can be conditional (routing based on state)
- Two types: normal edges and conditional edges

```python
# Normal edge
graph.add_edge("node_a", "node_b")

# Conditional edge (routing)
graph.add_conditional_edges(
    "node_a",
    routing_function,
    {
        "route_1": "node_b",
        "route_2": "node_c"
    }
)
```

### Graph Types

#### StateGraph

Standard graph for most use cases:

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("node_name", node_function)
graph.add_edge("start_node", "end_node")
graph.set_entry_point("start_node")
graph.set_finish_point("end_node")
```

#### MessageGraph

Specialized for message-based workflows (deprecated in favor of StateGraph with messages).

### Compiled Graph

- Convert graph definition to runnable
- Execute with `.invoke()` or `.stream()`

```python
compiled = graph.compile()

# Invoke - get final result
result = compiled.invoke({"messages": [...]})

# Stream - get intermediate results
for event in compiled.stream({"messages": [...]}, max_iterations=10):
    print(event)
```

## Common Patterns

### Agent Loop

```python
from langgraph.graph import StateGraph, END

class AgentState(TypedDict):
    messages: list

def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]
    if "done" in last_message.content.lower():
        return END
    return "continue"

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("action", action_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {"continue": "action", END: END})
graph.add_edge("action", "agent")
```

### Sub-graphs

Compose graphs within graphs:

```python
subgraph = create_subgraph()
main_graph.add_node("subgraph_name", subgraph.compile())
```

### Parallel Processing

Use conditional edges to route to multiple nodes in parallel within a single graph step.

## Tool Integration

### Adding Tools

```python
from langchain.tools import tool

@tool
def my_tool(input: str) -> str:
    """Tool description"""
    return f"Result: {input}"

tools = [my_tool]
```

### Tool Node

```python
from langgraph.prebuilt import ToolNode

tool_node = ToolNode(tools)
graph.add_node("tools", tool_node)
```

## Memory & Persistence

### Checkpointing

Save graph state at different points:

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
graph = StateGraph(AgentState)
# ... add nodes and edges ...
compiled = graph.compile(checkpointer=checkpointer)
```

### Thread IDs

Track conversations/sessions:

```python
compiled.invoke(
    {"messages": [...]},
    config={"configurable": {"thread_id": "user_123"}}
)
```

## Execution Modes

### Invoke

```python
result = compiled.invoke(input_state, config=config)
# Returns final state after all execution
```

### Stream

```python
for output in compiled.stream(input_state, config=config):
    print(output)
# Returns intermediate outputs
```

### Stream Mode Options

- `values`: Only return node outputs
- `updates`: Only return state updates
- `debug`: Detailed step-by-step info

```python
for output in compiled.stream(input_state, config=config, stream_mode="updates"):
    print(output)
```

## Visualization

### View Graph

```python
from IPython.display import Image, display

display(Image(compiled.get_graph().draw_mermaid_png()))
```

### Export Diagram

```python
compiled.get_graph().print_ascii()
```

## Error Handling

### Retry Logic

```python
from tenacity import retry, stop_after_attempt

@retry(stop=stop_after_attempt(3))
def node_with_retry(state: AgentState) -> AgentState:
    # Logic here
    return state
```

### Try-Except in Nodes

```python
def resilient_node(state: AgentState) -> AgentState:
    try:
        # Process state
        pass
    except Exception as e:
        state["error"] = str(e)
    return state
```

## Advanced Features

### Dynamic Routing

Route based on complex state analysis:

```python
def complex_router(state: AgentState) -> str:
    if state["iterations"] > 5:
        return "conclude"
    if len(state["messages"]) > 100:
        return "summarize"
    return "continue"

graph.add_conditional_edges("analyze", complex_router, {...})
```

### Breakpoints

Pause execution at specific nodes for debugging:

```python
compiled = graph.compile(debug=True)
```

### State Modifiers

Transform state between nodes (preprocessors/postprocessors).

## Best Practices

1. **Keep Nodes Focused** - Each node should do one thing well
2. **Use Type Hints** - Define clear state structures with TypedDict
3. **Handle Edge Cases** - Include fallback paths and error nodes
4. **Test Components** - Unit test individual nodes before graph composition
5. **Monitor State Size** - Keep state lean, especially in long-running graphs
6. **Use Conditional Edges for Routing** - Don't hardcode linear paths
7. **Document Node Purposes** - Clear comments on what each node does
8. **Plan Termination** - Ensure graphs have clear exit conditions

## Common Use Cases

### Chatbot with Tools

Agent that decides when to use tools based on conversation.

### Data Processing Pipeline

Multi-step pipeline with conditional routing based on data characteristics.

### Agentic Workflows

Autonomous agents that plan, act, and observe in loops.

### Multi-Agent Systems

Multiple agents collaborating through message passing.

### QA Systems

Question analysis → Retrieval → Answer generation with feedback loops.

## Dependencies

```bash
pip install langgraph langchain langchain-community
```

## Resources

- [LangGraph Documentation](https://python.langchain.com/docs/langgraph/)
- [LangChain Documentation](https://python.langchain.com/docs/)
- GitHub: langchain-ai/langgraph
