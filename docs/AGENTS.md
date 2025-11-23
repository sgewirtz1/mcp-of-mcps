# MCP of MCPs - Architecture Guide

> **For Developers**: A practical guide to understanding and working with this codebase

## What Does This Project Do?

MCP of MCPs is a **meta-server** that connects to multiple MCP servers and provides 4 powerful tools:

1. **semantic_search_tools** - Find tools by describing what you want to do
2. **get_mcps_servers_overview** - See all available tools (lightweight list)
3. **get_tools_overview** - Get detailed schemas for specific tools
4. **run_functions_code** - Execute JavaScript that orchestrates multiple tools

**The Big Idea**: Instead of loading 100+ tool definitions upfront (burning tokens), load only what you need, when you need it. Then execute complex workflows in a sandbox where data flows directly between tools.

---

## Project Structure

```
src/
├── index.ts                    # Entry point - starts everything
├── config/                     # Configuration & setup
│   ├── AppFactory.ts          # Creates and wires all components (DI)
│   └── ConfigLoader.ts        # Parses CLI arguments
├── domain/                     # Core types and interfaces
│   └── types.ts               # All TypeScript interfaces
├── interfaces/                 # Contracts for components
│   ├── IConnectionManager.ts
│   ├── IServerRegistry.ts
│   ├── IToolsParser.ts
│   ├── IVectorStore.ts
│   ├── ISandboxManager.ts
│   └── IToolDatabase.ts       # NEW: Tool database interface
├── presentation/               # MCP protocol layer
│   ├── McpOfMcps.ts           # Main MCP server
│   └── prompts/
│       └── PromptDefinitions.ts  # Tool schemas
├── application/                # Business logic
│   ├── handlers/
│   │   └── ToolCallHandler.ts    # Handles tool execution
│   ├── services/
│   │   ├── ServerRegistryService.ts  # Manages servers/tools
│   │   └── ToolsParserService.ts     # Formats tool info
│   └── validators/
│       └── ArgsValidator.ts      # Input validation
└── infrastructure/             # External integrations
    ├── connection/
    │   └── ConnectionManager.ts   # MCP client connections
    ├── database/                  # NEW: Persistent storage
    │   └── serversToolDatabaseImpl.ts  # Tool metadata database
    ├── storage/
    │   ├── VectorStoreImpl.ts     # Semantic search
    │   └── EmbeddingsManager.ts   # Text → vectors
    └── sandbox/
        └── SandboxManagerImpl.ts  # Code execution
```

---

## How It Works: The Flow

### 1. Startup Sequence

```
1. index.ts loads config from CLI args
2. AppFactory creates all components with dependencies
3. McpOfMcps.start() initializes everything:
   ├─> Initializes tool database (.database/mcps.db)
   ├─> Connects to child MCP servers (weather, time, etc.)
   ├─> Fetches tools from each server and syncs to database
   ├─> Cleans up orphaned servers from database
   ├─> Builds vector index for semantic search
   └─> Creates sandbox with tool wrappers
4. Ready to accept MCP requests on stdio
```

### 2. Request Flow

```
MCP Client (Claude/Cline)
    ↓
McpOfMcps (routes request)
    ↓
ToolCallHandler (handles logic)
    ↓
├─> ServerRegistryService (tool metadata)
├─> VectorStore (semantic search)
├─> SandboxManager (code execution)
└─> Child MCP Servers (actual tools)
```

---

## Key Components Explained

### AppFactory (config/AppFactory.ts)

**What it does**: Creates all components and wires them together

```typescript
// Infrastructure first
connectionManager = new ConnectionManager()
vectorStore = new VectorStoreImpl()
sandboxManager = new SandboxManagerImpl()

// Application layer
serverRegistry = new ServerRegistryService(connectionManager)
toolCallHandler = new ToolCallHandler(registry, parser, sandbox, vector)

// Presentation layer
return new McpOfMcps(config, handler, registry, ...)
```

**Why**: Clean dependency injection without frameworks

