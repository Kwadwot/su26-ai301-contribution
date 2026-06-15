# Contribution 1: Add compatibility test for $percentile

**Contribution Number:** 1  
**Student:** Kwadwo Twumasi  
**Issue:** [Add compatibility test for $percentile (second pass)
 #195](https://github.com/documentdb/functional-tests/issues/195)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because it aligns with my recent studies in the Python programming language, specifically when it
comes to writing tests of functions. In this case, I would be implementing and testing a Python compatibility test for this 
DocumentDB project's `$percentile` accumulator expression operator. This issue matches my skills since I have experience as a
Python programmer and working with a database (PostgreSQL and Supabase). I also chose this issue because I want to learn a bit 
about MongoDB as I work toward this contribution.

---

## Understanding the Issue

### Problem Description

The main problem this issue is trying to address is the lack of concrete exhaustive tests for the `$percentile` operator, since there is
only the smoke test in the current project.

### Expected Behavior

Besides the smoke test, the project requires more concrete and exhaustive tests implemented for the `$percentile` accumulator expression operator.

### Current Behavior

Currently, there is only one test for the `$percentile` operator, `test_smoke_accumulator_percentile.py`, which is checked and passes successfully.

### Affected Components

## `$percentile` Expression Operator — Codebase Map

`$percentile` is used here as an **expression** (evaluated per-document over an array `input`, inside stages like `$project` and `$set`), as opposed to its `$group` accumulator form.

### Dedicated test file (the subject of this issue)

| File | Status |
|------|--------|
| `documentdb_tests/compatibility/tests/core/operator/expressions/accumulator/percentile/test_smoke_expression_percentile.py` | Single happy-path smoke test (`@pytest.mark.smoke`, `$project` with `p: [0.5]`) — target for expanded coverage |

### Incidental references (expression form)

| File | Context |
|------|---------|
| `.../stages/project/test_project_expressions.py` | `$percentile` as expression inside `$project` |
| `.../stages/set/test_set_expressions.py` | `$percentile` as expression inside `$set` |
| `.../stages/bucket/test_bucket_accumulator_operators.py` | `$percentile` usage within `$bucket` |

### Supporting framework & metadata

| File | Role |
|------|------|
| `documentdb_tests/framework/error_codes.py` | `PERCENTILE_INVALID_P_FIELD_ERROR = 7750301`, `PERCENTILE_INVALID_P_VALUE_ERROR = 7750303` — for error-path assertions |
| `docs/feature-tree.csv` | Feature registry entry: `core,operator,expressions,accumulator,-,percentile` |
| `documentdb_tests/framework/executor.py` | `execute_command` — issues the aggregate command |
| `documentdb_tests/framework/assertions.py` | `assertSuccess` — result assertion helper |

### Related (accumulator form — out of scope for this issue)

The `$group` accumulator variant lives separately under `.../accumulators/percentile/` and is referenced in `.../stages/group/test_group_accumulators.py`. Distinct from the expression form addressed here.


---

## Reproduction Process

### Environment Setup

The framework expects a real MongoDB server reachable via a connection string, and
`$percentile` requires **MongoDB 7.0+** (the operator doesn't exist in earlier versions).

Challenges encountered and how I resolved them:

- **MongoDB Atlas free tier is incompatible.** M0/M2/M5 shared clusters cap database
  names at 38 bytes, but the framework generates longer unique per-test DB names
  (`test_<worker>_<hash>_<testname>`, truncated to 63 bytes, MongoDB's real limit).
  Tests failed with `AtlasError: Database name ... is too long`.
  **Fix:** ran a local MongoDB Community 7.0+ instance (`mongodb://localhost:27017`)
  instead, which has no such restriction.
- **Custom pytest options not recognized.** Running bare `pytest` from the repo root
  errored with `unrecognized arguments: --connection-string --engine-name`, because
  those options are registered in `documentdb_tests/conftest.py` (a subdirectory) and
  there's no `testpaths` in `pyproject.toml`.
  **Fix:** pass the test path so the subdir conftest loads at startup.

### Steps to Reproduce

1. Start a local MongoDB 7.0+ instance on `localhost:27017`.
2. Run the existing expression smoke test:
   ```bash
   pytest documentdb_tests/compatibility/tests/core/operator/expressions/accumulator/percentile/test_smoke_expression_percentile.py \
     --connection-string mongodb://localhost:27017 --engine-name mongodb -v
3. Observed result: the single test `test_smoke_expression_percentile` passes, but it is the only test in the folder. Coverage is limited
   to one happy path (`$project`, single `p: [0.5]`, `method: "approximate"`). No cases exist for multiple percentiles, `method` variants, `$set`,
   edge inputs (empty/null/non-numeric), or the documented error codes (`7750301`, `7750303`). This confirms the coverage gap the issue describes.

### Reproduction Evidence

- **Commit showing reproduction:** [`fix-issue-percentile-operator` branch](https://github.com/Kwadwot/functional-tests/tree/fix-issue-percentile-operator)
- **Screenshots/logs:** <img width="905" height="337" alt="image" src="https://github.com/user-attachments/assets/05a5c5ab-632f-46df-b992-1cd33ffd516e" />

- **My findings:** the `$percentile` expression operator is functional but minimally tested; expanding coverage means adding cases for multiple `p` values,
   both `approximate`/`discrete` (if supported) methods, `$set` usage, and error paths using the codes already defined in `framework/error_codes.py`.

---

## Solution Approach

### Analysis

Since this issue requires full coverage test implementations, there is no real error to identify and fix. Essentially,
the root cause of the documented error codes (`7750301`, `7750303`) is the lack of full coverage tests for the `$percentile`
expression operator.

### Proposed Solution

Expand the existing test module (`.../expressions/accumulator/percentile`) with a comprehensive set of compatibility
tests covering success paths, edge inputs, and error paths — reusing the framework's existing execution and assertion 
helpers rather than introducing new infrastructure.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The `$percentile` expression operator has only smoke-level coverage.

**Match:** Existing patterns to reuse:
- Smoke pattern: `execute_command(...)` + `assertSuccess(result, expected, msg=...)`
  (`test_smoke_expression_percentile.py`).
- Data-driven cases: the `StageTestCase` list pattern in
  `.../stages/group/test_group_accumulators.py` (`test_case.py` exposes an `error_code`
  field for negative cases).
- Assertion helpers in `framework/assertions.py`: `assertSuccess`, `assertFailure`,
  `assertFailureCode(result, expected_code)`, and `assertResult(..., error_code=...)`
  for combined success/error cases.
- Error codes already defined in `framework/error_codes.py` (import, no need to hardcode).

**Plan:**
1. Add success-path tests: multiple percentiles (`p: [0.25, 0.5, 0.95]`), `method`
   variants, and usage in both `$project` and `$set`.
2. Add edge-input tests: empty array, nulls/missing fields, non-numeric values — assert
   MongoDB's actual behavior.
3. Add error-path tests using `assertFailureCode` with `PERCENTILE_INVALID_P_FIELD_ERROR`
   and `PERCENTILE_INVALID_P_VALUE_ERROR` (e.g. `p` out of `[0,1]`, wrong type).
4. Apply correct markers (`@pytest.mark.aggregate`, plus `smoke` where appropriate) per the
   contribution guidelines, with a docstring per test.

**Implement:** WIP

**Review:** Self-review checklist (per `CONTRIBUTING.md`):
- [ ] Each test has a required operation marker and a clear docstring
- [ ] One behavior per test; assertions via framework helpers (no bare `assert`)
- [ ] Error codes imported from `error_codes.py`, not hardcoded
- [ ] `black .` / `isort .` clean; `pre-commit run --all-files` passes
- [ ] Expected values verified against real MongoDB 7.0+

**Evaluate:** Run the targeted suite against a local MongoDB 7.0+ instance and confirm all
cases pass.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
