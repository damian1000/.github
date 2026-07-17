# .github

Shared CI for the `damian1000` repositories. The `ci`, `codeql`, and `dep-review` pipelines are
defined once here as reusable workflows; each repository calls them so the pipeline is identical
everywhere and changes land in one place.

## Reusable workflows

| Workflow                             | Purpose                                                    |
| ------------------------------------ | ---------------------------------------------------------- |
| `.github/workflows/ci.yml`           | Attribution-history gate, Spotless, coverage-gated build, Codecov, test-report artifact. |
| `.github/workflows/codeql.yml`       | CodeQL analysis (`java-kotlin`).                            |
| `.github/workflows/dep-review.yml`   | Dependency review; fails a PR on a high-severity advisory. |

## Consuming them

A repository's `ci.yml` reduces to a caller that passes only what genuinely differs — the coverage
and test-report paths for its module layout:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build:
    uses: damian1000/.github/.github/workflows/ci.yml@main
    secrets: inherit
    with:
      coverage-report-files: build/reports/jacoco/test/jacocoTestReport.xml
      test-report-paths: build/reports/tests/test
```

Multi-module repositories pass comma-separated coverage files and newline-separated report paths:

```yaml
    with:
      coverage-report-files: core/build/reports/jacoco/test/jacocoTestReport.xml,app/build/reports/jacoco/test/jacocoTestReport.xml
      test-report-paths: |
        core/build/reports/tests/test
        app/build/reports/tests/test
```

`codeql.yml` and `dep-review.yml` callers take no inputs; they grant the token scopes the reusable
job needs (`security-events: write` / `pull-requests: write`).