---

### McpOfMcps (presentation/McpOfMcps.ts)

**What it does**: Main MCP server that implements the protocol

**Key responsibilities**:
- Handles `ListTools` requests (returns 4 meta-tools)
- Routes `CallTool` requests to ToolCallHandler
- Manages server lifecycle

**How to add a new tool**:
1. Define schema in `PromptDefinitions.ts`
2. Add validator in `ArgsValidator.ts`
3. Add handler in `ToolCallHandler.ts`
4. Route in `setupHandlers()`

---

### ToolCallHandler (application/handlers/ToolCallHandler.ts)

**What it does**: Executes the 4 meta-tools

```typescript
handleGetServersOverview()     // Returns tool list
handleGetToolsOverview(args)   // Returns tool schemas
handleSemanticSearch(args)     // Searches by description
handleRunCode(args)            // Executes JavaScript
```

**Pattern**: One method per tool, delegates to services

---

### ServerRegistryService (application/services/ServerRegistryService.ts)

**What it does**: Central registry of all connected servers and their tools

**Key methods**:
```typescript
registerServer(name)           // Connect and fetch tools
getServer(name)                // Get server info
getTool(server, tool)          // Find specific tool
getAllServers()                // Get everything
```

**Data stored**: `Map<serverName, ServerInfo>` where ServerInfo contains:
- Server name
- MCP Client connection
- Array of available tools

---

### VectorStoreImpl (infrastructure/storage/VectorStoreImpl.ts)

**What it does**: Semantic search over all tools

**How it works**:
1. At startup: Generate embeddings for all tool descriptions
2. Store in vector database (`.vector-index/`)
3. On search: Convert query → embedding → find similar tools

**Technology**:
- `@xenova/transformers` - Local ML model for embeddings
- `vectra` - Local vector database
- No external API calls needed

---

### SandboxManagerImpl (infrastructure/sandbox/SandboxManagerImpl.ts)

**What it does**: Executes user JavaScript code safely

**How it works**:
1. Creates `.sandbox/` directory structure:
   ```
   .sandbox/
   ├── weather/
   │   └── get_forecast.cjs
   ├── time/
   │   └── get_current_time.cjs
   ```

2. Each `.cjs` file is a wrapper that calls the real MCP tool:
   ```javascript
   async function get_forecast(args) {
     const client = serversInfo.get("weather").client;
     return await client.callTool({ name: "get_forecast", arguments: args });
   }
   ```

3. User code runs in `vm2` sandbox and can `require()` these wrappers

4. Data flows directly between tools, only final result returns

**Security**: Isolated VM, no file system access, limited require()

---

### ConnectionManager (infrastructure/connection/ConnectionManager.ts)

**What it does**: Manages connections to child MCP servers

**Key methods**:
```typescript
createConnection(config)       // Connect via stdio
getConnection(name)            // Get existing client
getAllConnections()            // Get all clients
```

**How connections work**:
- Each child server runs as subprocess
- Communication via stdio (stdin/stdout)
- MCP protocol for structured requests

---

### ServersToolDatabaseImpl (infrastructure/database/serversToolDatabaseImpl.ts) **NEW**

**What it does**: Persistent storage for tool metadata (specifically output schemas)

**Why it exists**: Some MCP servers don't provide output schemas consistently. This database preserves them across sessions so tools maintain consistent behavior.

**Key features**:
- **Singleton pattern** - Only one database instance exists
- **Auto-sync** - Automatically syncs tool metadata on startup
- **Orphan cleanup** - Removes tools from deleted servers
- **Preserves output schemas** - Stores output schemas even when servers return undefined

**Key methods**:
```typescript
initialize()                   // Create database and schema
saveTool(tool)                // Insert or update tool metadata
getTool(server, tool)         // Retrieve stored tool
getServerTools(server)        // Get all tools for a server
deleteServerTools(server)     // Clean up removed servers
getStats()                    // Get database statistics
```

