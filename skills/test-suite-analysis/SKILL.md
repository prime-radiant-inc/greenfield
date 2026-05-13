---
name: test-suite-analysis
description: Layer 1 skill for extracting behavioral intelligence from test suites. Framework detection, test code reading strategy, test execution strategy, behavioral claim extraction with Given/When/Then mapping, e2e vs unit value classification. Loaded by the analyzer agent during Layer 1.
---

# Test Suite Analysis Methodology

Extract behavioral intelligence from test suites. Tests are executable specifications -- they encode what the system MUST do in a form that can be verified. A passing test is a confirmed behavioral contract.

## When to Use This Mode

Test suite analysis activates when:
- The target repository contains test files
- The discovery inventory identifies test files in the project
- Other modes discover test directories during analysis

This mode runs independently of all other intelligence sources. All output is **RAW** (test code references internal implementation details).

## Why Tests Are High-Value Intelligence

Tests are the only source type that is simultaneously:
- **Behavioral** -- they describe what the system does, not how it's built
- **Executable** -- they can be run to confirm the behavior still holds
- **Specific** -- they provide exact inputs, expected outputs, and edge cases
- **Maintained** -- failing tests get fixed, so they track current behavior

A single end-to-end test is worth more than a page of documentation because the test is verified by CI on every commit.

## Framework Detection

Identify the test framework(s) in use before analyzing test code. Different frameworks use different assertion styles, test organization, and execution models.

| Framework | Language | Detection Signals |
|-----------|----------|-------------------|
| Jest | JavaScript/TypeScript | `jest.config.*`, `describe(` / `it(` / `expect(` in `__tests__/` or `*.test.*`, `@jest/globals` imports |
| Playwright | JavaScript/TypeScript | `playwright.config.*`, `@playwright/test` imports, `page.goto(` / `page.click(` |
| Cypress | JavaScript/TypeScript | `cypress.config.*`, `cypress/` directory, `cy.visit(` / `cy.get(` |
| pytest | Python | `conftest.py`, `pytest.ini` / `pyproject.toml` with `[tool.pytest]`, files named `test_*.py` / `*_test.py`, `assert` statements |
| Go testing | Go | `*_test.go` files, `testing.T` / `testing.B` parameters, `go test` in CI config |
| RSpec | Ruby | `.rspec`, `spec/` directory, `spec_helper.rb`, `describe` / `it` / `expect` blocks |
| JUnit | Java/Kotlin | `@Test` annotations, `src/test/` directory, `assertEquals` / `assertThat` calls |
| XCTest | Swift/Objective-C | `XCTestCase` subclasses, `func test*()` methods, `XCTAssert*` calls |
| Catch2 | C++ | `#include <catch2/catch.hpp>`, `TEST_CASE(` / `SECTION(` / `REQUIRE(` macros |

### Detection Strategy

```bash
# Check for test configuration files
ls -la jest.config.* playwright.config.* cypress.config.* .rspec pytest.ini 2>/dev/null

# Check pyproject.toml for pytest config
grep -l '\[tool\.pytest' pyproject.toml 2>/dev/null

# Find test directories
find . -maxdepth 3 -type d \( -name "__tests__" -o -name "test" -o -name "tests" -o -name "spec" -o -name "cypress" \) 2>/dev/null

# Find test files by naming convention
find . -maxdepth 4 -type f \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" -o -name "*_test.*" \) 2>/dev/null | head -50

# Count test files per pattern
echo "Jest/Mocha-style:" && find . -name "*.test.*" -o -name "*.spec.*" 2>/dev/null | wc -l
echo "Python-style:" && find . -name "test_*.py" -o -name "*_test.py" 2>/dev/null | wc -l
echo "Go-style:" && find . -name "*_test.go" 2>/dev/null | wc -l
echo "JUnit-style:" && find . -path "*/src/test/*" -name "*.java" 2>/dev/null | wc -l
```

Write detection results to `workspace/raw/test-evidence/test-inventory.md`.

## Strategy 1: Read Test Code

Read test files directly and extract behavioral claims. This strategy always works -- it requires no working environment, no dependencies, and no execution.

### 1.1 Test File Inventory

