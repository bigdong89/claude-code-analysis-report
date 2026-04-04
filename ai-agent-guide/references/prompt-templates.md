# Agent Prompt Templates

Generalized prompt structures for common agent types. Each template describes the design intent, key constraints, and output format — adaptable to any agent framework.

## Table of Contents

1. [Read-Only Explorer](#template-1-read-only-explorer)
2. [General-Purpose Executor](#template-2-general-purpose-executor)
3. [Planning/Architecture Agent](#template-3-planningarchitecture-agent)
4. [Parallel Worker (Context-Inheriting)](#template-4-parallel-worker-context-inheriting)
5. [Team Collaborator](#template-5-team-collaborator)
6. [Adversarial Verifier](#template-6-adversarial-verifier)
7. [Prompt Construction Guidelines](#prompt-construction-guidelines)

---

## Template 1: Read-Only Explorer

**Use for**: Codebase search, file discovery, architecture analysis, investigation.

### Design Intent
An explorer agent must be incapable of modifying anything. The prompt enforces this through an explicit prohibition list (not just a vague "don't edit") and restricts shell access to read-only commands.

### Structure

```
1. Identity: "[Role] that [primary capability]"
2. Critical constraint (READ-ONLY) — placed FIRST, before strengths
3. Explicit prohibition list:
   - No file creation, modification, deletion, or movement
   - No temporary files (including /tmp)
   - No shell redirects or heredocs
   - No commands that change system state
4. Strengths: what the agent does well (search, analysis, pattern matching)
5. Tool usage guidelines:
   - Pattern matching for broad file search
   - Content search with regex
   - Direct file reads for known paths
   - Shell whitelisted for: listing, inspection, version control reads
   - Shell denied for: creation, modification, installation, commits
6. Parallelism guidance: batch independent searches
```

### Key Design Principles
- **Explicit prohibition beats vague instruction**: "Don't modify files" is weaker than an enumerated list of forbidden operations
- **Shell whitelisting**: Instead of saying "be careful with shell," list exactly which commands are allowed
- **Parallelism encouragement**: Read-only agents benefit most from parallel tool calls since there are no write conflicts
- **Flexible output format**: Don't over-constrain reporting — let the agent adapt its output to the query

---

## Template 2: General-Purpose Executor

**Use for**: Multi-step research, code changes, implementation, any task.

### Design Intent
A general-purpose agent needs flexibility but with guardrails against over-engineering and file bloat. The prompt balances "complete the task" with "don't gold-plate it."

### Structure

```
1. Identity: Agent capable of using available tools to complete tasks
2. Completion framing: "Complete fully — don't over-engineer, but don't leave it half-done"
3. Output expectation: concise report (parent agent relays to user)
4. Strengths: search, multi-file analysis, investigation, multi-step tasks
5. File guidelines:
   - Prefer editing existing files over creating new ones
   - Never create documentation files unless explicitly requested
6. Search strategy: broad initial search, then narrow down
7. Thoroughness: check multiple locations, naming conventions, related files
```

### Key Design Principles
- **Gold-plate vs. half-done framing**: Two-sided constraint prevents both extremes
- **Parent relay awareness**: Agent knows its output goes through a parent, so it should be concise and factual
- **Documentation prohibition**: Prevents the most common form of file bloat (auto-generated READMEs, docs)
- **Edit over create**: Reinforces minimal-impact changes

---

## Template 3: Planning/Architecture Agent

**Use for**: Implementation planning, architecture review, design decisions.

### Design Intent
A planning agent explores thoroughly but must never modify. It follows a structured methodology and produces a concrete plan with specific file references.

### Structure

```
1. Identity: Software architect / planning specialist
2. Critical constraint: READ-ONLY (same prohibition list as explorer)
3. Input: requirements + optional analytical perspective
4. Process (numbered steps):
   a. Understand requirements
   b. Explore thoroughly (existing patterns, architecture, references, code paths)
   c. Design solution (trade-offs, architectural decisions, existing patterns)
   d. Detail the plan (step-by-step, dependencies, challenges)
5. Required output section:
   - "Critical Files for Implementation" — list 3-5 specific files
   - Must include file paths for traceability
6. Final reminder: "You can ONLY explore and plan"
```

### Key Design Principles
- **Structured output forces specificity**: Requiring file references prevents vague plans
- **Perspective parameter**: Enables the same agent to analyze from different angles (security, performance, maintainability)
- **Numbered process**: Provides consistent methodology the agent can follow
- **Dual reminder**: Both the initial constraint and the final line reinforce read-only behavior

---

## Template 4: Parallel Worker (Context-Inheriting)

**Use for**: Parallel work that inherits parent context, isolated execution.

### Design Intent
A worker spawned for parallel execution has the parent's full context but a narrow directive. The prompt must prevent recursion, scope creep, and unnecessary communication.

### Structure

```
1. Opening: STOP signal — demands attention before anything else
2. Identity: "You are a worker process, NOT the main agent"
3. Non-negotiable rules:
   a. Anti-recursion: do NOT spawn sub-agents (you ARE the sub-agent)
   b. No conversation: don't ask questions or suggest next steps
   c. No editorializing: no meta-commentary
   d. Execute directly: use tools, don't plan to use tools
   e. Commit before reporting (if modifying files)
   f. Silent execution: no text between tool calls
   g. Scope boundary: stay within directive, one sentence max for out-of-scope
   h. Word limit: concise report (e.g., 500 words max)
   i. Start with scope echo: "Scope: <one-sentence restatement>"
   j. Structured output format
4. Output format:
   - Scope: echo back the assigned scope
   - Result: key findings or answer
   - Key files: relevant paths
   - Files changed: list with version control reference
   - Issues: problems found
```

### Key Design Principles
- **Anti-recursion guard as rule #1**: The most critical rule — without it, workers spawn workers endlessly
- **Silent execution**: Workers should not emit text between tool calls to avoid context bloat
- **Structured output**: Machine-parseable format enables the parent to aggregate results programmatically
- **Scope echo**: Forces the worker to acknowledge and restate its assignment, catching misunderstandings early
- **No preamble**: "Scope:" as the first word eliminates conversational filler

---

## Template 5: Team Collaborator

**Use for**: Agents that coordinate via shared task lists in a multi-agent team.

### Design Intent
A teammate agent operates within a coordination framework, claiming tasks from a shared board and communicating results through a messaging system.

### Structure

```
1. Identity: Teammate in a multi-agent team
2. Communication rules:
   - Report to team lead on task completion
   - Check task list after completing each task
   - Claim unassigned, unblocked tasks (set owner)
   - Prefer tasks in ID order (lower IDs often set up context for higher ones)
   - If all tasks blocked: notify team lead, wait
   - When blocked: help resolve blocking tasks
3. Team discovery:
   - Read team configuration to find members
   - Address teammates by name (not by ID)
   - Use direct messaging for coordination
4. Work rhythm:
   - Complete task → mark complete → check for next → repeat
   - If no available work: go idle (normal state, not an error)
```

### Key Design Principles
- **Name-based addressing**: Human-readable names reduce errors and improve debuggability
- **ID-order preference**: Earlier tasks often set up context or dependencies for later tasks
- **Explicit blocking protocol**: Clear escalation path when stuck
- **Idle is normal**: Explicitly stating that idle is not an error prevents unnecessary activity

---

## Template 6: Adversarial Verifier

**Use for**: Testing, code review, security audit, quality assurance.

### Design Intent
A verifier agent tries to break things, not confirm they work. Its mindset must be adversarial — finding flaws is the goal, not confirming correctness.

### Structure

```
1. Identity: Adversarial verifier / quality assurance specialist
2. Mindset: "Your value is in finding the last 20%, not confirming the first 80%"
3. Verification methodology:
   a. Understand what changed and what the expected behavior is
   b. Attempt to break it (edge cases, invalid inputs, race conditions)
   c. Verify the fix actually addresses the root cause (not just symptoms)
   d. Check for regressions in related functionality
4. Required output format per check:
   - Check name: what's being verified
   - Command: exact command or action executed
   - Output: actual result
   - Result: PASS or FAIL (with expected vs. actual on failure)
5. Anti-rationalization rules:
   - "The code looks correct" → reading is not verification, run it
   - "Tests already pass" → tests may only cover happy paths
   - "This is probably fine" → "probably" is not verified
   - "I don't have the right tools" → check before assuming
6. Behavioral rules:
   - Never skip a check because it "seems fine"
   - Report ALL findings, not just failures
   - Prioritize: build → tests → lint → functionality → edge cases
```

### Key Design Principles
- **Adversarial framing**: The agent's identity is "breaker," not "confirmer"
- **Structured check format**: Forces rigor — every check must have command, output, and result
- **Anti-rationalization list**: Preempts the most common excuses for skipping verification
- **Prioritized verification order**: Fast checks first (build/lint) before slow ones (edge cases)

---

## Prompt Construction Guidelines

### When Delegating to Sub-Agents

| Aspect | Good Directive | Poor Directive |
|--------|---------------|----------------|
| Specificity | "Review migration file X for safety. Context: adding NOT NULL to 50M rows with backfill. I checked locking — verify concurrent write safety. Report: safe? What breaks?" | "Review the migration file for safety issues." |
| Background | Include what's already been checked so the agent doesn't redo it | Omit context, forcing redundant investigation |
| Output format | "Report: is this safe? If not, what specifically breaks?" | "Let me know what you think." |
| Scope | Define exact files, systems, and boundaries | "Look into it" |

### When Directing Parallel Workers (Context-Inheriting)

| Aspect | Good Directive | Poor Directive |
|--------|---------------|----------------|
| Checks | "Check: uncommitted changes, commits ahead of main, test existence, feature gate wiring, CI config changes. Report: punch list (done vs. missing). Under 200 words." | "Check if the branch is ready to ship." |
| Word limit | Explicit constraint prevents bloated reports | No limit → verbose output wastes tokens |
| Format | "Punch list: done vs. missing" | "Tell me what you find" |
| Background | Omit — worker already has parent context | Repeat background the worker already knows |

### General Principles

1. **Constraints near the top**: Models follow instructions more carefully when they appear early in the prompt
2. **Be specific, not vague**: "Fix the bug" → agent guesses scope. "Fix null pointer in auth handler at line 42" → agent knows exactly what to do
3. **Include what's already done**: Prevents sub-agents from redoing work the parent already completed
4. **Define output format**: "Structured report with Scope/Result/Files/Issues" > "Tell me what you find"
5. **Set word limits**: Prevents token waste on verbose reports
6. **Don't repeat inherited context**: If the worker inherits parent context, don't include the same background in the directive
