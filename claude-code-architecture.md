---
name: claude-code-architecture
description: Understand and work with Claude Code v2.1.88 architecture, including query engine, tool system, multi-agent patterns, and execution flow. Essential for any work involving Claude Code's CLI, REPL, or headless SDK modes.
version: 2.1.88
author: Anthropic Analysis
tags:
  - claude-code
  - architecture
  - tools
  - agents
  - mcp
  - query-engine
  - multi-agent
  - repl
  - sdk
filePattern:
  - "**/claude-code/**"
  - "**/@anthropic-ai/claude-code/**"
  - "**/QueryEngine.ts"
  - "**/query.ts"
  - "**/Tool.ts"
  - "**/tools/**"
  - "**/bridge/**"
  - "**/services/mcp/**"
---

# Claude Code Architecture Skill

## Overview

Claude Code v2.1.88 is a production-grade AI agent harness built by Anthropic. It extends the basic agent loop (API call → tool execution → result) with 12 layers of progressive mechanisms: planning, sub-agents, knowledge injection, context compression, persistent tasks, background operations, agent teams, and autonomous coordination.

## Core Architecture

### Entry Points

```
cli.tsx → main.tsx (4,683 lines)
    ├─→ REPL.tsx           (interactive terminal UI)
    └─→ QueryEngine.ts     (headless/SDK mode)
```

### Query Engine Lifecycle

The heart of Claude Code is the query loop in `src/query.ts` (~785KB, largest file):

```typescript
// Simplified query flow
async function* query(prompt: string): AsyncGenerator<SDKMessage> {
  // 1. Process user input (parse /commands)
  const userMessage = processUserInput(prompt);

  // 2. Build system prompt (tools, permissions, CLAUDE.md)
  const systemPrompt = await fetchSystemPromptParts();

  // 3. Main agent loop
  while (true) {
    // 3a. Call Claude API with tools
    const response = await claudeApi(messages, tools);

    // 3b. Stream response to consumer
    for await (const event of response) {
      if (event.type === 'content_block_delta') {
        yield event; // Text or tool_use
      }

      // 3c. Execute tools when requested
      if (event.type === 'tool_use') {
        const results = await StreamingToolExecutor(event.tools);
        messages.push({ role: 'tool_result', content: results });
      }

      // 3d. Check stop condition
      if (event.stop_reason !== 'tool_use') {
        break; // Final response received
      }
    }
  }
}
```

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **QueryEngine** | `src/QueryEngine.ts` | Headless SDK entry point |
| **Main Loop** | `src/query.ts` | Core agent orchestration |
| **Tool Factory** | `src/Tool.ts` | Tool interface + builder |
| **Tool Registry** | `src/tools.ts` | Tool registration & filtering |
| **Commands** | `src/commands.ts` | Slash command definitions |
| **State** | `src/state/AppState.tsx` | Application state management |
| **Bridge** | `src/bridge/` | Claude Desktop/remote integration |

---

## Tool System

### Tool Interface

Every tool in Claude Code implements a standard interface:

```typescript
interface Tool<Input, Output, Progress> {
  // Lifecycle
  call(input: Input): Promise<ToolResult<Output>>
  validateInput?(input: Input): ValidationResult

  // Capabilities
  isEnabled?(): boolean
  isConcurrencySafe?(): boolean
  isReadOnly?(): boolean
  isDestructive?(): boolean

  // Permissions
  checkPermissions?(input: Input): PermissionCheck

  // UI Rendering (React/Ink)
  renderToolUseMessage?(input: Input): ReactNode
  renderToolResultMessage?(result: Output): ReactNode

  // AI-facing
  prompt(): string
  description(): string
}
```

### Tool Categories

| Category | Tools | Purpose |
|----------|-------|---------|
| **File Ops** | FileRead, FileEdit, FileWrite, NotebookEdit | File I/O |
| **Search** | Glob, Grep, ToolSearch | Discovery |
| **Execution** | Bash, PowerShell | Command execution |
| **Web** | WebFetch, WebSearch | HTTP/Network |
| **Agents** | Agent, SendMessage | Sub-agents |
| **Tasks** | TaskCreate, Update, Get, List, Stop | Task management |
| **Teams** | TeamCreate, TeamDelete, TodoWrite | Multi-agent teams |
| **Planning** | EnterPlanMode, ExitPlanMode | Plan workflow |
| **MCP** | MCPTool, ListMcpResources, ReadMcpResource | MCP protocol |
| **Interaction** | AskUserQuestion, Brief | User communication |

### Permission Flow

```
Tool Request
    ↓
validateInput() ──────→ Reject bad args early
    ↓
PreToolUse Hooks ──────→ User-defined (settings.json hooks)
    ↓                    Can: approve, deny, modify
Permission Rules ──────→ Match tool name/pattern
    ↓                    ├─ alwaysAllowRules
Interactive? ───────────→ ├─ alwaysDenyRules
    ↓                    └─ alwaysAskRules
checkPermissions() ────→ Tool-specific logic
    ↓
EXECUTE
```

