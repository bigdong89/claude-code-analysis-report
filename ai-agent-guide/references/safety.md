# Agent Safety Patterns

Comprehensive security guardrails for AI agent systems, organized as a defense-in-depth model.

## Table of Contents

1. [Defense in Depth Overview](#defense-in-depth-overview)
2. [Tool Restriction Layers](#tool-restriction-layers)
3. [Permission Modes](#permission-modes)
4. [Shell Command Safety](#shell-command-safety)
5. [Output Safety](#output-safety)
6. [Anti-Recursion Guards](#anti-recursion-guards)
7. [Resource Safety](#resource-safety)
8. [Isolation Strategies](#isolation-strategies)
9. [Safe Agent Configuration Checklist](#safe-agent-configuration-checklist)

## Defense in Depth Overview

Agent security operates on four independent layers. Each layer further restricts the agent — never expands. A tool blocked at layer 1 cannot be re-enabled by layer 4.

```
Layer 1: Tool Restriction
  → Global deny list (dangerous tools removed for ALL agents)
  → Source-specific restrictions (custom agents get tighter limits)
  → Background/async restrictions (only safe tools without user visibility)
  → Agent-specific allow/deny lists (per-agent customization)

Layer 2: Execution Safety
  → Shell command validation (block dangerous patterns before execution)
  → Permission mode gating (auto-approve vs. prompt for approval)
  → Turn limit enforcement (prevent infinite loops)

Layer 3: Output Safety
  → Transcript review (inspect agent actions before parent uses results)
  → Security warning injection (flagged results surface to parent)

Layer 4: Resource Safety
  → Timeout enforcement (heartbeat kills stalled tasks)
  → Memory/resource bounds (context window, token limits)
  → Isolation boundaries (filesystem, process)
```

### The Never-Expand Principle

Each layer can only make the agent **more** restricted. If a tool is denied at layer 1, no subsequent layer can re-enable it. This prevents misconfiguration from creating security gaps.

## Tool Restriction Layers

### Layer 1: Global Deny List

Tools that NO agent can access, regardless of type or trust level. These are tools that are dangerous in any context:

- **Session management tools**: Tools that can modify the agent's own session or configuration
- **System-level operations**: Tools that affect shared state beyond the agent's scope
- **Credential access tools**: Tools that can read secrets, tokens, or API keys
- **Agent-spawning tools for custom agents**: Prevents recursive delegation

### Layer 2: Source-Specific Restrictions

Additional restrictions applied based on where the agent definition comes from:

- **Built-in agents**: Trust the platform's tool assignments
- **Plugin agents**: Remove tools that could affect other plugins or the host system
- **User-defined agents**: Tighter restrictions — remove agent-spawning tools, system access tools
- **Project-defined agents**: Same as user-defined, scoped to project context

**Rationale**: User and project-defined agents are inherently less trusted than built-in agents because they haven't been vetted by the platform team. They should have a smaller tool set.

### Layer 3: Background/Async Filter

For agents running asynchronously (without real-time user visibility):

- Only allow tools safe for unattended execution
- Block all file-write tools (modifications without user oversight)
- Block dangerous shell commands
- Block network access to untrusted endpoints

**Rationale**: A background agent modifying files without the user seeing it in real time creates a trust gap. If the user can't observe and intervene, the agent must be more restricted.

### Layer 4: Agent-Specific Configuration

Per-agent allow/deny lists defined in the agent's configuration:

- **Allow list**: If defined, ONLY these tools are available (whitelist mode). Most restrictive.
- **Deny list**: Additional tools to remove from whatever passed previous layers.
- **No explicit list**: All tools passing previous layers are available. Least restrictive.

### Resolution Order

```
All tools available in the platform
  → Remove global deny list
    → Remove source-specific restrictions
      → Remove background/async filter (if applicable)
        → Remove agent-specific deny list
          → If agent-specific allow list exists: KEEP ONLY those
            → Final tool set for this agent
```

## Permission Modes

### Auto Mode (Default)

- Safe operations are auto-approved
- Risky operations prompt the user for approval
- A safety classifier can override with warnings

**Use for**: Most agents, especially those interacting with users directly.

### Review Mode

- Agent can explore freely (read files, search, analyze)
- Cannot modify anything without explicit approval
- All write operations require user confirmation

**Use for**: Planning agents, analysis agents, and any agent where the user wants to see what the agent proposes before it acts.

### Bypass Mode

- All operations are auto-approved with no user prompts
- Maximum autonomy, maximum risk

**Use for**: ONLY for trusted internal agents in controlled environments. Never expose to user-defined agents.

### Bubble Mode

- Permission prompts surface to the parent agent or terminal
- The parent agent is responsible for approval decisions

**Use for**: Child agents executing under parent supervision. The parent has already established trust with the user, so it can make informed decisions about the child's actions.

## Shell Command Safety

### Dangerous Patterns to Block

These patterns should be blocked before shell commands reach execution:

| Pattern | Why Dangerous |
|---------|--------------|
| Command substitution `$(...)` | Executes arbitrary commands |
| Backtick substitution `` `...` `` | Same as above, different syntax |
| Pipe to eval `| eval` | Executes arbitrary code |
| Pipe to exec `| exec` | Replaces current process |
| Shell-specific exploits | Zsh/bash-specific escape sequences |
| Glob expansion attacks | `**/*` patterns causing resource exhaustion |
| Redirect to sensitive paths | Writing to `/etc/`, system directories |

### Safe Read-Only Patterns

These patterns are generally safe for agents:

| Pattern | Use Case |
|---------|----------|
| `ls`, `find`, `tree` | Directory listing |
| `git status`, `git log`, `git diff` | Version control inspection |
| `cat`, `head`, `tail` | File content inspection |
| `grep`, `rg` | Content search |
| `wc`, `stat` | File metadata |
| `echo` (without redirects) | Debugging output |

### Validation Approach

1. Parse command structure before execution
2. Check against denylist of dangerous patterns
3. Block or sanitize flagged commands
4. Log blocked attempts for auditing
5. Rate-limit shell access to prevent brute-force attempts

## Output Safety

### Transcript Review

When a sub-agent completes, its actions should be reviewed before the parent agent uses the results:

```
Sub-agent completes
  → Build transcript from all agent messages
  → Run transcript classifier (if available)
  → Decision: allowed | blocked | unavailable
  → If blocked: inject security warning into parent context
  → If unavailable: allow with caution warning
  → Parent reviews flagged actions before proceeding
```

### What to Check

- Whether the agent performed dangerous or unexpected operations
- Whether the output contains security-sensitive content (credentials, tokens, PII)
- Whether the agent's actions violate safety policy
- Whether the agent attempted to access resources outside its scope

### Security Warning Format

When a sub-agent's actions are flagged:

```
SECURITY WARNING: This sub-agent performed actions that may violate safety policy.
Reason: <specific reason from classifier>.
Review the sub-agent's actions carefully before acting on its output.
```

The parent agent should then review the specific actions before using the sub-agent's results.

## Anti-Recursion Guards

### The Problem

Without guards, agents can spawn sub-agents that spawn sub-agents, creating infinite delegation chains that consume tokens and resources without making progress.

### Detection Methods

1. **Conversation history marker**: Inject a special marker into the child's context that identifies it as a child. If the child detects this marker, it must not spawn further children.
2. **Depth counter**: Track the nesting depth and block spawning beyond a threshold.
3. **Tool-level blocking**: Remove the agent-spawning tool from sub-agents (most effective, prevents the problem at the source).

### Mutual Exclusivity

Some modes are mutually exclusive and must be enforced:

- **Delegation mode vs. coordination mode**: An agent cannot be both a delegate (child) and a coordinator (parent) simultaneously
- **Interactive mode vs. non-interactive**: Some features only make sense when the agent can interact with a user

## Resource Safety

### Timeout Enforcement

A heartbeat system should monitor agent health:

```
Every N seconds (configurable):
  1. Check for idle agents exceeding idle timeout
  2. Terminate timed-out agents
  3. Check running tasks exceeding task timeout
  4. Fail timed-out tasks (available for reassignment if configured)
  5. Clean up old completed tasks
  6. Clean up old messages
```

### Turn Limit Enforcement

Every agent must have a maximum turn count. When reached:

1. Stop the agentic loop immediately
2. Generate a summary of what was accomplished
3. Return the partial result to the parent
4. Log the limit event for observability

### Partial Result Preservation

When an agent is killed (abort or timeout):

1. Scan accumulated messages in reverse order
2. Extract the last text content from an assistant message
3. Return this partial result to the parent
4. The parent decides: retry, reassign, or accept partial result

This prevents total work loss on termination.

### Memory Bounds

- **Context window limit**: When the context exceeds a threshold, trigger compression (see [context-optimization.md](context-optimization.md))
- **Token budget**: Set per-agent and per-task token limits to control cost
- **Output size limit**: Cap the size of individual tool results to prevent a single tool call from consuming the entire context

## Isolation Strategies

### Filesystem Isolation

Each agent works in an isolated copy of the file system:

- Changes don't affect the parent or other agents
- Relative paths work identically (same directory structure)
- Auto-cleanup when no changes were made
- Each agent works on its own branch or copy

### Process Isolation

For untrusted or third-party agents:

- Separate process with restricted permissions
- Limited network access
- No access to host credentials or secrets
- Resource quotas (memory, CPU, file handles)

### Path Translation

When an agent inherits context from a parent operating in a different directory:

```
Parent context says: auth/handler.ts (relative to parent's working directory)
Agent must translate: same relative path from its own working directory root
```

### Stale Data Prevention

When context is inherited across isolation boundaries, inject a notice:

```
You've inherited conversation context from a parent operating at [parent directory].
You are in an isolated environment at [your directory].
Paths in inherited context refer to the parent's directory.
Re-read files before editing if the parent may have modified them.
```

## Safe Agent Configuration Checklist

Every agent configuration should be checked against this list:

- [ ] **maxTurns is set** — prevents infinite loops
- [ ] **Tools are restricted to minimum needed** — principle of least privilege
- [ ] **Write tools are denied for read-only agents** — explicit blocking, not just omission
- [ ] **Agent-spawning tools are denied for sub-agents** — prevents recursive delegation
- [ ] **Permission mode matches agent's trust level** — auto for most, bypass only for trusted internal agents
- [ ] **Isolation enabled for parallel write agents** — prevents file conflicts
- [ ] **Memory scope is appropriate** — don't persist sensitive data in project scope
- [ ] **Project conventions omitted when not needed** — saves tokens for sub-agents
- [ ] **Initial prompt contains no secrets** — credentials, API keys, tokens
- [ ] **Timeout is configured** — prevents runaway resource consumption
- [ ] **Output review is enabled** — for agents whose results are used by other agents