**Database schema**:
```sql
CREATE TABLE tools (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  serverName TEXT NOT NULL,
  toolName TEXT NOT NULL,
  outputSchema TEXT,              -- JSON string of output schema
  originalOutputSchema INTEGER,   -- 1 if from server, 0 if preserved
  lastUpdated INTEGER NOT NULL,   -- Unix timestamp
  UNIQUE(serverName, toolName)
)
```

**Storage location**: `.database/mcps.db` (automatically created, gitignored)

**Integration points**:
- **ServerRegistryService** calls database during tool sync
- **SandboxManagerImpl** updates output schemas after code execution
- **AppFactory** creates singleton instance

---

## Data Models (domain/types.ts)

```typescript
// Configuration for a child server
interface McpServerConnectionConfig {
  name: string          // "weather"
  command: string       // "npx"
  args: string[]        // ["-y", "@h1deya/mcp-server-weather"]
}

// Stored server information
interface ServerInfo {
  name: string          // Server name
  client: Client        // MCP connection
  tools: Tool[]         // Available tools
}

// Tool with metadata for search
interface ToolWithMetadata {
  serverName: string
  toolName: string
  description: string
  tool: Tool            // Full tool definition
}

// Search result with similarity score
interface SearchResult extends ToolWithMetadata {
  score: number         // 0.0 to 1.0
}
```

---

## Common Development Tasks

### Adding a New Meta-Tool

1. **Define the tool** in `src/presentation/prompts/PromptDefinitions.ts`:
```typescript
export const MY_TOOL: Tool = {
  name: "my_tool",
  description: "What it does",
  inputSchema: { /* zod-compatible schema */ }
};
```

2. **Add validation** in `src/application/validators/ArgsValidator.ts`:
```typescript
static validateMyTool(args: unknown): ValidationResult<MyToolArgs> {
  const schema = z.object({ /* define schema */ });
  return this.validate(schema, args);
}
```

3. **Implement handler** in `src/application/handlers/ToolCallHandler.ts`:
```typescript
async handleMyTool(args: MyToolArgs): Promise<ToolResponse> {
  // Your logic here
  return { content: [{ type: "text", text: result }] };
}
```

4. **Route it** in `src/presentation/McpOfMcps.ts`:
```typescript
case "my_tool": {
  const validation = ArgsValidator.validateMyTool(args);
  if (!validation.success) return errorResponse(validation.error);
  return await this.toolCallHandler.handleMyTool(validation.data);
}
```

### Testing Locally

```bash
# Build the project
npm run build

# Run with config
node build/index.js --config '[{"name":"weather","command":"npx","args":["-y","@h1deya/mcp-server-weather"]}]'

# Or with config file
node build/index.js --config-file ./config.json
```

### Debugging

- All logs go to `stderr` (use `2>debug.log`)
- Check `.vector-index/` directory exists
- Check `.sandbox/` structure is created
- Verify child servers connect successfully

---

## Design Patterns Used

### 1. Layered Architecture
- **Presentation** (MCP protocol) → **Application** (logic) → **Infrastructure** (external)
- Dependencies flow inward

### 2. Dependency Injection
- Manual DI via AppFactory
- Components receive dependencies in constructor
- Easy to test and swap implementations

### 3. Interface Segregation
- All major components implement interfaces (`I*`)
- Loose coupling between layers

### 4. Repository Pattern
- ServerRegistryService is the single source of truth for server data

### 5. Command/Handler Pattern
- Each tool has its own handler method
- Clear separation of concerns

---

## Technology Choices

| Library | Purpose | Why |
|---------|---------|-----|
| `@modelcontextprotocol/sdk` | MCP protocol | Official MCP implementation |
| `@xenova/transformers` | Embeddings | Runs locally, no API calls |
| `vectra` | Vector DB | Simple, local, no setup |
| `better-sqlite3` | Storage | Used by vectra, reliable |
| `vm2` | Sandbox | Secure code execution |
| `zod` | Validation | Type-safe validation |

