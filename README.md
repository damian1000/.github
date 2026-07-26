# .github

Shared CI for the `damian1000` repositories. The `ci`, `codeql`, `dep-review`, and
`dependency-check` pipelines are defined once here as reusable workflows; each repository calls
them so the pipeline is identical everywhere and changes land in one place.

Every third-party action is pinned to a commit SHA with its version as a trailing comment. A
mutable tag like `@v7` resolves at run time, so whoever can move that tag can change what runs
against the repositories and their secrets. Dependabot's `github-actions` updates keep the pins
current, arriving as reviewable pull requests.

## Reusable workflows

| Workflow                             | Purpose                                                    |
| ------------------------------------ | ---------------------------------------------------------- |
| `.github/workflows/ci.yml`           | Attribution-history gate, Spotless, coverage-gated build, Codecov, test-report artifact. |
| `.github/workflows/codeql.yml`       | CodeQL analysis (`java-kotlin`).                            |
| `.github/workflows/dep-review.yml`   | Dependency review; fails a PR on a high-severity advisory. |
| `.github/workflows/dependency-check.yml` | Weekly OWASP dependency-check; fails on CVSS >= 7.0. Needs an `NVD_API_KEY` secret. |

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

`dependency-check.yml` runs on the caller's own weekly schedule. It is deliberately not part of
`ci.yml`: the scan reports advisories published since the last build rather than anything about
the diff, and pull requests are already gated by `dep-review.yml`. A multi-module root passes
`task: dependencyCheckAggregate` and the aggregate report path:

```yaml
name: Dependency check
on:
  schedule:
    - cron: "0 6 * * 1"
  workflow_dispatch:
jobs:
  scan:
    uses: damian1000/.github/.github/workflows/dependency-check.yml@main
    secrets: inherit
```
