# Contribution 1: Add compatibility test for $percentile

**Contribution Number:** 1  
**Student:** Kwadwo Twumasi  
**Issue:** [Add compatibility test for $percentile (second pass)
 #195](https://github.com/documentdb/functional-tests/issues/195)  
**Status:** Phase IV Complete

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
- [x] Each test has a required operation marker and a clear docstring
- [x] One behavior per test; assertions via framework helpers (no bare `assert`)
- [x] Error codes imported from `error_codes.py`, not hardcoded
- [x] `black .` / `isort .` clean; `pre-commit run --all-files` passes
- [x] Expected values verified against real MongoDB 7.0+

**Evaluate:** Run the targeted suite against a local MongoDB 7.0+ instance and confirm all
cases pass.

---

## Testing Strategy

> Note: This contribution *adds* compatibility tests, so the deliverable is the test
> suite itself. The categories below describe the coverage added rather than tests
> written against new product code. All cases run against a live MongoDB instance via
> the framework's `execute_expression` helpers.

### Test Coverage Added (`$percentile` expression operator)

Located in `documentdb_tests/compatibility/tests/core/operator/expressions/accumulator/percentile/`,
split by aspect (mirroring the `$avg` sibling). **49 cases total**, all passing.

- [x] **Core selection & ordering** (`test_percentile_core.py`): median, p=0 (min), p=1 (max),
      single-element, unsorted input, all-equal, large (10k) input, multiple/descending/duplicate p,
      `[0.0, 1.0]` boundaries.
- [x] **Input forms** (`test_percentile_input_forms.py`): `$literal` array, raw array, scalar,
      expression operator (`$concatArrays`), field reference, dotted path, empty array → `[null]`.
- [x] **Numeric types & special values** (`test_percentile_data_types.py`): int32/int64/double/
      Decimal128, mixed types, return-type-is-double, NaN/±Infinity ordering in input.
- [x] **Non-numeric handling** (`test_percentile_non_numeric.py`): non-numeric ignored;
      all-non-numeric and all-null → `[null]`.
- [x] **Null/missing input** (`test_percentile_null.py`): null and missing field → `[null]`
      (incl. one null per requested p).
- [x] **`p` validation** (`test_percentile_p_validation.py`): out-of-range, scalar/empty/non-numeric
      `p`, and valid `0.0`/`1.0` boundaries, asserting the documented error codes.
- [x] **`method` & spec validation** (`test_percentile_method.py`): unsupported/invalid/wrong-type
      method, missing/unknown spec fields, non-object spec.

### Cross-Suite Verification

- [x] Full percentile folder (new files + untouched smoke test): **49 passed**.
- [x] Regression on the reference sibling `$avg` suite: **224 passed** (no breakage from the
      shared `error_codes.py` additions).
- [x] Lint/type gates: `isort`, `flake8`, `mypy` clean. (`black` runs in CI, but local run is blocked
      by a known Python 3.12.5 AST-checker bug, not a formatting issue.)


### Manual Testing

Expected values and error codes were **captured empirically** against a local MongoDB 8.3.4 instance
(rather than assumed), via a throwaway probe script exercising every planned case. Two behaviors
surfaced that shaped the suite:

- `method: "discrete"` / `"continuous"` are **not supported** on 8.3.4 — only `"approximate"`
  (others return `BadValue`). Asserted as error cases.
- Passing **`NaN` as a `p` value crashed the `mongod` process** — excluded from the suite (a
  server-crashing test shouldn't ship) and flagged as a likely upstream robustness bug.

---

## Implementation Notes

### Week 3 Progress

- **Environment:** Settled on a local MongoDB 8.3.4 instance (`mongodb://localhost:27017`).
  MongoDB Atlas free tier was ruled out — its 38-byte database-name cap conflicts with the
  framework's auto-generated per-test DB names.
- **First pass:** Implemented `$percentile` expression coverage as a single file using
  `StageTestCase` + raw `execute_command`. 18 cases, green.
- **Review against project conventions** (`TEST_FORMAT.md`, `TEST_COVERAGE.md`, `FOLDER_STRUCTURE.md`
  and the `$avg` reference) revealed structural divergences → refactored.
- **Refactor:** Moved to the idiomatic shape — a `PercentileTest(BaseTestCase)` dataclass + a
  `percentile_spec()` builder in a local `utils/`, the shared `execute_expression*` helpers, and
  `assert_expression_result`. Split into 7 aspect files and expanded coverage to 49 cases.
- **Captured all expected values** from the live server; resolved two new server error codes.

### Code Changes

- **Files added:**
  - `expressions/accumulator/percentile/utils/percentile_common.py` — `PercentileTest`, `percentile_spec()`
  - `test_percentile_core.py`, `test_percentile_input_forms.py`, `test_percentile_data_types.py`,
    `test_percentile_non_numeric.py`, `test_percentile_null.py`, `test_percentile_p_validation.py`,
    `test_percentile_method.py`
- **Files modified:**
  - `documentdb_tests/framework/error_codes.py`: added `PERCENTILE_INVALID_P_TYPE_ERROR = 7750302`
    and `PERCENTILE_SPEC_NOT_OBJECT_ERROR = 7436200` (both sorted; both confirmed against server output).
- **Files removed:**
  - `test_expression_percentile.py`: superseded by the aspect-split files.
- **Key commits:** [Commit 0a1ed4b](https://github.com/Kwadwot/functional-tests/commit/0a1ed4b7d91e7ce002a0ec0625455458bbc19cbd)
- **Approach decisions:**
  - *Followed the `$avg` pattern exactly* (dataclass + shared helpers + aspect-split files) so the
    suite reads like its siblings and reviewers can pattern-match it.
  - *Captured expected values empirically* instead of hand-deriving them — appropriate for a
    compatibility suite, and it caught the t-digest rank behavior and the always-double return type.
  - *Scoped to the expression form only.* Pipeline-context wiring (§11) and the accumulator form are
    noted as out of scope / follow-ups.
  - *Added error-code constants rather than hardcoding* the two new codes, per `TEST_FORMAT.md`.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted](https://github.com/documentdb/functional-tests/pull/655)

**PR Description:**
### What does this PR do?

Adds comprehensive compatibility tests for the `$percentile` aggregation **expression** operator (used inside `$project`/`$set`/`$addFields`), which previously had only a single happy-path smoke test. The new tests live under `expressions/accumulator/percentile/` and follow the `$avg` sibling pattern: a `PercentileTest(BaseTestCase)` dataclass plus a `percentile_spec()` builder in `utils/`, the shared `execute_expression` helpers, and `assert_expression_result`. Coverage is split by aspect across 7 files (49 cases): core selection and `p` ordering, input forms, numeric types and special values, non-numeric handling, null/missing input, `p` validation, and `method`/spec validation. Two error-code constants used by the tests are added to `framework/error_codes.py`: `PERCENTILE_INVALID_P_TYPE_ERROR` (7750302) and `PERCENTILE_SPEC_NOT_OBJECT_ERROR` (7436200).

### Why was this PR needed?

`$percentile` is a tracked feature in the taxonomy (`docs/feature-tree.csv`) but had only smoke-level coverage. Characterizing its behavior against a live server surfaced specifics worth pinning down: under `method: "approximate"` it returns an order statistic at rank ceil(p·n) and always returns `double` (even for Decimal128 input); non-numeric/null/missing inputs yield `[null]`; and malformed `p`/`method`/spec values map to distinct error codes. Two findings emerged during investigation (captured on MongoDB 8.3.4): `method: "discrete"` and `"continuous"` are **not supported** (only `"approximate"`, others return `BadValue`), and passing **`NaN` as a `p` value crashed the `mongod` process** — that case was deliberately excluded from the suite as a likely upstream robustness bug.

### What are the relevant issue numbers?

Closes #195

### How to test it (on a MongoDB instance) + Screenshot:

Test run against MongoDB 8.3.4:
``` shell
pytest documentdb_tests/compatibility/tests/core/operator/expressions/accumulator/percentile/ --connection-string mongodb://localhost:27017 --engine-name mongodb -q
```
49 passed in 1.18s
#### Tests success screenshot:
<img width="988" height="312" alt="image" src="https://github.com/user-attachments/assets/f344293d-0a86-478f-8fd3-82719b87bd03" />



Regression check on the reference sibling suite (`$avg`): `224 passed`.

### Does this PR meet the acceptance criteria?

- [x] Tests added for new/changed behavior — 49 cases across 7 files
- [x] All tests passing — 49 passed; `$avg` regression 224 passed; isort/flake8/mypy clean
- [x] Follows project style guide — mirrors the `$avg` pattern per `TEST_FORMAT.md` / `TEST_COVERAGE.md` / `FOLDER_STRUCTURE.md`
- [x] No breaking changes introduced — additive only (new tests + two sorted error-code constants)
- [x] Documentation updated (if applicable) — n/a (test-only contribution; no user-facing docs affected)


**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

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
