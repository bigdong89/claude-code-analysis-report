# Claude Code Agent Architecture Deep-Dive

Concrete implementation patterns extracted from the Claude Code source code.

## Directory Structure

```
src/
├── tools/AgentTool/           # Agent tool implementation
│   ├── prompt.ts              # Tool description and usage guidance
│   ├── agentToolUtils.ts      # Tool filtering, lifecycle, progress
│   ├── forkSubagent.ts        # Fork pattern (context inheritance)
│   ├── runAgent.ts           # Agent execution engine
│   ├── loadAgentsDir.ts       # Agent definition loading & parsing
│   ├── builtInAgents.ts       # Built-in agent registry
│   ├── agentMemory.ts        # Persistent memory system
│   ├── agentMemorySnapshot.ts # Memory snapshot management
│   ├── constants.ts          # Tool name constants
│   └── built-in/
│       ├── generalPurposeAgent.ts  # Full-capability agent
│       ├── exploreAgent.ts         # Read-only search agent
│       ├── planAgent.ts           # Architecture planning agent
│       ├── verificationAgent.ts   # Post-implementation verification
│       └── claudeCodeGuideAgent.ts # Claude Code docs expert
├── coordinator/               # Multi-agent orchestration
│   ├── AgentCoordinator.ts    # Central coordination layer
│   ├── TeammateImpl.ts        # Teammate agent implementation
│   └── WorktreeManager.ts     # Worktree isolation management
├── utils/
│   ├── teammateMailbox.ts    # File-based message passing
│   ├── teammateContext.ts     # In-process teammate detection
│   └── swarm/                # Swarm utilities
│       ├── teammateModel.ts   # Teammate model configuration
│       ├── teammateInit.ts    # Teammate initialization
│       └── teamHelpers.ts     # Team coordination helpers
└── tasks/
    └── LocalAgentTask/       # Async agent task management
```

## AgentDefinition Type Hierarchy

```typescript
// Base — common fields for all agents
BaseAgentDefinition {
  agentType: string           // Unique identifier
  whenToUse: string           // Routing signal for orchestrator
  tools?: string[]            // Tool allowlist
  disallowedTools?: string[]  // Tool denylist
  model?: string              // LLM model selection
  maxTurns?: number           // Loop limit
  permissionMode?: PermissionMode
  memory?: 'user' | 'project' | 'local'
  isolation?: 'worktree' | 'remote'
  background?: boolean         // Always run async
  omitClaudeMd?: boolean       // Skip CLAUDE.md loading
}

// Built-in — dynamic system prompts
BuiltInAgentDefinition extends BaseAgentDefinition {
  source: 'built-in'
  getSystemPrompt: (ctx) => string  // Dynamic prompt generation
}

// Custom — user/project-defined agents
CustomAgentDefinition extends BaseAgentDefinition {
  source: SettingSource  // 'userSettings' | 'projectSettings' | etc.
  getSystemPrompt: () => string
  filename?: string
}

// Plugin — third-party extensions
PluginAgentDefinition extends BaseAgentDefinition {
  source: 'plugin'
  plugin: string  // Plugin identifier
}
```

## Tool Filtering Pipeline

```typescript
// From agentToolUtils.ts — layered filtering
function filterToolsForAgent(tools, { isBuiltIn, isAsync, permissionMode }) {
  return tools.filter(tool => {
    // 1. MCP tools always pass through
    if (tool.name.startsWith('mcp__')) return true
    // 2. Global disallow list (dangerous tools)
    if (ALL_AGENT_DISALLOWED_TO_TOOLS.has(tool.name)) return false
    // 3. Custom agent extra restrictions
    if (!isBuiltIn && CUSTOM_AGENT_DISALLOWED_TOOLS.has(tool.name)) return false
    // 4. Async agents: only safe tools allowed
    if (isAsync && !ASYNC_AGENT_ALLOWED_TOOLS.has(tool.name)) return false
    return true
  })
}

function resolveAgentTools(agentDef, availableTools, isAsync, isMainThread) {
  // 1. Apply global filtering (unless main thread)
  // 2. Apply agent's disallowedTools
  // 3. Expand agent's tools (handle wildcards)
  // 4. Validate against available tools
  // 5. Return resolved set with valid/invalid tracking
}
```

## Async Agent Lifecycle

