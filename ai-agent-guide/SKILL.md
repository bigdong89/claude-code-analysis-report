---
name: ai-agent-guide
description: |
  Guide for designing, building, and improving AI agents using proven patterns from production systems. Use when:
  - Building a new AI agent (chatbot, autonomous agent, multi-agent, tool-using agent)
  - Defining agent requirements, specifications, or architecture
  - Writing or improving agent prompts, system prompts, or definitions
  - Implementing coordination patterns (delegation, teams, swarms)
  - Designing agent safety, permissions, and guardrails
  - Verifying agent correctness (adversarial testing, quality assurance)
  - Optimizing performance (context management, model selection, cost)
  - Troubleshooting agent issues (hallucination, context loss, loops)
  - Keywords: agent, AI agent, LLM agent, subagent, multi-agent, tool use, function calling, agent architecture, prompt engineering, coordination, swarm, delegation, verification, context optimization
---

# AI Agent Development Guide

Proven patterns from production agent systems, applicable to any AI agent framework.

## Agent Development Lifecycle

```
Requirements → Specification → Architecture → Implementation → Verification → Iteration
     ↑                                                        │
     └────────────── feedback loop ─────────────────────────────┘
```

**Requirements phase**: Identify target users, core use cases, and success metrics before touching any code.

**Specification phase**: Define inputs, outputs, constraints, and edge cases. A precise spec prevents scope creep and ambiguous agent behavior.

**Architecture phase**: Decide on single-agent vs multi-agent, tool selection, permission model, isolation strategy.

**Verification phase**: Adversarial testing — prove the agent *can't* break, not just that it works.

## Requirements & Specification

### Requirements Framework

For each agent, answer:

| Question | Why It Matters |
|---|---|
| Who invokes this agent? | Determines prompt style and trust level |
| What triggers invocation? | Routing signal for orchestration |
| What's the expected output? | Shapes system prompt constraints |
| What should it NEVER do? | Safety guardrails |
| What's the failure mode? | Error handling and recovery strategy |

### Specification Checklist

Every agent spec should include:

1. **Input contract**: What context does the agent receive? (conversation history, files, metadata)
2. **Output contract**: What format/structure must output follow? (free text, structured report, code)
3. **Scope boundary**: What's explicitly in scope vs. out of scope?
4. **Quality bar**: What defines "done"? (tests passing, output validated, review complete)
5. **Dependencies**: What other agents, tools, or services must exist?

### Anti-Pattern: Skipping Specification

| Symptom | Root Cause | Fix |
|---|---|---|
| Agent does unexpected things | No explicit out-of-scope list | Add "MUST NOT" constraints |
| Output format inconsistent | No output contract | Define structured output template |
| Agent conflicts with siblings | No scope boundaries | Document domain ownership |

## Architecture Design

### Single vs Multi-Agent Decision Framework

**Use a single agent when:**
- Task is linear (no parallel sub-problems)
- Full context is needed for all decisions
- Tool set is small and well-defined

**Use multi-agent when:**
- Sub-problems are independent and parallelizable
- Different sub-problems need different tools or model capabilities
- Isolation between tasks is important (e.g., code review vs. implementation)

### Tool Selection Principles

1. **Minimum privilege**: Each agent gets only the tools it needs
2. **Write denial for read-only agents**: Explicitly block write tools to prevent accidental side effects
3. **Wildcard (`tools: ['*']`) only for trusted agents**: Maximum flexibility comes with maximum risk
4. **Tool descriptions are routing signals**: Invest in clarity — the orchestrator uses these to select agents

### Permission Model Design

| Mode | Behavior | Use Case |
|---|---|---|
| Auto | Safe actions approved, risky ones prompt user | Default for most agents |
| Review | Agent explores freely but can't modify without approval | Planning, analysis agents |
| Bypass | All actions auto-approved | Trusted internal agents only |
| Bubble | Permission prompts surface to parent agent | Child agents executing under parent supervision |

### Isolation Strategies

