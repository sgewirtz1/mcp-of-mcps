# MCP of MCPs

A meta-server that aggregates multiple MCP servers and enables efficient code execution for AI agents. Reduces token usage, improving speed and reliability.

## The Problem

When AI agents connect to multiple MCP servers, they face three critical issues:

### 1. 🔥 Burns Tokens
Every tool definition and intermediate result consumes valuable context window space. With hundreds of tools across multiple servers, agents can waste 150,000+ tokens on a single workflow.

### 2. ⏱️ Slows Everything Down  
The model must process and reason about all tool definitions upfront, even for tools it won't use. This adds significant latency before the agent can even start working.

### 3. 🤯 Increases Hallucinations
When models see too much irrelevant information, they're more likely to get confused and make mistakes. Loading all tools upfront reduces accuracy.

## The Solution

MCP of MCPs is a meta-server that provides three powerful tools for efficient agent orchestration:

### Tool 1: `get_mcps_servers_overview`
**Discovery Tool** - This tool returns only tool names without full schemas, giving agents a lightweight overview in seconds instead of loading hundreds of detailed definitions upfront. By showing just what's available without overwhelming details, it prevents confusion and hallucinations while eliminating loading delays.

```javascript
// Returns:
// google_drive/download_file
// google_drive/upload_file
// slack/send_message
// database/execute_query
// ...
```


### Tool 2: `get_tools_overview`  
**Introspection Tool** - Load only the tools you actually need instead of all 30+ tools, saving thousands of tokens through selective loading. This on-demand approach provides faster responses and focused context that reduces confusion and improves accuracy.


```javascript
// Input: ["google_drive/download_file", "slack/send_message"]
// Returns: Full tool definitions with:
// - Parameter schemas
// - Required vs optional fields
// - Example usage code
```


### Tool 3: `run_functions_code`
**Execution Tool** - Data flows directly between tools without round-trips through the model, so a 2MB file transfer uses ~100 tokens instead of 50,000+. The model only sees clean final results instead of noisy intermediate data, executing complex workflows in one shot without processing delays.

```javascript
// Write code that:
// - Calls multiple tools in sequence or parallel
// - Processes and transforms data
// - Implements complex logic and error handling
// - Returns only final results to the model
```

## How The Full Flow Solves All Problems

The three tools work together through **progressive disclosure**: first, `get_mcps_servers_overview` returns just tool names (not full schemas), so the model scans a simple list instead of parsing 500KB of definitions. Next, `get_tools_overview` loads only the 2-3 tools you need instead of all 30+, reducing tokens and giving the model focused context without confusing irrelevant options. Finally, `run_functions_code` executes workflows where data flows directly between tools in memory—intermediate results not get serialized into tokens, they stay as native objects passing from one tool to another while the model only sees the final result. This pattern cuts token usage, speeds up execution by avoiding unnecessary model processing, and eliminates hallucinations by showing only relevant information at each step.

### Real-World Example

**Traditional Approach:**
```
TOOL CALL: gdrive.getDocument(documentId: "abc123")
  → returns full transcript text (loads into context: 50,000 tokens)
  
TOOL CALL: salesforce.updateRecord(...)
  → model writes entire transcript again (doubles the tokens: +50,000 tokens)
```

**Total: 100,000+ tokens processed, slow response time**

**With MCP of MCPs:**
```javascript
const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  data: { Notes: transcript }
});
```

The code executes in one operation. Data flows directly between tools. Only the final result returns to the model.

**Total: 2,000 tokens processed (98.7% reduction) ⚡**

## Key Benefits

✅ **Faster Response Time** - No need to load all tools upfront  
✅ **Reduced Hallucinations** - Model sees only relevant information  
✅ **Progressive Disclosure** - Load tools on-demand as needed  
✅ **Code Composition** - Orchestrate complex workflows with familiar JavaScript  

## Setup

### Prerequisites

- Node.js 
- npm or yarn
- Configured MCP servers you want to aggregate

### Installation

1. **Clone the repository**
```bash
git clone https://github.com/eliavamar/mcp-of-mcps.git
cd mcp-of-mcps
```

2. **Install dependencies**
```bash
npm install
```

3. **Build the project**
```bash
npm run build
```

### Add to Cline

Add this to your Cline MCP settings file:

**Option 1: Using inline configuration (recommended)**
```json
{
  "mcpServers": {
    "mcp-of-mcps": {
      "autoApprove": [],
      "disabled": false,
      "timeout": 60,
      "type": "stdio",
      "command": "node",
      "args": [
        "/absolute/path/to/mcp-of-mcps/build/index.js",
        "--config",
        "[{\"name\":\"weather\",\"command\":\"npx\",\"args\":[\"-y\",\"@h1deya/mcp-server-weather\"]},{\"name\":\"time\",\"command\":\"uvx\",\"args\":[\"mcp-server-time\"]}]"
      ]
    }
  }
}
```

**Option 2: Using a config file**

First, create a `config.json` file that specifies which MCP servers to connect to:

```json
[
  {
    "name": "weather",
    "command": "npx",
    "args": ["-y", "@h1deya/mcp-server-weather"]
  },
  {
    "name": "time",
    "command": "uvx",
    "args": ["mcp-server-time"]
  }
]
```

Then reference this file in your Cline settings:

```json
{
  "mcpServers": {
    "mcp-of-mcps": {
      "autoApprove": [],
      "disabled": false,
      "timeout": 60,
      "type": "stdio",
      "command": "node",
      "args": [
        "/absolute/path/to/mcp-of-mcps/build/index.js",
        "/absolute/path/to/your/config.json"
      ]
    }
  }
}
```

**Configuration Options:**
- `autoApprove`: Array of tool names that don't require approval (e.g., `["get_mcps_servers_overview"]`)
- `disabled`: Set to `false` to enable the server
- `timeout`: Timeout in seconds for tool execution (default: 60)
- `type`: Connection type, always `"stdio"` for MCP servers

## Learn More

This implementation follows the patterns described in Anthropic's article on code execution with MCP:  
📖 [Code execution with MCP: Building more efficient agents](https://www.anthropic.com/engineering/code-execution-with-mcp)

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

ISC
