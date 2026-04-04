# Agent Safety Patterns

Comprehensive security guardrails extracted from Claude Code's agent system.

## Defense in Depth Layers

```
Layer 1: Tool Restriction
  └─ Global disallow list (dangerous tools removed for ALL agents)
  └─ Source-specific restrictions (custom agents get tighter limits)
  └─ Async filtering (background agents only get safe tools)
  └─ Agent-specific allow/deny lists (per-agent customization)

Layer 2: Execution Safety
  └─ Shell command validation (block dangerous patterns)
  └─ Permission mode gating (auto/plan/bypass per operation)
  └─ maxTurns enforcement (prevent infinite loops)

Layer 3: Output Safety
  └─ Transcript classifier (review agent actions before parent uses results)
  └─ Security warning injection (flagged results surface to parent)

Layer 4: Resource Safety
  └─ Timeout enforcement (heartbeat kills stalled tasks)
  └─ Memory/resource bounds (context window, token limits)
  └─ Worktree isolation (filesystem changes don't affect parent)
```

## Tool Disallow Lists

### Global Disallow (ALL_AGENT_DISALLOWED_TOOLS)
Tools that NO agent can access — dangerous regardless of agent type.

### Custom Agent Disallow (CUSTOM_AGENT_DISALLOWED_TOOLS)
Additional restrictions for user/project-defined agents (not built-in).

### Async Agent Allow (ASYNC_AGENT_ALLOWED_TOOLS)
Only tools safe for background execution — prevents background agents from running dangerous operations without user visibility.

### In-Process Teammate Allow (IN_PROCESS_TEAMMATE_ALLOWED_TOOLS)
Additional tools for teammates running in the coordinator process (e.g., AgentTool for spawning sync subagents, TaskCreate for coordination).

## Permission Modes

### Auto Mode (Default)
- Safe operations auto-approved
- Risky operations prompt the user
- Classifier can override with warnings

### Plan Mode
- Agent can explore codebase freely
- Cannot edit files without approval
- Used for: planning agents that need context but shouldn't modify

### Bypass Mode
- All operations auto-approved
- Only for trusted internal agents
- Never expose to user-defined agents

### Bubble Mode
- Permission prompts surface to parent terminal
- Used for fork/child agents
- Parent agent is responsible for approval decisions

## Transcript Classification

When a sub-agent completes, its transcript is classified before the parent acts on results.

### Classification Flow

```
Sub-agent completes
  → Build transcript from agent messages
  → Run transcript classifier (if feature enabled and mode is auto)
  → Decision: allowed | blocked | unavailable
  → If blocked: inject security warning into parent context
  → If unavailable: allow with warning
  → Parent reviews flagged actions before proceeding
```

### What the Classifier Checks
- Whether agent performed dangerous operations
- Whether output contains security-sensitive content
- Whether actions violate safety policy
- Returns structured reason when blocking

### Safety Warning Format
```
SECURITY WARNING: This sub-agent performed actions that may violate security policy.
Reason: <specific reason from classifier>.
Review the sub-agent's actions carefully before acting on its output.
```

## Shell Security Patterns

### Dangerous Pattern Blocking
```
Blocked:
- Command substitution: $(...), `...`
- Zsh exploits
- Glob expansion attacks
- Pipe to eval

Allowed (read-only):
- ls, git status, git log, git diff
- find, grep, cat, head, tail
- File listing and inspection
```

### Command Validation Approach
1. Parse command structure before execution
2. Check against denylist of dangerous patterns
3. Block or sanitize flagged commands
4. Log blocked attempts for auditing

## Fork Safety Guards

### Anti-Recursion

```typescript
// Detect if running inside a fork child
function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    return m.message.content.some(
      block => block.type === 'text' &&
      block.text.includes('<fork-boilerplate>')
    )
  })
}

// Fork children inherit parent's tool pool for cache sharing,
// but AgentTool calls are rejected at runtime
```

### Mutual Exclusivity
- Fork mode and coordinator mode are mutually exclusive
- Fork mode disabled in non-interactive sessions
- Coordinator already owns orchestration role

## Task Timeout and Resource Management

### Heartbeat System
```
Every 30 seconds (configurable):
  1. Check for idle agents exceeding idle timeout (5 min default)
  2. Terminate timed-out agents
  3. Check running tasks exceeding task timeout
  4. Fail timed-out tasks (available for reassignment if autoReassign enabled)
  5. Clean up old completed tasks (older than 1 hour)
  6. Clean up old messages (older than 1 hour)
```

### Abort Handling
```
On abort (user kill or system termination):
  1. Transition task status to 'killed'
  2. Extract partial result (last text content from messages)
  3. Clean up worktree if applicable
  4. Enqueue notification with partial result
  5. Clear agent-specific state (skills, dump state)
```

## Worktree Security

### Isolation Guarantees
- Separate git working copy — changes don't affect parent
- Same repository structure — relative paths work identically
- Auto-cleanup on no changes — no orphan worktrees
- Branch isolation — each agent works on its own branch

### Path Translation
When agent inherits context from parent operating in different directory:
```
Parent context says: src/auth/handler.ts (relative to parent's cwd)
Agent must translate: same relative path from worktree root
```

### Stale Data Prevention
```
Worktree notice injected into fork child:
"You've inherited conversation context from parent at {parentCwd}.
You are in an isolated worktree at {worktreeCwd}.
Paths in inherited context refer to parent's directory.
Re-read files before editing if parent may have modified them."
```

## Safe Agent Configuration Checklist

- [ ] `maxTurns` is set (prevents infinite loops)
- [ ] Tools are restricted to minimum needed (principle of least privilege)
- [ ] Write tools are denied for read-only agents
- [ ] `disallowedTools` includes AgentTool for sub-agents (prevents recursive delegation)
- [ ] Permission mode matches agent's trust level
- [ ] Worktree isolation enabled for parallel write agents
- [ ] Memory scope is appropriate (don't persist sensitive data in `project` scope)
- [ ] `omitClaudeMd: true` for agents that don't need full project conventions
- [ ] Initial prompt doesn't contain secrets or credentials