```bash
# Build complete inventory of test files with metadata
find . -type f \( -name "*.test.*" -o -name "*.spec.*" -o -name "test_*" -o -name "*_test.*" -o -name "*_test.go" \) 2>/dev/null | while read f; do
  lines=$(wc -l < "$f")
  echo "$lines $f"
done | sort -rn
```

### 1.2 Assertion Extraction

For each test file, extract the assertions -- these are the behavioral contracts:

```bash
# Jest/Mocha assertions
grep -n "expect\|assert\|should\|toBe\|toEqual\|toContain\|toThrow\|toHaveBeenCalled" "$TEST_FILE"

# pytest assertions
grep -n "assert \|assert_\|assertEqual\|assertRaises\|pytest.raises" "$TEST_FILE"

# Go test assertions
grep -n "t\.Error\|t\.Fatal\|t\.Log\|assert\.\|require\." "$TEST_FILE"

# RSpec assertions
grep -n "expect\|should\|is_expected\|eq(\|include(\|raise_error" "$TEST_FILE"
```

### 1.3 Given/When/Then Extraction

Transform test code into behavioral claims using Given/When/Then structure:

For each test case (`it(`, `test(`, `func Test*`, `def test_*`), extract:

- **Given** (setup/preconditions): fixture creation, mock configuration, state initialization
- **When** (action): the function call, API request, or user action being tested
- **Then** (assertions): the expected outcomes encoded in assertions

```markdown
## Test: "should reject expired tokens"

**Given:** A token with expiry date in the past
**When:** The token is validated via `checkPermissions()`
**Then:**
- Returns false
- Sets error to "TOKEN_EXPIRED"
- Does not call the downstream service

**Source:** `auth.test.ts:45-62`
**Confidence:** confirmed (test assertion is an explicit behavioral contract)
```

### 1.4 E2E vs Unit Value Classification

Not all tests carry equal behavioral intelligence value:

| Test Type | Detection Signals | Behavioral Value |
|-----------|-------------------|-----------------|
| End-to-end (e2e) | Browser automation, HTTP requests to running server, multi-service interaction | **High** -- tests the system as a user experiences it |
| Integration | Database connections, external service calls, multi-module interaction | **High** -- tests behavioral contracts between components |
| Functional | Single module tested with real dependencies | **Medium** -- tests module-level behavioral contracts |
| Unit | Mocked dependencies, isolated function tests | **Lower** -- tests implementation contracts, not user-visible behavior |
| Snapshot | `toMatchSnapshot()`, `toMatchInlineSnapshot()` | **Low** -- captures output format, not behavioral intent |

Focus extraction effort on e2e and integration tests first. Unit tests fill in details after the behavioral surface is mapped.

### 1.5 Edge Case Mining

Tests are the richest source of edge case documentation. Look for:

```bash
# Boundary value tests
grep -n "boundary\|limit\|max\|min\|overflow\|underflow\|zero\|empty\|null\|undefined" "$TEST_FILE" -i

# Error condition tests
grep -n "error\|fail\|reject\|throw\|invalid\|unauthorized\|forbidden\|timeout" "$TEST_FILE" -i

# Concurrency tests
grep -n "concurrent\|parallel\|race\|deadlock\|lock\|mutex\|async\|await" "$TEST_FILE" -i

# Special character and encoding tests
grep -n "unicode\|utf\|encoding\|escape\|special\|whitespace" "$TEST_FILE" -i
```

Write behavioral claims to `workspace/raw/test-evidence/behavioral-claims.md`.
Write e2e flow documentation to `workspace/raw/test-evidence/e2e-flows.md`.
Write edge case documentation to `workspace/raw/test-evidence/edge-cases.md`.

## Strategy 2: Run Test Suite

Execute the test suite and observe its behavior. This strategy requires a working environment with all dependencies installed. It produces higher-confidence claims but has higher setup cost.

### 2.1 Prerequisites

Before attempting test execution:
- Verify the container has all dependencies installed
- Check for required environment variables or config files
- Look for test setup scripts (`beforeAll`, `setUp`, fixtures, factories)
- Identify tests that require external services (databases, APIs)

### 2.2 Execute with Maximum Verbosity

