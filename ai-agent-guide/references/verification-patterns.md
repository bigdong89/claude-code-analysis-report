# Verification Patterns

Adversarial methodology for proving agents and code work correctly, extracted from production verification systems.

## Table of Contents

1. [Adversarial Mindset](#adversarial-mindset)
2. [Type-Specific Verification Matrix](#type-specific-verification-matrix)
3. [Required Baseline Checks](#required-baseline-checks)
4. [Adversarial Probes](#adversarial-probes)
5. [Anti-Rationalization Patterns](#anti-rationalization-patterns)
6. [Structured Verification Report](#structured-verification-report)

## Adversarial Mindset

Verification agents must **try to break** the implementation, not confirm it works.

> "The first 80% is the easy part. Your entire value is in finding the last 20%."

### Why Adversarial Verification Matters

| Approach | Result | Coverage |
|----------|--------|----------|
| Confirm it works | False confidence | Happy paths only |
| Try to break it | Genuine confidence | Edge cases, failure modes |
| Assume it's broken | Highest confidence | Root cause analysis |

### The Verification Hierarchy

```
Level 1: Does it build? (compilation, dependency resolution)
Level 2: Do existing tests pass? (regression check)
Level 3: Does the change work as intended? (functional verification)
Level 4: Does it break anything else? (regression detection)
Level 5: Can a user break it? (adversarial probing)
Level 6: Does it handle edge cases? (boundary testing)
```

Each level catches different classes of bugs. Skipping levels creates blind spots.

## Type-Specific Verification Matrix

### Frontend Changes

| Check | Method | What to Look For |
|-------|--------|------------------|
| Visual rendering | Browser automation | Layout shifts, missing content, broken styles |
| Interaction | Click, type, navigate flows | Event handlers fire correctly, state updates |
| Subresource loading | Network request monitoring | Missing assets, failed API calls, 404s |
| Console errors | Browser console capture | JavaScript errors, warnings, deprecations |
| Responsive design | Viewport resize testing | Breakpoints, overflow, hidden content |
| Accessibility | Automated a11y audit | Missing labels, keyboard navigation, contrast |

### Backend / API Changes

| Check | Method | What to Look For |
|-------|--------|------------------|
| Response shape | Direct endpoint calls | Correct status codes, expected field structure |
| Error handling | Invalid input requests | Appropriate error messages, no stack traces leaked |
| Edge cases | Boundary value inputs | Empty inputs, oversized inputs, special characters |
| Authentication | Authenticated/unauthenticated requests | Proper access control, token validation |
| Performance | Load or timing tests | Response time within acceptable bounds |

### CLI / Scripts

| Check | Method | What to Look For |
|-------|--------|------------------|
| stdout/stderr | Run with representative inputs | Correct output format, error messages to stderr |
| Exit codes | Various input scenarios | 0 for success, non-zero for failure |
| Edge inputs | Empty, malformed, boundary values | Graceful handling, not crashes |
| Help output | `--help` or no arguments | Usage documentation present |

### Infrastructure / Configuration

| Check | Method | What to Look For |
|-------|--------|------------------|
| Syntax | Linting/validation tools | No syntax errors in config files |
| Dry run | Preview mode where available | Changes match expectations |
| Environment | Check variable references | Variables are used, not just defined |
| Secrets | Scan for committed secrets | No API keys, passwords, tokens in code |

### Library / Package

| Check | Method | What to Look For |
|-------|--------|------------------|
| Build | Compile/build from clean state | No build errors |
| Tests | Full test suite | All tests pass |
| Imports | Fresh context import | No missing dependencies |
| Public API | Exercise all public methods | Documented behavior matches actual |
| Type exports | Type checking | Exported types match documentation |

### Bug Fix

| Check | Method | What to Look For |
|-------|--------|------------------|
| Reproduction | Reproduce original bug | Bug exists before fix, gone after |
| Fix verification | Test the specific fix | Root cause addressed, not just symptoms |
| Regression | Run related functionality | No side effects from the fix |
| Similar bugs | Search for same pattern | Other instances of the same bug class |

### Refactoring

| Check | Method | What to Look For |
|-------|--------|------------------|
| Existing tests | Run unchanged test suite | All tests pass without modification |
| Public API | Compare before/after signatures | No breaking changes to public interface |
| Observable behavior | Same input → same output | Behavioral equivalence maintained |
| Internal structure | Code review | Simpler/cleaner without behavior change |

### Data Pipeline

| Check | Method | What to Look For |
|-------|--------|------------------|
| Sample input | Run with representative data | Correct output shape and values |
| Schema validation | Verify output structure | Matches expected schema |
| Empty input | Run with no data | Graceful handling, no crashes |
| Edge values | Null, NaN, extreme values | Proper handling, not propagation |

## Required Baseline Checks

Every verification must include these universal checks:

1. **Build compiles** without errors
2. **Test suite passes** (existing tests, not just new ones)
3. **Linters/type-checkers pass** (if configured)
4. **No regressions** in related functionality

Skipping any of these creates risk. There are no exceptions.

## Adversarial Probes

After baseline checks pass, apply targeted probes based on the change type:

### Concurrency Probes
- Two simultaneous requests to the same resource
- Overlapping writes to shared state
- Race conditions in async operations

### Boundary Value Probes
- Empty string, empty array, empty object
- Maximum allowed value (integer overflow, string length)
- Zero, negative numbers, NaN
- Unicode edge cases (emojis, RTL text, combining characters)

### Idempotency Probes
- Same operation executed twice → same result as once
- Failed operation retried → correct recovery state

### Isolation Probes
- Operation on one resource doesn't affect another
- Transaction rollback leaves no side effects
- Error in one step doesn't corrupt previous steps

### Security Probes
- SQL/command injection in input fields
- Authentication bypass attempts
- Authorization escalation (lower privilege accessing higher privilege resources)
- Data leakage between users/tenants

## Anti-Rationalization Patterns

These are excuses agents (and humans) reach for when tempted to skip verification:

| Excuse | Reality | Counter-Action |
|--------|---------|----------------|
| "The code looks correct" | Reading code is not verification — run it | Execute the code and observe actual behavior |
| "The existing tests already pass" | Tests may only cover happy paths | Verify independently, don't trust existing coverage |
| "This is probably fine" | "Probably" is not verified | Run the actual check |
| "I don't have the right tools" | Check for available tools before assuming | Survey available tools first |
| "This would take too long" | Verification scope is not the agent's call | Execute the check regardless of time estimate |
| "It worked in my head" | Mental models miss edge cases | Run it for real |
| "The author is experienced" | Experienced authors make mistakes too | Trust evidence, not reputation |
| "This is a simple change" | Simple changes cause complex regressions | Verify the change area and its dependencies |

### Pattern: The Rationalization Cascade

Watch for chains of excuses that build on each other:

```
"I don't need to test this" → "The code looks right" → "Tests probably pass"
→ "Running tests takes too long" → "It's fine, moving on"
```

Each step seems reasonable in isolation. Together, they skip all verification. A verification agent must recognize this pattern and break the chain at the first excuse.

## Structured Verification Report

Every verification check must follow this format:

```
### Check: [what you're verifying]
**Command:** [exact command executed]
**Output:** [actual terminal output, truncated if long]
**Result:** PASS (or FAIL — with expected vs actual)
```

### Report Quality Criteria

- **Specific**: "PASS — auth middleware correctly rejects expired tokens" (good) vs. "PASS — it works" (bad)
- **Reproducible**: Anyone can run the same command and get the same result
- **Complete**: Both passing and failing checks are reported
- **Ordered**: Fast checks first, slow checks last (build → lint → test → manual)
- **Honest**: Failures are reported clearly, not downplayed or explained away

### Example Report

```
### Check: Build compiles without errors
**Command:** npm run build
**Output:** Build completed successfully in 3.2s
**Result:** PASS

### Check: Existing tests pass
**Command:** npm test
**Output:** 47 tests passed, 0 failed
**Result:** PASS

### Check: Empty input handling
**Command:** node parser.js --input ""
**Output:** Error: Input cannot be empty (exit code 1)
**Result:** PASS — gracefully rejects empty input

### Check: Unicode handling
**Command:** node parser.js --input "test\u0000null"
**Output:** Segmentation fault (exit code 139)
**Result:** FAIL — null byte causes crash
  Expected: Graceful error or sanitization
  Actual: Segfault
```
