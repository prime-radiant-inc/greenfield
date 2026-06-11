---
name: fidelity-validation
description: Cross-validates sanitized output specs against raw source specs to detect lost behavioral detail, dropped constants, missing features, or diluted precision. Run AFTER sanitization and AFTER contamination audit passes.
---

# Fidelity Validation

Sanitization removes source code identifiers. But aggressive sanitization can also remove behavioral detail that an implementor needs. This skill detects information loss.

## Why This Exists

The sanitization pass rewrites raw specs into output specs. Each rewrite risks:
- **Dropped behaviors**: A feature or edge case described in the raw specs that doesn't appear in the output specs
- **Lost constants**: A numeric threshold, timeout, limit, or size that was accidentally removed or changed during rewriting
- **Diluted precision**: A specific behavioral rule replaced with vague language ("the system handles errors" instead of "the system retries with configured backoff — N attempts, base delay, maximum delay, and jitter")
- **Missing decision trees**: A conditional behavior with multiple branches that was simplified to just the happy path
- **Dropped error conditions**: Error handling that was described in the raw specs but omitted from the output specs
- **Feature gaps**: Entire features or sub-features present in the raw specs but missing from the output domain spec

## When to Run

Run this AFTER:
1. Sanitization (Layer 5) is complete
2. Contamination audit (Layer 6) passes — output specs are confirmed free of source identifiers
3. All remediation/rewriting rounds are done

This is the final quality gate before handing the output to the implementer.

## The Principle

**Every behavioral claim in the raw specs must have a corresponding claim in the output specs — with equal or greater precision.**

The output specs may use different words (that's the point of sanitization), but it must convey the same behavior. If the raw spec says "retry N times with a specific base, cap, and jitter", the output specs must say the same — not just "the system retries on failure."

## Process

### Phase 1: Build the Source Claim Inventory

For each raw module spec (`workspace/raw/specs/modules/*.md`), extract:
1. **All numeric constants** — timeouts, limits, sizes, counts, thresholds, intervals, percentages
2. **All behavioral rules** — "when X happens, the system does Y"
3. **All error conditions** — what errors occur and how they're handled
4. **All state transitions** — state machines, mode changes, lifecycle events
5. **All decision trees** — if/else branches, priority orders, cascades
6. **All features** — distinct capabilities described

Write this inventory to `workspace/raw/audit/fidelity-inventory.md`.

### Phase 2: Cross-Validate Against Output

For each item in the source claim inventory, search the corresponding output domain spec(s) for a matching behavioral claim.

**Match criteria:**
- The output specs describes the same behavior (possibly with different words)
- Numeric constants match exactly (no rounding, no approximation)
- Decision branches are all present (not just the happy path)
- Error conditions are all present
- State transitions are complete

**Report format per item:**

```
### [RAW-SPEC: section-name] Claim: "description of the behavioral claim"
- Constant/Rule/Error/State/Decision/Feature
- Raw value: [exact value from raw specs]
- Output location: [file:section where it appears in the output] or MISSING
- Output value: [exact value from output specs] or N/A
- Status: MATCH | WEAKENED | MISSING | CHANGED
- Notes: [if WEAKENED/MISSING/CHANGED, explain what was lost]
```

Status definitions:
- **MATCH**: Output spec conveys the same behavior with the same precision
- **WEAKENED**: Output spec mentions the behavior but with less precision (e.g., "retries on failure" instead of "retries with specific backoff params")
- **MISSING**: Output spec does not mention this behavior at all
- **CHANGED**: Output spec describes different behavior for the same scenario

### Phase 3: Also Validate Cross-Cutting Artifacts

Repeat Phase 2 for:
- Journey specs: raw journeys vs output journeys
- Contracts: raw contracts vs output contracts
- Protocol specs: raw protocols vs output protocols
- Test vectors: raw test vectors vs output test vectors (constants must match)
- Acceptance criteria: raw ACs vs output ACs (criteria must be present)

### Phase 4: Severity Assessment

Categorize all WEAKENED, MISSING, and CHANGED findings:

| Severity | Criteria | Action |
|----------|----------|--------|
| **P0-CRITICAL** | A P0 behavior (critical path) is MISSING or CHANGED | Must fix before implementation |
| **P0-WEAKENED** | A P0 behavior lost precision (constants, branch coverage) | Must fix before implementation |
| **P1-MISSING** | A P1 behavior is entirely absent from the output | Should fix |
| **P1-WEAKENED** | A P1 behavior lost precision | Should fix |
| **P2-MISSING** | A P2/P3 behavior is absent | Advisory — may fix |
| **P2-WEAKENED** | A P2/P3 behavior lost precision | Advisory — may fix |

### Phase 5: Write Report

Write the full report to `workspace/raw/audit/fidelity-report.md`.

The report SHALL include:
1. **Summary statistics**: total claims checked, MATCH count, WEAKENED count, MISSING count, CHANGED count
2. **P0-CRITICAL and P0-WEAKENED findings**: full details with exact raw vs output text
3. **P1 findings**: full details
4. **P2 findings**: summary only
5. **Verdict**: PASS (zero P0 issues) or FAIL (any P0 issue exists)

Write a summary to `workspace/output/audit/fidelity-summary.md`:
- Verdict: PASS or FAIL
- Total claims validated
- Match rate percentage
- Number of issues by severity
- NO details about what was found or what raw specs say — this is an implementer-facing artifact

## Agent Dispatch Pattern

The fidelity validation is compute-heavy (reading all raw specs AND all output specs). Dispatch as:

1. **Inventory builders** (parallel, one per 4-5 raw module specs): Extract behavioral claims from raw specs → `workspace/raw/audit/fidelity-inventory-batch-N.md`

2. **Cross-validators** (parallel, one per output domain spec): Read the relevant inventory batches + the output domain spec → produce per-domain findings

3. **Aggregator** (sequential): Merge all findings → severity assessment → final report

Each agent is dispatched as `greenfield:analyzer` since they need to read both raw/ and output/.

## What This Does NOT Check

- Contamination (that's the Layer 6 audit's job)
- Structural organization (output specs may organize differently than raw — that's fine)
- Prose quality (rewording is expected)
- Provenance citations (these are intentionally stripped)

## Relationship to Other Skills

- **spec-sanitization**: Creates the output specs this skill validates
- **second-pass-review**: Checks for contamination; this skill checks for information loss
- **validation-methodology**: Defines the acceptance criteria that both raw and output specs must satisfy
- **behavioral-spec-writing**: Defines what behavioral claims look like

Together, the audit (Layer 6) and fidelity validation form a two-sided check:
- Layer 6 asks: "Is there anything in output/ that shouldn't be?" (contamination)
- Fidelity validation asks: "Is there anything NOT in output/ that should be?" (information loss)