```typescript
// From agentToolUtils.ts — the complete lifecycle
async function runAsyncAgentLifecycle({
  taskId, abortController, makeStream, metadata, ...
}) {
  const tracker = createProgressTracker()

  for await (const message of makeStream(onCacheSafeParams)) {
    agentMessages.push(message)
    updateProgressFromMessage(tracker, message, ...)
    updateAsyncAgentProgress(taskId, progress, setAppState)
    emitTaskProgress(tracker, taskId, ...)
  }

  // IMPORTANT: Complete BEFORE safety classification
  completeAsyncAgent(agentResult, setAppState)

  const handoffWarning = await classifyHandoffIfNeeded(...)
  if (handoffWarning) finalMessage = handoffWarning + '\n\n' + finalMessage

  enqueueAgentNotification({ taskId, status: 'completed', ... })
}
```

## Coordinator Architecture

```typescript
// From AgentCoordinator.ts — key internal classes

class AgentRegistry {
  // Agent CRUD with name indexing
  register(agent): void
  unregister(agentId): AgentState | undefined
  get(agentId): AgentState | undefined
  getIdleAgents(capabilities?): AgentState[]
  assignTask(agentId, taskId): void
  unassignTask(agentId, taskId): void
  getIdleAgentsSorted(): AgentState[]  // Least recently active first
  countByStatus(status): number
  getIdleAgentsTimeout(timeoutMs): AgentState[]
}

class TaskBoard {
  // Task lifecycle with dependency tracking
  createTask(task): CoordinatorTask
  getReadyTasks(): CoordinatorTask[]  // Dependencies satisfied
  getPendingTasksSorted(): CoordinatorTask[]  // By priority
  assignTask(taskId, agentId): boolean
  completeTask(taskId): void      // Unblock dependents
  failTask(taskId, error): void
  cleanupOldTasks(olderThanMs): string[]
}

class MessageBus {
  send(message): string           // Direct or broadcast
  getPending(agentId): AgentMessage[]
  peekPending(agentId): AgentMessage[]  // Non-consuming
}
```

## Fork Subagent Pattern

```typescript
// From forkSubagent.ts — context inheritance for cheap parallelism

// Fork inherits parent's conversation by:
// 1. Keeping the full parent assistant message (all tool_use blocks)
// 2. Building placeholder tool_results for every tool_use
// 3. Appending the child-specific directive as final text block
//
// Result: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
// Only the final text block differs per child → maximizes prompt cache hits

function buildForkedMessages(directive, assistantMessage): MessageType[] {
  const fullAssistantMessage = { ...assistantMessage, content: [...assistantMessage.content] }
  const toolUseBlocks = assistantMessage.message.content.filter(b => b.type === 'tool_use')
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result',
    tool_use_id: block.id,
    content: [{ type: 'text', text: 'Fork started — processing in background' }]
  }))
  return [fullAssistantMessage, { type: 'user', content: [...toolResultBlocks, { type: 'text', text: childMessage }] }]
}
```

## Message Passing System

```typescript
// From teammateMailbox.ts — file-based async messaging

// Messages stored as JSON files
// Path: .claude/teams/{team}/inboxes/{agent}.json
// Features:
// - Read receipts (message locking)
// - Concurrency control (file-level locking)
// - Automatic delivery ordering
```

## Built-in Agent Configurations

### General Purpose Agent
- `tools: ['*']` — all tools
- No model override (uses default)
- No turn limit specified
- Full CLAUDE.md loaded
- Prompt: "Complete the task fully — don't gold-plate, but don't leave it half-done"

### Explore Agent
- `disallowedTools: [AgentTool, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]`
- Model: `inherit` (internal) / `haiku` (external) — fast for search
- `omitClaudeMd: true` — saves tokens, main agent has full context
- Key rule: "parallel tool calls for speed"

### Plan Agent
- Same disallowed tools as Explore
- Model: `inherit`
- `omitClaudeMd: true`
- Structured output: ends with "Critical Files for Implementation" section
- Process: Understand → Explore → Design → Detail Plan

### Key Design Decisions

1. **Explore uses haiku model**: Read-only search doesn't need frontier model
2. **Plan uses inherit model**: Needs same context understanding as main agent
3. **Explore omits CLAUDE.md**: Saves ~5-15 Gtok/week across millions of spawns
4. **No AgentTool in subagents**: Prevents recursive delegation depth issues
5. **Worktree isolation**: Default for parallel implementation agents
