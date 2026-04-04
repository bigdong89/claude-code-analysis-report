---
name: ai-agent-guide
description: |
  Comprehensive guide for designing, building, and improving AI agents. Use when:
  - Building a new AI agent (chatbot, autonomous agent, multi-agent system, tool-using agent)
  - Designing agent architecture (tool selection, memory, planning, orchestration)
  - Writing or improving agent prompts, system prompts, or agent definitions
  - Implementing agent coordination patterns (delegation, teams, swarms, pipelines)
  - Adding tool use, function calling, or action capabilities to agents
  - Designing agent safety, permissions, and guardrails
  - Troubleshooting agent issues (hallucination, context loss, tool misuse, loop detection)
  - Creating agent skills, plugins, or extensions
  - Planning agent features (requirements, specs, architecture)
  Keywords: agent, AI agent, LLM agent, subagent, autonomous agent, multi-agent, tool use, function calling, agent architecture, agent design, prompt engineering, agent coordination, swarm, delegation, agent skill, agent plugin
---

# AI Agent Development Guide

Proven patterns and architecture from Claude Code's agent system, applicable to any AI agent project.

## Agent Definition Anatomy

Every agent needs these core properties:

```
name:          Unique identifier (lowercase, hyphen-separated)
description:   When-to-use trigger for the orchestrator (primary routing signal)
model:         Which LLM runs this agent (match capability to task complexity)
tools:         Allowlist of tools the agent can use (restrict to minimum needed)
maxTurns:      Agentic loop limit (prevent runaway agents)
permissionMode: auto | plan | bypass (controls human-in-the-loop)
```

## Tool Design Principles

### Tool Schema
- Use Zod/JSON Schema for input validation
- Every tool needs: `name`, `description` (routing signal), `parameters`
- Descriptions are primary routing signals — invest in clarity
- Keep parameter count low; prefer structured objects over many primitives

### Tool Selection Matrix

| Agent Type | Tool Strategy | Rationale |
|---|---|---|
| Read-only (explore, plan) | Deny write tools entirely | Prevents side effects |
| Implementation | Allow write + relevant read tools | Full capability for code changes |
| General-purpose | `tools: ['*']` (all tools) | Maximum flexibility |
| Specialized | Explicit allowlist | Minimum privilege |

Pattern from Claude Code:
```
// Explore agent — read-only, no side effects
disallowedTools: [AgentTool, ExitPlanMode, FileEdit, FileWrite, NotebookEdit]

// General purpose — all tools available
tools: ['*']
```

### Tool Filtering Architecture

```
Available Tools
  -> Global disallow list (ALL_AGENT_DISALLOWED_TOOLS)
    -> Source-specific disallow list (CUSTOM_AGENT_DISALLOWED_TOOLS)
      -> Async filter (only safe tools for background agents)
        -> Agent-specific disallow list
          -> Agent-specific allow list
            -> Resolved tool set
```

Each layer further restricts — never expands.

## Prompt Engineering for Agents

### System Prompt Structure

```
1. Identity: "You are a [role] for [product]"
2. Critical constraints: READ-ONLY / NO FILE MODIFICATIONS (for read-only agents)
3. Role-specific strengths: what this agent excels at
4. Operational guidelines: tool usage rules, search strategies
5. Output format: structured report format (Scope, Result, Key Files, Issues)
6. Behavioral rules: what NOT to do (don't editorialize, don't ask questions, etc.)
```

### Prompt Writing Anti-Patterns

| Anti-Pattern | Why It Fails | Fix |
|---|---|---|
| Terse commands ("fix the bug") | Shallow, generic work | Brief like a colleague — explain context, what's been tried |
| "Based on your findings, fix it" | Delegates understanding to agent | Include specific file paths, line numbers, what to change |
| Vague scope | Agent goes off-track | Define what's in, what's out, what another agent handles |
| Repeating background in every prompt | Wastes context | Fork inherits context; fresh agents need full background |

### Fork vs Fresh Agent Prompting

**Fork** (inherits parent context):
- Write a *directive* — what to do, not what the situation is
- Be specific about scope boundaries
- Don't re-explain background the fork already has

**Fresh agent** (starts with zero context):
- Brief like a smart colleague who just walked in
- Explain what you're trying to accomplish and why
- Describe what you've already learned or ruled out
- Give enough context for judgment calls, not just narrow instructions

## Agent Lifecycle Management

### State Machine

```
spawned -> idle -> busy -> idle -> ... -> completed
                             |
                          failed / killed
```

### Key Lifecycle Patterns

1. **Progress tracking**: Accumulate token counts and tool activity per turn
2. **Abort handling**: AbortController per agent; graceful shutdown on kill
3. **Partial result preservation**: On kill, extract last text content from accumulated messages
4. **Completion notification**: Queue notification for orchestrator; include usage metadata
5. **Timeout enforcement**: Heartbeat timer checks task age vs. configured timeout

### Async Agent Lifecycle

```
spawn -> create progress tracker
      -> stream messages (update progress per message)
      -> complete: finalize result -> classify safety -> enqueue notification
      -> abort: kill task -> extract partial result -> enqueue notification
      -> error: fail task -> enqueue error notification
      -> finally: cleanup resources, clear state
```

Mark task completed BEFORE safety classification (which may be slow) so consumers unblock immediately.