| Strategy | When to Use | Trade-off |
|---|---|---|
| None | Sequential single-agent tasks | Simplest, no overhead |
| Filesystem | Parallel write agents | Prevents file conflicts, moderate setup |
| Process | Untrusted or third-party agents | Strongest isolation, highest overhead |

## Prompt Engineering

### System Prompt Structure

```
1. Identity: "You are a [role] that [primary capability]"
2. Critical constraints: "READ-ONLY" / "DO NOT MODIFY" (first, not last)
3. Strengths: what this agent does exceptionally well
4. Operational rules: tool usage, search strategy, parallel execution
5. Output format: structured template (Scope / Result / Key Files / Issues)
6. Behavioral rules: what NOT to do (don't editorialize, don't ask unnecessary questions)
```

**Key insight**: Place constraints near the top. Models tend to follow instructions more carefully when they appear early in the prompt.

### Context Inheritance vs. Fresh Start

| Situation | Approach | Rationale |
|---|---|---|
| Sub-task needs parent's findings | Context inheritance | Reuse accumulated context; only append child directive |
| Independent analysis needed | Fresh start with background | Provide full context; agent starts clean |
| Parallel sub-tasks from same parent | Context inheritance for all | Identical prefixes maximize prompt cache hits |

### Prompt Anti-Patterns

| Anti-Pattern | Why It Fails | Fix |
|---|---|---|
| "Fix the bug" | Agent guesses scope | Provide file path, line number, expected vs actual behavior |
| "Based on your findings, fix it" | Delegates understanding | Include what was found, what to change, why |
| Vague scope | Agent drifts | Define what's in-scope, out-of-scope, and what another agent handles |
| Repeating context | Wastes tokens | Inherited context agents don't need background repeated |
| "Be thorough" | Unbounded exploration | Define specific checks to perform, not vagueness |

### Prompt Cache Optimization

When spawning parallel sub-tasks that share parent context:
- Keep the parent's full conversation prefix identical across all children
- Only the final instruction block differs per child
- This maximizes prompt cache hits, reducing cost and latency
- Don't change model between parent and children — different models can't share cache

## Context Management

### Response Tiers for Context Pressure

| Pressure Level | Strategy | Trade-off |
|---|---|---|---|
| Low (<70%) | No action needed | Full fidelity |
| Medium (70-85%) | Selective tool result compression | Remove verbose outputs, keep key data |
| High (85-95%) | Micro-compaction of older tool results | Summarize large outputs, preserve recent context |
| Critical (>95%) | Full compaction | Summarize entire conversation history |

### Selective Compression Principles

Only compress tool results that are:
- **Large** (exceeding a size threshold)
- **Re-obtainable** (can be regenerated by re-running the tool)
- **Low information density** (verbose output where key facts can be summarized)

Never compress:
- The most recent messages (they drive current reasoning)
- Tool results that contain unique, hard-to-reproduce data
- Structured outputs that lose meaning when summarized

### Sub-Agent Context Pruning

When spawning sub-agents, omit information the sub-agent doesn't need:
- **Project convention files** (README, style guides) if the parent agent already has them and will interpret the sub-agent's output
- **Irrelevant conversation history** — only pass what's needed for the specific task
- **Other sub-agents' outputs** — prevent context pollution between siblings

This saves tokens proportional to the number of sub-agents spawned, which compounds in multi-agent systems.

## Multi-Agent Coordination

### Coordinator Architecture

Three core components:

```
Agent Registry:   Tracks active/idle agents, capabilities, assignment history
Task Board:       Pending → Ready → Assigned → Running → Completed (with dependencies)
Message Bus:      Direct, broadcast, and protocol messages between agents
```

### Task Assignment Strategy

1. Task created with priority and optional dependencies
2. Identify ready tasks (all dependencies satisfied)
3. Sort by priority, then by creation time
4. Match capable idle agents to ready tasks
5. Assign to least-recently-active agent (load balancing)
6. On completion: unblock dependents, attempt further assignments

### Message Patterns

**Direct**: Point-to-point communication
```
{ from: "researcher", to: "implementer", content: "API schema found in auth module" }
```