```bash
# Jest
npx jest --verbose --no-coverage 2>&1 | tee workspace/raw/test-evidence/run-output.txt

# pytest
python -m pytest -v --tb=long 2>&1 | tee workspace/raw/test-evidence/run-output.txt

# Go
go test -v ./... 2>&1 | tee workspace/raw/test-evidence/run-output.txt

# RSpec
bundle exec rspec --format documentation 2>&1 | tee workspace/raw/test-evidence/run-output.txt
```

### 2.3 Observe Runtime Behavior

During test execution, capture:

- **Network calls** -- tests that make HTTP requests reveal API contracts
- **File I/O** -- tests that read/write files reveal data format contracts
- **Timing** -- slow tests may indicate external dependency interaction
- **Failures** -- failed tests reveal behavioral regressions or environment-specific behavior
- **Warnings** -- deprecation warnings and lint output reveal upcoming behavioral changes

```bash
# Capture network activity during tests (if strace/dtrace available)
strace -e trace=network -f npx jest 2> workspace/raw/test-evidence/network-trace.txt

# Capture file I/O during tests
strace -e trace=file -f npx jest 2> workspace/raw/test-evidence/file-trace.txt
```

### 2.4 Failure Analysis

Failed tests are behavioral intelligence:
- A failing test documents a behavioral contract that is currently violated
- The expected value in the assertion documents what the behavior SHOULD be
- The actual value documents what the behavior currently IS
- The gap between expected and actual is a behavioral specification

Record every failure with:
- Test name and location
- Expected behavior (from the assertion)
- Actual behavior (from the error output)
- Whether this appears to be a genuine regression or an environment issue

## Provenance Rules

### Source Type

All claims from test suite analysis use `source=test-suite`:

```markdown
- Expired tokens are rejected with a TOKEN_EXPIRED error
  <!-- cite: source=test-suite, ref=src/auth/__tests__/auth.test.ts:45-62, confidence=confirmed, agent=test-analyzer -->
```

### Confidence Levels

- **confirmed** -- the test assertion explicitly encodes the behavioral claim AND the test passes (or the claim is directly readable from the test code regardless of execution)
- **inferred** -- the behavioral claim is derived from test setup, fixture data, or mock configuration rather than direct assertion
- **assumed** -- the behavioral claim is derived from test naming, file organization, or structural patterns rather than assertion content

### Test Code Assertions Are Confirmed

Unlike source code analysis (where claims are typically `inferred`), behavioral claims extracted directly from test assertions are `confirmed`. A test assertion is an explicit, executable behavioral contract. The developer who wrote `expect(result).toBe(42)` is asserting that this behavior is required.

### Cite As You Go

Every behavioral claim gets an inline citation immediately after the claim. The `ref` field should be `<file-path>:<line-range>`.

## Output Structure

```
workspace/raw/test-evidence/
    test-inventory.md          # Framework detection, test file inventory, classification
    behavioral-claims.md       # Behavioral claims in Given/When/Then format
    e2e-flows.md               # End-to-end flow documentation from e2e/integration tests
    edge-cases.md              # Edge cases, boundary conditions, error handling from tests
    run-output.txt             # Raw test execution output (if Strategy 2 was used)
```

## Rules

1. **Test code is RAW** -- test files reference internal implementation details. All output goes to `workspace/raw/`.
2. **Assertions are behavioral contracts** -- treat every assertion as a confirmed behavioral claim. The developer is stating that this behavior is required.
3. **E2E tests first** -- prioritize end-to-end and integration tests over unit tests. They encode user-visible behavior.
4. **Do not test mocked behavior** -- a test that asserts a mock was called proves nothing about the system's actual behavior. Skip mock-only tests when extracting behavioral claims.
5. **Failed tests are data** -- a failing test documents both the expected behavior (from the assertion) and the actual behavior (from the error). Record both.
6. **Strategy 1 always works** -- reading test code requires no environment. Always perform Strategy 1. Strategy 2 is additive and optional.
7. **Cite as you go** -- every behavioral claim gets an inline `<!-- cite: -->` comment immediately after the claim. Never defer citation to a later step.
8. **Map the full surface** -- do not stop after the first interesting test file. Build a complete inventory before deep-diving into individual files.
