# Test

## How MCP Works: A Small Demonstration

The **Model Context Protocol (MCP)** is an open standard that lets AI applications (hosts/clients) talk to external tools and data sources (servers) through a common JSON-RPC 2.0 interface.

### Architecture

```
+-----------+      +-------------+   JSON-RPC   +-------------+
|   User    | ---> | MCP Client  | <==========> | MCP Server  | ---> Tools / Files / APIs
| (LLM app) |      | (in host)   |  stdio/HTTP  |             |
+-----------+      +-------------+              +-------------+
```

- **Host**: the AI app (e.g. a chat assistant or IDE).
- **Client**: connector inside the host, one per server.
- **Server**: exposes **tools** (actions), **resources** (data) and **prompts** (templates).

### Message flow

1. **Initialize** – client and server negotiate protocol version and capabilities.
2. **Discover** – client asks what tools the server offers (`tools/list`).
3. **Call** – the model decides to use a tool; client sends `tools/call`.
4. **Respond** – server returns the result, which is fed back to the model.

Example exchange:

```jsonc
// Client -> Server: list tools
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }

// Server -> Client
{
  "jsonrpc": "2.0", "id": 1,
  "result": {
    "tools": [{
      "name": "add",
      "description": "Add two numbers",
      "inputSchema": {
        "type": "object",
        "properties": { "a": {"type": "number"}, "b": {"type": "number"} },
        "required": ["a", "b"]
      }
    }]
  }
}

// Client -> Server: call the tool
{
  "jsonrpc": "2.0", "id": 2, "method": "tools/call",
  "params": { "name": "add", "arguments": { "a": 2, "b": 3 } }
}

// Server -> Client
{
  "jsonrpc": "2.0", "id": 2,
  "result": { "content": [{ "type": "text", "text": "5" }] }
}
```

### Minimal server (Python)

Install the SDK: `pip install "mcp[cli]"`

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

if __name__ == "__main__":
    mcp.run()  # communicates over stdio
```

### Minimal client (Python)

```python
# client.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    params = StdioServerParameters(command="python", args=["server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            print([t.name for t in tools.tools])           # ['add']
            result = await session.call_tool("add", {"a": 2, "b": 3})
            print(result.content[0].text)                  # 5

asyncio.run(main())
```

In a real application, an LLM sits between the user and the client: it reads the tool list, chooses a tool and arguments, and uses the returned result to craft its answer.