**Broadcast**: Coordinator to all agents (use sparingly — expensive)
```
{ from: "coordinator", to: "*", content: "Blocking issue in shared dependency" }
```

**Protocol**: Structured control messages
```
{ type: "shutdown_request", reason: "Task complete, resources freeing" }
{ type: "shutdown_response", request_id: "...", approve: true }
{ type: "plan_approval_request", plan: "..." }
```

### Parallel Execution with Isolation

When multiple agents modify the same resource:
- Each agent works in an isolated copy (e.g., separate branch/directory)
- Changes are merged back by a coordinator after verification
- Conflicts detected at merge time, not at runtime

## Safety & Permissions

### Defense in Depth (4 Layers)

```
Layer 1: Tool Restrictions
  → Global deny list (no agent gets dangerous tools)
  → Source-specific restrictions (custom agents get tighter limits)
  → Background agent restrictions (only safe tools without user visibility)
  → Agent-specific allow/deny lists

Layer 2: Execution Safety
  → Shell command validation (block dangerous patterns before execution)
  → Permission mode gating (auto-approve vs. prompt for approval)
  → Turn limit enforcement (prevent infinite agent loops)

Layer 3: Output Safety
  → Transcript classifier (review agent actions before parent uses results)
  → Security warning injection (flagged results surface to parent agent)

Layer 4: Resource Safety
  → Timeout enforcement (heartbeat kills stalled tasks)
  → Token and memory bounds
  → Isolation boundaries (filesystem, process)
```

Each layer further restricts — never expands. An agent can only be more restricted than the layer above it.

### Tool Filtering Pipeline

```
All available tools
  → Remove globally dangerous tools (destructive, system access)
    → Remove tools inappropriate for agent source (custom agents: tighter)
      → Remove unsafe tools for background execution
        → Apply agent-specific deny list
          → Apply agent-specific allow list (if defined)
            → Resolved tool set for this agent
```

### Handoff Safety

When a sub-agent completes, review its actions before using results:
- **Classifier available and clear**: Accept results
- **Classifier flags concern**: Inject warning into parent's context for review
- **Classifier unavailable**: Accept with caution warning

### Shell Command Safety

**Blocked patterns** (examples):
- Command substitution: `$(...)`, backticks
- Shell exploits: zsh-specific escape sequences
- Glob expansion attacks
- Pipe to eval or exec

**Allowed read-only patterns**:
- Listing: `ls`, `find`, `git status`
- Inspection: `cat`, `head`, `tail`, `grep`, `git log`, `git diff`

## Verification Strategies

### Adversarial Mindset

Verification agents should **try to break** the implementation, not confirm it works:

> "The first 80% is the easy part. Your entire value is in finding the last 20%."

### Type-Specific Verification Matrix

| Change Type | Primary Strategy | Secondary Checks |
|---|---|---|
| Frontend | Navigate and interact via browser automation; check subresource loading | Accessibility, console errors, responsive |
| Backend/API | Hit endpoints directly; verify response shapes (not just status codes) | Error handling, edge cases, auth |
| CLI/Scripts | Run with representative inputs; check stdout/stderr/exit codes | Edge inputs (empty, malformed, boundary) |
| Infrastructure | Validate syntax; dry-run where possible | Check env vars are referenced, not just defined |
| Library/Package | Build and test; import from fresh context; exercise public API | Type exports match docs |
| Bug Fix | Reproduce original bug → verify fix → regression tests | Related functionality for side effects |
| Refactoring | Existing tests pass unchanged; public API surface unchanged | Observable behavior identical (same in → same out) |
| Data Pipeline | Run with sample input; verify output shape/schema | Empty input, single row, NaN/null handling |

### Required Baseline Checks (Universal)

1. Build compiles without errors
2. Test suite passes
3. Linters/type-checkers pass (if configured)
4. No regressions in related code

### Anti-Rationalization Patterns

These are excuses agents (and humans) reach for when tempted to skip verification:

| Excuse | Reality |
|---|---|
| "The code looks correct" | Reading code is not verification — run it |
| "The existing tests already pass" | Tests may only cover happy paths — verify independently |
| "This is probably fine" | "Probably" is not verified — run it |
| "I don't have the right tools" | Check for available tools before assuming |
| "This would take too long" | Verification scope is not the agent's call |

### Structured Verification Report

Every check must follow this format:

```
### Check: [what you're verifying]
**Command:** [exact command executed]
**Output:** [actual terminal output]
**Result:** PASS (or FAIL — with expected vs actual)
```

## Model Selection

### Decision Framework

| Task Complexity | Recommended Model | Rationale |
|---|---|---|
| Search/exploration (read-only) | Lightweight (fast, cheap) | No generation needed, pattern matching suffices |
| Planning/architecture | Same tier as main agent | Needs same context understanding to plan effectively |
| Implementation (creative work) | Full capability (most capable) | Complex reasoning and code generation required |
| Verification (adversarial) | Same tier or higher | Needs to find edge cases the implementer missed |

### Cost Optimization Principles

- **Read-only agents = cheap models**: Exploration and search don't need frontier capabilities
- **Avoid model mixing in parallel forks**: Different models break prompt cache sharing
- **Omit unnecessary context**: Project convention files in sub-agents cost tokens with no benefit
- **Compress before expanding**: Clean up context before spawning sub-agents

## Lifecycle Management

### State Machine

```
spawned → idle → busy → idle → ... → completed
                           │
                        failed / killed / timed_out
```

### Key Patterns

1. **Progress tracking**: Accumulate token counts and tool activity per turn
2. **Abort handling**: Cancellation controller per agent; graceful shutdown on kill
3. **Partial result preservation**: On kill, extract last text output from accumulated messages
4. **Completion notification**: Queue notification for orchestrator with usage metadata
5. **Timeout enforcement**: Heartbeat timer checks task age vs. configured timeout
6. **Mark complete before post-processing**: Finalize result before slow operations (safety classification, worktree result) so consumers unblock immediately

## Memory & Persistence

### Memory Scopes

| Scope | Lifetime | Use Case |
|---|---|---|
| User-level | Cross-project, persistent | User preferences, learned patterns |
| Project-level | Project-scoped | Project conventions, architecture decisions |
| Session-level | Session-scoped only | Temporary state, intermediate results |

### Memory Integration

When memory is enabled:
1. Inject read/write tools (agent needs file access for memory)
2. Append memory management instructions to system prompt
3. Initialize from snapshot if no local memory exists
4. Detect and notify when newer snapshots are available

## Common Pitfalls

| Pitfall | Symptom | Prevention |
|---|---|---|
| No turn limit | Agent loops indefinitely | Always set maximum turns |
| Too many tools | Agent confused by choices | Restrict to minimum needed |
| Vague prompts | Shallow, generic output | Provide context, constraints, output format |
| No isolation | Parallel agents conflict on shared resources | Use filesystem/process isolation |
| Missing output review | Malicious agent output accepted | Always review sub-agent actions before use |
| Recursive delegation | Agent spawns agent spawns agent... | Detect and block recursive patterns |
| No partial results | Kill loses all work | Extract last output on abort |
| All agents same model | Wastes money on simple tasks | Match model to task complexity |

## Reference Files

- **Architecture patterns**: [references/architecture.md](references/architecture.md) — agent type hierarchy, tool filtering, lifecycle, coordination, context inheritance
- **Prompt templates**: [references/prompt-templates.md](references/prompt-templates.md) — ready-to-adapt prompt structures for 6 agent types
- **Safety patterns**: [references/safety.md](references/safety.md) — defense in depth, tool filtering, permissions, handoff safety, shell security
- **Verification patterns**: [references/verification-patterns.md](references/verification-patterns.md) — adversarial methodology, strategy matrix, anti-rationalization
- **Context optimization**: [references/context-optimization.md](references/context-optimization.md) — compression tiers, cache optimization, sub-agent pruning