---

## Common Questions

### Q: How does semantic search work?

A: 
1. At startup, convert all tool descriptions to vectors (embeddings)
2. Store in local vector database
3. When user searches, convert query to vector
4. Find most similar tool vectors
5. Return tools ranked by similarity score

### Q: How is code execution secure?

A:
- Runs in `vm2` isolated context
- No access to file system
- Can only call MCP tools via wrappers
- No network access except through tools

### Q: How do I add a new child server?

Just add to config:
```json
{
  "name": "my-server",
  "command": "npx",
  "args": ["-y", "my-mcp-server"]
}
```

### Q: Where is data stored?

- **Tool metadata database**: `.database/mcps.db` (persistent, gitignored)
- **Vector index**: `.vector-index/` (SQLite, recreated on startup)
- **Sandbox**: `.sandbox/` (generated .cjs files, recreated on startup)

### Q: How do I modify the sandbox?

Edit `SandboxManagerImpl.generateToolFile()` to change wrapper template

---

## Quick Reference: File Purposes

| File | One-Line Summary |
|------|------------------|
| `index.ts` | Entry point, starts server |
| `AppFactory.ts` | Creates and wires all components |
| `ConfigLoader.ts` | Parses CLI arguments |
| `McpOfMcps.ts` | Main MCP server implementation |
| `ToolCallHandler.ts` | Executes the 4 meta-tools |
| `ServerRegistryService.ts` | Manages server connections and tools |
| `ToolsParserService.ts` | Formats tool information |
| `ConnectionManager.ts` | Connects to child MCP servers |
| `serversToolDatabaseImpl.ts` | **NEW:** Persistent tool metadata storage |
| `VectorStoreImpl.ts` | Semantic search implementation |
| `EmbeddingsManager.ts` | Text to vector conversion |
| `SandboxManagerImpl.ts` | Secure code execution |
| `ArgsValidator.ts` | Input validation |
| `PromptDefinitions.ts` | Tool schemas |
| `IToolDatabase.ts` | **NEW:** Database interface |
| `types.ts` | TypeScript interfaces |

---

## Next Steps for Contributors

1. **Read the README.md** for high-level understanding
2. **Follow a request** through the codebase (start at `McpOfMcps.setupHandlers()`)
3. **Run locally** with a simple config
4. **Add console.error()** logs to understand flow
5. **Try adding** a simple meta-tool
6. **Explore** the sandbox structure in `.sandbox/`

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     MCP Client                          │
│                  (Claude / Cline)                       │
└────────────────────┬────────────────────────────────────┘
                     │ stdio
                     ↓
┌─────────────────────────────────────────────────────────┐
│                   McpOfMcps                             │
│              (Presentation Layer)                        │
│  - Handles MCP protocol                                 │
│  - Routes requests to ToolCallHandler                   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────────────┐
│                ToolCallHandler                          │
│              (Application Layer)                         │
│  - Executes tool logic                                  │
│  - Delegates to services                                │
└─────┬───────────┬────────────┬────────────┬────────────┘
      │           │            │            │
      ↓           ↓            ↓            ↓
┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│Registry │ │  Tools   │ │  Vector  │ │ Sandbox  │
│Service  │ │  Parser  │ │  Store   │ │ Manager  │
└────┬────┘ └──────────┘ └────┬─────┘ └────┬─────┘
     │                         │            │
     │                         ↓            │
     │                    ┌──────────┐      │
     │                    │Embeddings│      │
     │                    │ Manager  │      │
     │                    └──────────┘      │
     ↓                                      ↓
┌─────────────┐                      ┌──────────┐
│ Connection  │                      │   vm2    │
│  Manager    │                      │ Sandbox  │
└──────┬──────┘                      └────┬─────┘
       │                                  │
       ↓                                  ↓
┌─────────────────────────────────────────────┐
│        Child MCP Servers                    │
│  (weather, time, database, etc.)            │
└─────────────────────────────────────────────┘
```

---