### Tool Execution Patterns

```typescript
// Parallel-safe tools run concurrently
const parallelTools = tools.filter(t => t.isConcurrencySafe());
await Promise.all(parallelTools.map(t => t.call(input)));

// Serial tools run one at a time
for (const tool of serialTools) {
  await tool.call(input);
}
```

---

## Multi-Agent System

### Spawn Modes

| Mode | Process | Messages | Cache | Use Case |
|------|---------|----------|-------|----------|
| **default** | In-process | Shared | Shared | Simple tasks |
| **fork** | Child process | Fresh | Shared | Isolated context |
| **worktree** | Child process | Fresh | Shared | Git isolation |
| **remote** | Bridge session | Fresh | Isolated | Container/remote |

### Agent Communication

```typescript
// Send message to specific teammate
await SendMessageTool({
  to: "teammate-name",
  message: "Work on task #123"
});

// Shared task board
await TaskCreate({
  subject: "Fix authentication bug",
  description: "..."
});

await TaskUpdate({
  taskId: "123",
  status: "in_progress",
  owner: "teammate-name"
});
```

### Team Lifecycle

```typescript
// 1. Create team
await TeamCreate({
  team_name: "my-project",
  description: "Build feature X"
});

// 2. Spawn teammates
await AgentTool({
  name: "researcher",
  team_name: "my-project"
});

// 3. Assign tasks
await TaskUpdate({
  taskId: "456",
  owner: "researcher"
});

// 4. Cleanup
await TeamDelete(); // Auto-cleanup on session end
```

---

## MCP (Model Context Protocol)

### Transport Types

| Transport | Use When | Config |
|-----------|----------|--------|
| **stdio** | Child process | `{ command: "path/to/server" }` |
| **sse** | HTTP EventSource | `{ url: "http://localhost:3000/sse" }` |
| **http** | Streamable HTTP | `{ url: "http://localhost:3000" }` |
| **ws** | WebSocket | `{ url: "ws://localhost:3000" }` |
| **sdk** | In-process | Direct function calls |

### MCP Tool Wrapper

```typescript
// MCP tools are automatically wrapped
// Tool name format: mcp__<server>__<tool>

// Example usage
await mcp__my_server__search_database({
  query: "SELECT * FROM users"
});
```

### Authentication

```typescript
// OAuth 2.0 flow
{
  transport: "sse",
  url: "https://api.example.com/mcp",
  auth: {
    type: "oauth",
    client_id: "...",
    client_secret: "..."
  }
}

// API key
{
  transport: "http",
  url: "https://api.example.com/mcp",
  headers: {
    "Authorization": "Bearer ${API_KEY}"
  }
}
```

---

## Context Management

### Compression Strategies

```typescript
// Strategy 1: autoCompact (always active)
// Summarizes old messages when token limit approached
await autoCompact(messages);

// Strategy 2: snipCompact (HISTORY_SNIP feature flag)
// Removes zombie messages and stale markers
await snipCompact(messages);

// Strategy 3: contextCollapse (CONTEXT_COLLAPSE feature flag)
// Restructures context for efficiency
await contextCollapse(messages);
```

### Context Window Budget

```
┌─────────────────────────────────────────┐
│ System Prompt (~20-30%)                 │
│ • Tool definitions                      │
│ • Permission rules                      │
│ • CLAUDE.md content                     │
├─────────────────────────────────────────┤
│ Conversation History (variable)         │
│ ┌───────────────────────────────────┐  │
│ │ [compacted summary]               │  │
│ │ [compact_boundary marker]         │  │
│ │ [recent messages - full fidelity]│  │
│ └───────────────────────────────────┘  │
├─────────────────────────────────────────┤
│ Current Turn (~10-20%)                  │
│ • User message                          │
│ • Assistant response                    │
└─────────────────────────────────────────┘
```

---

## Session Persistence

### Storage Format

```bash
~/.claude/projects/<hash>/sessions/
└── <session-id>.jsonl    # Append-only log
    ├── {"type":"user",...}
    ├── {"type":"assistant",...}
    ├── {"type":"tool_use",...}
    ├── {"type":"tool_result",...}
    └── {"type":"system","subtype":"compact_boundary"}
```

### Resume Flow

```typescript
// Continue last session in cwd
claude-code --continue

// Resume specific session
claude-code --resume <session-id>

// Fork from existing session
claude-code --fork-session <session-id>
```

---

## 12-Layer Progressive Harness

Claude Code demonstrates 12 layered mechanisms for production AI agents:

1. **THE LOOP**: Basic agent loop (API → tools → results)
2. **TOOL DISPATCH**: Tool registration and execution
3. **PLANNING**: TodoWrite before execution
4. **SUB-AGENTS**: Fork processes for isolation
5. **KNOWLEDGE ON DEMAND**: SkillTool lazy loading
6. **CONTEXT COMPRESSION**: Auto-compact strategies
7. **PERSISTENT TASKS**: File-based task tracking
8. **BACKGROUND TASKS**: DreamTask async execution
9. **AGENT TEAMS**: Multi-agent coordination
10. **TEAM PROTOCOLS**: SendMessage communication
11. **AUTONOMOUS AGENTS**: Auto-claim task cycle
12. **WORKTREE ISOLATION**: Git-managed worktrees

---

## Design Patterns

| Pattern | Location | Purpose |
|---------|----------|---------|
| **AsyncGenerator** | QueryEngine | Full-chain streaming |
| **Builder + Factory** | buildTool() | Safe defaults |
| **Branded Types** | SystemPrompt | Type safety |
| **Feature Flags + DCE** | feature() | Compile-time optimization |
| **Discriminated Unions** | Message types | Type-safe handling |
| **Observer + State Machine** | StreamingToolExecutor | Lifecycle tracking |
| **Snapshot State** | FileHistoryState | Undo/redo |
| **Fire-and-Forget Write** | recordTranscript() | Non-blocking I/O |
| **Context Isolation** | AsyncLocalStorage | Per-agent context |
| **Ring Buffer** | Error log | Bounded memory |

---

## Working with Claude Code

### When to Use This Skill

Invoke this skill when:
- Reading or modifying Claude Code source
- Building tools for Claude Code
- Implementing MCP servers
- Creating custom agents or teammates
- Working with the bridge API
- Analyzing Claude Code behavior

### Key File Locations

```
src/
├── main.tsx                 # Entry point
├── QueryEngine.ts           # SDK mode
├── query.ts                 # Main loop (785KB)
├── Tool.ts                  # Tool interface
├── tools.ts                 # Tool registry
├── tools/                   # 40+ tool implementations
├── services/
│   ├── api/                 # Claude API client
│   ├── mcp/                 # MCP client
│   └── tools/               # Tool executor
├── bridge/                  # Claude Desktop integration
└── state/                   # Application state
```

### Common Patterns

#### Tool Definition

```typescript
export default buildTool({
  name: "my-tool",
  description: "Does something useful",
  inputSchema: {
    type: "object",
    properties: {
      input: { type: "string" }
    },
    required: ["input"]
  },

  async call({ input }) {
    // Implementation
    return { result: "done" };
  },

  // Optional capabilities
  isReadOnly: () => true,
  isConcurrencySafe: () => true
});
```

#### Command Definition

```typescript
export default {
  name: "my-command",
  description: "Does something",
  options: [
    { name: "option", description: "An option" }
  ],

  async execute(args) {
    // Implementation
  }
};
```

---

## Best Practices

### For Tool Development

1. **Always validate input** before processing
2. **Declare capabilities** (readOnly, destructive, concurrencySafe)
3. **Use specific types** for input/output
4. **Handle errors gracefully** with clear messages
5. **Consider permissions** - what should be allowed by default?

### For Agent Development

1. **Use forks for isolation** when tasks are independent
2. **Share tasks via board** instead of direct messages
3. **Clean up resources** (teams, tasks) when done
4. **Handle errors** - don't leave tasks in limbo
5. **Use descriptive names** for agents and tasks

### For MCP Integration

1. **Follow MCP spec** for tool schemas
2. **Handle authentication** properly (OAuth, API keys)
3. **Provide clear descriptions** for tools
4. **Support progress updates** for long operations
5. **Test locally** before deploying

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Tool not found | Not registered | Add to tools.ts or MCP server |
| Permission denied | Rule mismatch | Check alwaysAllow/alwaysDeny rules |
| Context overflow | No compression | Enable autoCompact |
| Agent not responding | Deadlock | Check message inbox |
| MCP connection fail | Transport mismatch | Verify server config |

### Debug Flags

```bash
# Enable verbose logging
VERCEL_LOG=trace claude-code

# Dump system prompt
DUMP_SYSTEM_PROMPT=1 claude-code

# Disable analytics
DISABLE_ANALYTICS=1 claude-code
```

---

## References

- **Main Repository**: `@anthropic-ai/claude-code` npm package
- **Documentation**: README.md in source root
- **MCP Spec**: https://modelcontextprotocol.io
- **API Docs**: Anthropic Messages API reference

---

## Version Notes

- **Current Version**: 2.1.88
- **Last Updated**: 2026-04-01
- **Source Lines**: ~512K
- **Files**: ~1,884 TypeScript files
- **Tools**: 43 built-in + MCP extensions

---

> This skill is based on decompiled source analysis. For official documentation, refer to Anthropic's published materials.
