---
name: test-smell-detection
description: >
  Structured taxonomy audit for existing .NET tests. Returns a severity-ranked
  report with calibrated remediations using the testsmells.org 19-smell
  taxonomy — strictly more thorough than ad-hoc review and reuses the same
  catalog across runs.
  Load whenever a user asks to review, audit, assess, or sanity-check the
  design or quality of existing .NET tests: "review my tests", "audit my
  test suite", "are my tests well-designed?", "tests pass but feel off",
  "objective assessment of test design", "test design problems or
  anti-patterns" without naming specific patterns, "evaluate these tests
  before using as a template".
  DO NOT USE FOR: writing or scaffolding new tests, improving coverage by
  authoring tests (use code-testing-agent); quick pragmatic scan or audits
  that name specific anti-patterns such as flaky timing, ordering
  dependency, swallowed exceptions, coverage inflation, self-referential
  assertions (use test-anti-patterns); assertion depth only (use
  assertion-quality); tagging tests (use test-tagging).
license: MIT
---

# Test Smell Detection

Formal audit for existing tests. Detect smell patterns that reduce readability, reliability, and diagnostic value, then return severity-ranked findings with concrete fixes. If the suite is clean, say so explicitly.

## When to Use

- Review existing test files or a test project for smells or design problems.
- Give an objective quality assessment before reusing a suite as a template.
- Check integration tests without over-flagging legitimate external-resource usage.
- Produce a formal taxonomy-based report instead of a quick heuristic review.

## When Not to Use

- Writing or generating new tests from scratch (use `code-testing-generator`).
- Quick pragmatic anti-pattern review (use `test-anti-patterns`).
- Assertion-depth analysis only (use `assertion-quality`).
- Running or fixing failing tests directly (diagnose directly or use execution skills).

## Inputs

| Input | Required | Notes |
|---|---|---|
| Test code | Yes | One or more test files, or a test project/directory |
| Production code | No | Useful for judging whether a pattern is justified |
| Test framework markers | No | If needed, consult `dotnet-test-frameworks` for assertion, skip, and test-file markers |

## Workflow

1. **Gather context**
   - Read the provided test files or scan the test project.
   - Identify framework markers and whether the suite is unit, integration, or end-to-end.
2. **Classify smells**
   - Check each test method and class against the core taxonomy below.
   - Use `references/test-smell-catalog.md` for the extended catalog (19 smells total).
3. **Calibrate before flagging**
   - Downgrade legitimate integration-test patterns.
   - Do not flag simple foreach-assert loops, self-explanatory count values, or pending/inconclusive tests as assertion-free.
   - Capture-and-assert exception patterns are still a smell, but milder than swallowed exceptions.
4. **Report**
   - If the suite is clean, say so up front and keep suggestions optional.
   - Otherwise rank findings by severity, show exact locations, and give concrete remediations.

## Core smell taxonomy

| Smell | Flag when you see | Default severity | Calibration / exception |
|---|---|---|---|
| Conditional Test Logic | `if`, `else`, `switch`, ternary, `for`, `foreach`, `while` in test bodies | High | Simple foreach assertion loops are fine |
| Mystery Guest | File, DB, HTTP, env-var, or path dependency not made explicit | High | In-memory fakes/test handlers are fine; integration tests may justify real resources |
| Sleepy Test | `Thread.Sleep`, delay-based waiting, polling sleeps | High | Prefer deterministic synchronization |
| Assertion-Free Test | Test executes code but makes no real assertion | High | `DoesNotThrow`-style tests may be intentional; note that nuance |
| Eager Test | One test exercises many distinct production methods | Medium | Workflow/integration tests may legitimately span multiple calls |
| Magic Number Test | Unexplained numeric literal used as an expected value | Medium | Obvious counts or boundaries are fine |
| Sensitive Equality | Assertions compare `ToString()` output or other formatting-sensitive text | Medium | Prefer asserting semantic state |
| Exception Handling in Tests | `try`/`catch`, manual `throw`, or swallowed exceptions inside the test | Medium | Capture-and-assert is less severe than swallowing |
| General Fixture | Setup initializes fields used by only a minority of tests | Low | Shared expensive setup can be justified if documented |
| Ignored / Disabled Test | Skip/ignore attributes, disabled tests, or conditional compilation hiding tests | Low | Skips with a documented reason are less concerning |

## Decision procedure

1. **Scope first.** Decide whether the class is unit, integration, or end-to-end.
2. **Method pass.** For every test method, record candidate smells, file path, and method name.
3. **Calibration pass.** Apply the exception rules above before finalizing severity.
4. **Clean-suite check.** If only minor or contextually justified issues remain, acknowledge the suite as largely clean.
5. **Remediation pass.** For each reported smell, propose the smallest concrete rewrite that improves the test.

## Output format

### 1. Summary dashboard

| Severity | Smell count | Affected tests |
|---|---:|---:|
| High | n | n |
| Medium | n | n |
| Low | n | n |
| Total | n | n |

### 2. Findings by severity

For each finding, include:

- smell name and severity with a short rationale
- exact file path and test method
- a small code snippet showing the smell
- a concrete fix example
- risk if left unchanged

Example fix style:

```csharp
Assert.ThrowsException<ValidationException>(
    () => processor.ProcessOrder(emptyOrder));
```

### 3. Smell-free patterns

Call out clean Arrange-Act-Assert structure, good exception assertions, clear test names, or other strong patterns when present.

### 4. Prioritized remediation plan

Order fixes by:

1. false-confidence risks first (`Assertion-Free`, swallowed exceptions)
2. flakiness next (`Sleepy Test`, hidden external dependencies)
3. readability and maintainability issues after that

## Validation checklist

- [ ] Every finding names the file and test method
- [ ] Every finding shows the smell in a code snippet
- [ ] Every finding includes a concrete fix
- [ ] Integration tests are not over-penalized for legitimate patterns
- [ ] Clean suites are explicitly acknowledged as clean
- [ ] Severity is justified by evidence, not just by taxonomy label

## References

- Academic taxonomy: <https://testsmells.org>
- Extended catalog: `references/test-smell-catalog.md`
- Framework-specific markers and assertion/skip APIs: `dotnet-test-frameworks`
