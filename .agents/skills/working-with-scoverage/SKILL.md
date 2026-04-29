---
name: working-with-scoverage
description: Runs Scala 3 compilation tests with Scoverage instrumentation, explains coverage test filtering and excludelist behavior, and guides coverage-suite extension work. Use when working on coverage support, running coverage-enabled test suites, or updating the scoverage ignore excludelist.
---

# Working With Scoverage

Use this skill when running Scala 3 compilation tests with coverage instrumentation, narrowing a coverage run to a suite or a file, or maintaining `compiler/test/dotc/scoverage-ignore.excludelist`.

## Source Of Truth

Prefer the implementation over stale notes. The main files are:

- `project/Build.scala` for `testCompilation` argument handling and `--enable-coverage-phase`
- `compiler/test/dotty/Properties.scala` for `dotty.tests.filter` and `dotty.tests.instrumentCoverage`
- `compiler/test/dotty/tools/vulpix/ParallelTesting.scala` for filter matching
- `compiler/test/dotty/tools/dotc/CoverageSupport.scala` for coverage flag injection, skip logic, and verification
- `compiler/test/dotty/tools/TestSources.scala` for excludelist loading
- `compiler/test/dotc/scoverage-ignore.excludelist` for excluded tests

The original reference material for this skill lives in:

- `.ctxlayer/scoverage/test-surface/reference/01-scoverage-test-infrastructure-reference.md`
- `.ctxlayer/scoverage/test-surface/reference/02-scoverage-test-exclusion-reference.md`
- `.ctxlayer/scoverage/test-surface/reference/03-coverage-extension-implementation-guide.md`

## Core Commands

Use the wrapper script for coverage-enabled compilation test runs.

Run the full bootstrapped compilation coverage suite:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase"
```

Run a whole suite with coverage:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/pos"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/run"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/warn"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/explicit-nulls/pos"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/explicit-nulls/warn"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/init"
```

Run the rewrites suite with coverage:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase rewrites"
```

Use `rewrites`, not `tests/rewrites`. Rewrite tests are copied into `out/rewrites/...` before filtering, so `tests/rewrites` does not match the path seen by the filter.

Run a single test file with coverage:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/pos/i12345.scala"
```

Run without coverage for comparison:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation tests/pos"
```

## How Filtering Works

`testCompilation` forwards the remaining argument text into `-Ddotty.tests.filter=...`, and the test runner keeps a test when the filter string is contained in the relevant path.

- This is substring matching, not regex matching and not glob matching.
- For regular file-based suites, the relevant path is the source file path, such as `tests/pos/i12345.scala`.
- For directory-based separate-compilation tests, the relevant path is the directory path.
- For rewrites, the relevant path is under `out/rewrites/...`, so use `rewrites` as the filter.

To run several tests by pattern, choose a substring shared by all of them.

Examples:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/pos/i12"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/run/patmat"
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/init/special"
```

Useful narrowing trick:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/pos/"
```

The trailing slash matters. `tests/pos/` matches files directly under `tests/pos/`, but does not match sibling directories such as `tests/pos-macros/...`.

### Multiple Filters

The underlying property reader splits `dotty.tests.filter` on commas, but the standard `testCompilation` task is designed around a single filter argument. In normal use, prefer one shared substring per run. If you need disjoint groups, run separate commands instead of relying on ad hoc spacing.

## Excludelist Workflow

The coverage ignore list lives at:

```text
compiler/test/dotc/scoverage-ignore.excludelist
```

The file is only used when the run includes `--enable-coverage-phase`.

### File Format

- One entry per line
- Use the bare filename for file tests, for example `i10848a.scala`
- Use the bare directory name for directory tests, for example `alphanumeric-infix-operator-compat`
- Do not use full paths
- Blank lines are ignored
- Lines starting with `#` are comments
- Inline comments after `#` also work because the loader trims trailing comments
- Keep the file alphabetically sorted

### How Matching Works

`CoverageSupport.withCoverage()` excludes a test when any of these match an excludelist entry:

- a source file name
- a source file's parent directory name
- the basename of the test target title

That means the same list can exclude:

- a single file test like `tests/pos/i10848a.scala`
- a directory-backed test like `tests/pos-special/i18589/test_1.scala` by listing `i18589`
- separate-compilation targets whose title basename matches the entry

When an entry matches, the test is skipped before coverage flags are added, and the run prints:

```text
[Scoverage] Skipping test: <name> (matches scoverage ignore excludelist)
```

### Add Or Remove Entries

Add an entry by editing the excludelist directly, then keep it sorted.

Remove an entry by deleting the line and rerunning the relevant coverage command to see whether the test now executes cleanly.

Verify that exclusion works with a targeted run such as:

```bash
./project/scripts/sbt "scala3-bootstrapped/testCompilation --enable-coverage-phase tests/pos/erased-24.scala"
```

If `erased-24.scala` is on the list, the run should emit the skip message and not execute that test.

## Failure-Collection Workflow

When extending coverage to a new suite or cleaning up failures:

1. Run the suite with `--enable-coverage-phase`.
2. Inspect the newest log under `testlogs/tests-YYYY-MM-DD/`.
3. Find failed entries in the `Test Report` section.
4. Convert each failing path to the short excludelist form.
5. Add the entries to `compiler/test/dotc/scoverage-ignore.excludelist`.
6. Rerun the suite and confirm the failures are now skipped.

Useful log facts:

- Main logs live under `testlogs/tests-YYYY-MM-DD/tests-YYYY-MM-DD-THH-mm-ss.log`
- Failed tests appear in the final report as `    <path> failed`
- `testlogs/last-failed.log` is a separate rerun helper, not the main coverage run log

## Extending Coverage To Another Suite

Keep the implementation minimal and local.

- Keep coverage-specific logic in `compiler/test/dotty/tools/dotc/CoverageSupport.scala`
- Wrap the suite's `CompilationTest` in `withCoverage(...)`
- Use `runWithCoverageOrFallback[...]` or the existing coverage-aware test types instead of inventing a new flow
- Do not refactor unrelated test infrastructure while adding coverage to one suite

Canonical patterns:

- Compilation-only suites use `PosTestWithCoverage` or `WarnTestWithCoverage`
- Run suites use `RunTestWithCoverage`
- Rewrite suites use `RewriteTestWithCoverage`
- Aggregate suites should wrap the aggregated test, not each child just for coverage support

## Verification Checklist

Before claiming the change works:

1. Run at least one focused command with `--enable-coverage-phase`.
2. Confirm the target was actually selected by the filter you used.
3. If you changed exclusions, confirm the skip message appears for excluded tests.
4. If you changed coverage support code, make sure `verifyCoverageFile()` still succeeds.
5. Run the closest relevant baseline command without coverage if you need to isolate a coverage-only regression.
