# .NET Extension

Language-specific guidance for .NET (C#/F#/VB) test generation.

## Build Commands

| Scope | Command |
|-------|---------|
| Specific test project | `dotnet build MyProject.Tests.csproj` |
| Full solution (final validation) | `dotnet build MySolution.sln --no-incremental` |
| From repo root (no .sln) | `dotnet build --no-incremental` |

- Use `--no-restore` if dependencies are already restored
- Use `-v:q` (quiet) to reduce output noise
- Always use `--no-incremental` for the final validation build — incremental builds hide errors like CS7036

## Test Commands

| Scope | Command |
|-------|---------|
| All tests | `dotnet test` |
| Filtered | `dotnet test --filter "FullyQualifiedName~ClassName"` |
| After build | `dotnet test --no-build` |

- Use `--no-build` if already built
- Use `-v:q` for quieter output

## Lint Command

```bash
dotnet format --include path/to/file.cs
dotnet format MySolution.sln         # full solution
```

## Project Reference Validation

Before writing test code, read the test project's `.csproj` to verify it has `<ProjectReference>` entries for the assemblies your tests will use. If a reference is missing, add it:

```xml
<ItemGroup>
    <ProjectReference Include="../SourceProject/SourceProject.csproj" />
</ItemGroup>
```

This prevents CS0234 ("namespace not found") and CS0246 ("type not found") errors.

## Discovering Untested Files (`find-untested-sources`)

Before manually walking the repo with `find`/`grep`/`glob` to figure out which
files have tests, **invoke the `find-untested-sources` skill**. It is a
parse-only Roslyn static analyzer (no build, no `Compilation`, no coverage —
seconds on multi-thousand-file repos) that returns a deterministic JSON test
pairing map for the whole repo.

Use it whenever the request scope is larger than a single named file: full
project, "improve coverage", "add tests for module X", "what should I test
next", etc.

### Invocation

Invoke via the `skill` tool with name `find-untested-sources`. The skill's
SKILL.md tells you how to run its `scripts/Find-UntestedSources.cs` file-based
app on the repo root; it prints a JSON report to stdout shaped like:

```json
{
  "repo": "/abs/path",
  "elapsed_ms": 8883,
  "counts": { "source_files": 3036, "test_files": 867, "untested_files": 1852, "paired_files": 1184 },
  "untested": [
    { "source": "src/Foo/Bar.cs", "decl_count": 8, "suggested_test_path": "tests/Foo.Tests/Bar/BarTests.cs" }
  ],
  "source_to_tests": {
    "src/Foo/Baz.cs": ["tests/Foo.Tests/BazTests.cs"]
  }
}
```

`untested` is already sorted by `decl_count` (API surface) descending. Pass
`--top N` to truncate.

### How to use the output

- **Worklist** — drive your test-generation plan from the `untested` list
  (already ordered by `decl_count` descending). Skip your own discovery walk.
- **File placement** — write each new test file at `suggested_test_path`. It's
  derived from real `<ProjectReference>` edges, so it already lives in a test
  project that compiles against the source.
- **Weakly paired files** — entries in `source_to_tests` with exactly one
  referrer are good candidates for additional depth tests.
- **Sanity check after generating** — re-run after writing tests to confirm
  each new test file now appears in `source_to_tests` for its target.

This skill is **C#-only**. For other languages, fall back to manual discovery
or the language's own pairing conventions.

## Common CS Error Codes

| Error | Meaning | Fix |
|-------|---------|-----|
| CS0234 | Namespace not found | Add `<ProjectReference>` to the source project in the test `.csproj` |
| CS0246 | Type not found | Add `using Namespace;` or add missing `<ProjectReference>` |
| CS0103 | Name not found | Check spelling, add `using` statement |
| CS1061 | Missing member | Verify method/property name matches the source code exactly |
| CS0029 | Type mismatch | Cast or change the type to match the expected signature |
| CS7036 | Missing required parameter | Read the constructor/method signature and pass all required arguments |

## `.csproj` / `.sln` Handling

- During phase implementation, build only the specific test `.csproj` for speed
- For the final validation, build the full `.sln` with `--no-incremental`
- Full-solution builds catch cross-project reference errors invisible in scoped builds

### Registering a new test project

If a new test project was created, register it with the solution so `dotnet test` can discover it:

1. Use the exact solution or solution-filter target identified in `.testagent/research.md` or `.testagent/plan.md` — do not search for or substitute a different `.sln`, `.slnx`, or `.slnf` target.
2. If that target is a `.sln` or `.slnx`, run `dotnet sln <solution> add <test-project.csproj>`.
3. If the target is a `.slnf` (solution filter), also ensure the new project is included in the filter; adding only to the underlying `.sln` may not be enough for test discovery.
4. Skip this if the project is already included in the solution or solution filter used for testing.
5. Prefer the researched test command. If you need to run the solution directly, use `dotnet test --solution <solution>` only for repos on .NET SDK 10+ with MTP-style syntax; otherwise use the standard positional form `dotnet test <solution>`.

## MSTest Template

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace ProjectName.Tests;

[TestClass]
public sealed class ClassNameTests
{
    [TestMethod]
    public void MethodName_Scenario_ExpectedResult()
    {
        // Arrange
        var sut = new ClassName();

        // Act
        var result = sut.MethodName(input);

        // Assert
        Assert.AreEqual(expected, result);
    }

    [TestMethod]
    [DataRow(2, 3, 5, DisplayName = "Positive numbers")]
    [DataRow(-1, 1, 0, DisplayName = "Negative and positive")]
    public void Add_ValidInputs_ReturnsSum(int a, int b, int expected)
    {
        // Act
        var result = _sut.Add(a, b);

        // Assert
        Assert.AreEqual(expected, result);
    }
}
```

## Skip Coverage Tools

Do not configure or run code coverage measurement tools (coverlet, dotnet-coverage, XPlat Code Coverage). These tools have inconsistent cross-configuration behavior and waste significant time. Coverage is measured separately by the evaluation harness.
