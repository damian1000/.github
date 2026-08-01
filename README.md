# .github

Shared CI for the `damian1000` repositories. The `ci`, `codeql`, `dep-review`,
`dependency-check`, and `dependency-submission` pipelines are defined once here as reusable
workflows; each repository calls them so the pipeline is identical everywhere and changes land
in one place.

Every third-party action is pinned to a commit SHA with its version as a trailing comment. A
mutable tag like `@v7` resolves at run time, so whoever can move that tag can change what runs
against the repositories and their secrets. Dependabot's `github-actions` updates keep the pins
current, arriving as reviewable pull requests.

## Reusable workflows

| Workflow                                      | Purpose                                                                                                                |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `.github/workflows/ci.yml`                    | Attribution-history gate, Spotless, coverage-gated build, Codecov, test-report artifact.                               |
| `.github/workflows/codeql.yml`                | CodeQL analysis (`java-kotlin`).                                                                                       |
| `.github/workflows/dep-review.yml`            | Dependency review; fails a PR on a high-severity advisory.                                                             |
| `.github/workflows/dependency-check.yml`      | Weekly OWASP dependency-check; fails on CVSS >= 7.0. Needs an `NVD_API_KEY` secret.                                    |
| `.github/workflows/dependency-submission.yml` | Submits the resolved Gradle dependency graph. `dep-review.yml` and Dependabot alerts see no JVM dependency without it. |

## This repository's own gate

`.github/workflows/lint.yml` is the one workflow here that is not reusable. Every other file
is `on: workflow_call`, so none of them run against this repository's pull requests, which
left the repository every caller references at `@main` with no check to require. The lint job
runs `actionlint` over the workflow files and Prettier 3.4.2 over the YAML and Markdown — the
same formatter, version, and targets as the `config` Spotless block in the shared convention
plugins, so these files are held to the estate's bar without a Gradle build to run Spotless
from.

`actionlint` is installed from a fixed release archive whose published SHA-256 is verified
before it runs. Its own installer script lives on a mutable branch URL, which is the same
trust problem as a mutable action tag.

## Why dependency submission exists

GitHub does not resolve a Gradle build. Left to its automatic detection, a repository's
dependency graph contains its npm packages and the actions its workflows call, and nothing
else — every JVM dependency is invisible. Both pull-request-time dependency gates read that
graph, so both were inert for the ecosystem these repositories are written in: Dependabot
security alerts had nothing to match against the advisory database, and `dep-review.yml`
compared two graphs with no JVM entries on either side. OWASP dependency-check was the only
thing reading real coordinates, and it runs on a schedule rather than against a diff.

Callers trigger it on `push` to `main` and on `pull_request`: the comparison needs a snapshot
for the base and for the head. `dep-review.yml` waits up to ten minutes for the pull request's
snapshot rather than deciding on a graph that is still being written.

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

`dependency-submission.yml` is called on both `push` to `main` and `pull_request`, and needs no
inputs for a standard build:

```yaml
name: Dependency submission
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  submit:
    uses: damian1000/.github/.github/workflows/dependency-submission.yml@main
```