## Multi-Agent Coordination

### Architecture Layers

```
Coordinator (orchestration)
  +-- AgentRegistry (tracks active/idle agents)
  +-- TaskBoard (pending/running/completed tasks with dependencies)
  +-- MessageBus (inter-agent communication)
  +-- ResourceManager (shared resources: worktrees, MCP servers)
```

### Task Assignment

1. Task created with priority and optional dependencies
2. Board identifies ready tasks (all dependencies satisfied)
3. Sort ready tasks by priority (higher first), then creation time
4. Find capable idle agents (capability matching)
5. Assign to least-recently-active agent (load balancing)
6. On completion: unblock dependent tasks, try to assign more

### Message Patterns

**Direct message**: Agent A sends to Agent B
```
{ from: "researcher", to: "implementer", content: "Found API in auth.ts:45" }
```

**Broadcast**: Coordinator sends to all agents
```
{ from: "coordinator", to: "*", content: "Critical blocking issue" }
```

**Protocol messages**: Structured shutdown/approval
```
{ type: "shutdown_request", reason: "Task complete" }
{ type: "shutdown_response", request_id: "...", approve: true }
```

### Fork Pattern

Forking creates a child that inherits parent's full conversation context:
- Build messages: keep parent's assistant message + placeholder tool results + child directive
- All forks share prompt cache (identical prefix, only final block differs)
- Guard against recursive forking: detect boilerplate tag in history
- Isolation via git worktrees: child operates in separate working copy

## Safety and Permissions

### Permission Modes

| Mode | Behavior | Use When |
|---|---|---|
| `auto` | Auto-approve safe actions, prompt for risky ones | Default for most agents |
| `plan` | Agent can explore but needs approval to edit | Planning agents |
| `bypass` | Auto-approve everything | Trusted internal agents |
| `bubble` | Surface permission prompts to parent terminal | Fork/child agents |

### Safety Layers (Defense in Depth)

1. **Tool allow/deny lists**: Restrict what agents CAN do
2. **Shell security**: Block dangerous patterns (command substitution, zsh exploits)
3. **Transcript classifier**: Review agent output before handoff to parent
4. **Permission prompts**: User approval for risky operations
5. **Resource limits**: maxTurns, timeouts, memory bounds

### Handoff Safety

When a sub-agent finishes, classify its transcript before the parent acts on results:
- If classifier unavailable: allow with warning
- If flagged: inject security warning into parent's context
- Parent reviews flagged actions before proceeding

## Memory Patterns

### Memory Scopes

| Scope | Lifetime | Storage |
|---|---|---|
| `user` | Cross-project, persistent | `~/.claude/agents/{type}/` |
| `project` | Project-scoped | `.claude/agents/{type}/` |
| `local` | Session-scoped | In-memory only |

### Memory Integration

When memory is enabled:
1. Inject Read/Write/Edit tools (agent needs file access for memory)
2. Append memory prompt to system prompt
3. Initialize from project snapshot if no local memory exists
4. Detect and notify when newer snapshots are available

## Isolation Patterns

### Worktree Isolation
- Agent runs in temporary git worktree (same repo, separate working copy)
- Paths from inherited context refer to parent — translate to worktree root
- Re-read files before editing if parent may have modified them
- Auto-cleanup if no changes; return worktree path and branch if changed

### Process Isolation
- Each agent is a separate process/subprocess
- Communication via message passing (file-based mailboxes or event queues)
- Agent crash doesn't affect coordinator or other agents

## Agent Definition Formats

### Markdown (`.claude/agents/*.md`)

```markdown
---
name: my-agent
description: When to use this agent (primary routing signal)
model: sonnet
tools: [Read, Grep, Glob]
disallowedTools: [Write, Edit]
maxTurns: 50
permissionMode: auto
memory: project
isolation: worktree
background: true
---

System prompt body here...
```

### JSON (settings)

```json
{
  "my-agent": {
    "description": "When to use this agent",
    "prompt": "System prompt...",
    "tools": ["Read", "Grep"],
    "model": "sonnet",
    "maxTurns": 50
  }
}
```

### Agent Priority (resolution order when names collide)

1. Built-in agents (platform defaults)
2. Plugin agents
3. User settings agents
4. Project settings agents
5. Policy/managed agents

Later sources override earlier ones for the same `name`.

## Common Pitfalls

| Pitfall | Symptom | Prevention |
|---|---|---|
| No turn limit | Agent loops forever | Always set `maxTurns` |
| Too many tools | Agent confused by options | Restrict to minimum needed |
| Vague prompts | Shallow, generic output | Provide context, constraints, output format |
| No isolation | Agents conflict on shared files | Use worktrees for parallel writes |
| Missing safety classification | Malicious agent output accepted | Always classify before handoff |
| Recursive delegation | Agent spawns agent spawns agent... | Detect and block recursive patterns |
| No partial result handling | Kill loses all work | Extract last text on abort |

## Reference Files

- **Architecture deep-dive**: See [references/architecture.md](references/architecture.md) for Claude Code's specific implementation patterns with file paths and code structure
- **Prompt templates**: See [references/prompt-templates.md](references/prompt-templates.md) for ready-to-use prompt structures for different agent types
- **Safety patterns**: See [references/safety.md](references/safety.md) for comprehensive security guardrails and classification patterns
