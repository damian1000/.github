# .github

Shared CI and delivery for the `damian1000` repositories. The `ci`, `codeql`, `dep-review`,
`dependency-check`, `dependency-submission`, `automerge`, `deploy`, and `release` pipelines are
defined once here as reusable workflows; each repository calls them so the pipeline is identical
everywhere and changes land in one place.

Every third-party action is pinned to a commit SHA with its version as a trailing comment. A
mutable tag like `@v7` resolves at run time, so whoever can move that tag can change what runs
against the repositories and their secrets. Dependabot's `github-actions` updates keep the pins
current, arriving as reviewable pull requests.

## Reusable workflows

| Workflow                                      | Purpose                                                                                                                                                                |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.github/workflows/ci.yml`                    | Attribution-history gate, Spotless, coverage-gated build, Codecov, test-report artifact.                                                                               |
| `.github/workflows/codeql.yml`                | CodeQL analysis (`java-kotlin`).                                                                                                                                       |
| `.github/workflows/dep-review.yml`            | Dependency review; fails a PR on a high-severity advisory.                                                                                                             |
| `.github/workflows/dependency-check.yml`      | Weekly OWASP dependency-check; fails on CVSS >= 7.0. Needs an `NVD_API_KEY` secret.                                                                                    |
| `.github/workflows/dependency-submission.yml` | Submits the resolved Gradle dependency graph. `dep-review.yml` and Dependabot alerts see no JVM dependency without it.                                                 |
| `.github/workflows/automerge.yml`             | Enables auto-merge so GitHub squash-merges once the required checks pass. Needs an `AUTOMERGE_TOKEN` secret.                                                           |
| `.github/workflows/deploy.yml`                | Production deploy: re-runs the gate, ships the tested artifact, switches release atomically, gates on readiness, rolls back on the host. Needs the `DEPLOY_*` secrets. |
| `.github/workflows/release.yml`               | Verifies a tagged commit, then publishes its release. Notes-only unless artifacts are named.                                                                           |

## This repository's own gate

`lint.yml` and `auto-merge.yml` are the two workflows here that are not reusable — everything
else in the table is `on: workflow_call`. `auto-merge.yml` is an ordinary caller of
`automerge.yml` beside it, so this repository is not a special case.

`.github/workflows/lint.yml` exists because none of the reusable files run against this
repository's own pull requests, which left the repository every caller references at `@main`
with no check to require. The lint job
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

## Why auto-merge does not use `GITHUB_TOKEN`

GitHub does not raise workflow runs for events triggered by `GITHUB_TOKEN`. A merge performed
with it therefore lands on `main` and starts nothing — no CI run against `main`, and, in the
repositories that deploy on push, no deployment. The evidence is three Dependabot pull requests
that auto-merged on 2026-07-14 under the previous `GITHUB_TOKEN` workflow:

| Repository           | Merge commit | Check runs | Workflow runs |
| -------------------- | ------------ | ---------- | ------------- |
| bank-csv-to-qif      | `98f3eacf`   | none       | none          |
| kotlin-blockchain    | `d8a9fc43`   | none       | none          |
| sudoku-dancing-links | `153e95ee`   | none       | none          |

`automerge.yml` merges with `AUTOMERGE_TOKEN`, a fine-grained personal access token, so the
merge is attributed to a person and the push behaves like any other.

It also runs on `pull_request_target` rather than `pull_request`. A `pull_request` run raised
by Dependabot is given Dependabot's secret scope instead of the repository's Actions secrets,
so the token would arrive empty. `pull_request_target` reads them because it runs in the base
repository's context; the trigger is only hazardous when a workflow checks out and executes the
pull request's code, and this one checks out nothing.

Auto-merge is restricted to non-draft pull requests authored by `damian1000` or
`dependabot[bot]`. These repositories are public, and a green build is not a review.

## How a change reaches production

`deploy.yml` runs on merge to `main` for every service. Five repositories previously carried five
hand-maintained copies of it, and those copies had already drifted in ways nobody chose: the
readiness hold existed in two of them, `systemctl enable` in four. A duplicated deploy script
diverges quietly, and the divergence surfaces during an incident.

The pipeline re-runs the full quality gate before packaging rather than trusting an earlier
workflow, and ships the bytes it just tested. Delivery owns its own go/no-go.

Four properties are the reason it is shaped this way:

- **The host key is pinned** from a file in the calling repository, so the deploy authenticates the
  host instead of trusting whatever answers on the address.
- **The release switch is atomic.** Each release unpacks into its own directory and the install
  path is moved onto it by renaming a symlink, so a restart can never observe a half-copied
  install — which a copy-in-place release can.
- **Readiness has to hold, not merely appear.** A single sample can pass in the window before a
  service's first real self-check has run, so the gate re-probes after a wait. Services with slower
  self-checks raise that wait rather than weakening the gate.
- **Rollback is decided on the host.** Release, health check and rollback are one remote script, so
  a runner that dies between restarting and checking cannot leave a broken release serving with
  nothing left to react to it. Three releases are retained, which is what makes the rollback target
  still there to point at.

A service needing remote work beyond the unit sync provides `deploy/post-release.sh`, run after the
sync and before the release switch. It is a file in the consuming repository rather than a workflow
input, so the work stays reviewable in the repository it belongs to and the shared workflow never
gains a way to inject arbitrary commands.

## Releases

`release.yml` publishes a release when a version tag is pushed. It does not create the tag: cutting
a version stays deliberate.

It verifies the tagged commit before publishing it. A release names a commit as tested, and an
immutable version is the wrong place to discover that it was not.

`artifact-paths` is optional, and empty is the common case. The libraries here are built from the
tag by the consuming build service, so attaching a second copy of the artifact would publish two
things that could disagree. Where files are attached they are staged together and checksummed into
`SHA256SUMS.txt`.

This is the only workflow here holding `contents: write`, the widest grant in the repository, which
is why its actions are pinned to commit SHAs like everything else — whoever can move a mutable tag
would otherwise decide what runs against that token.

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

`deploy.yml` callers pass only what genuinely differs between services, and serialise on a shared
concurrency group so two releases never land at once:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:

# One production deploy at a time; never cancel a release mid-flight.
concurrency:
  group: deploy-production
  cancel-in-progress: false

jobs:
  deploy:
    uses: damian1000/.github/.github/workflows/deploy.yml@main
    with:
      app: risk-engine
      dist-module: app
      url: https://risk.damianhoward.com
      health-port: "8081"
    secrets: inherit
```

`dist-module` is empty for a single-module root project. It is one input rather than a separate
task and directory, which could be set inconsistently. `hold-seconds` defaults to 15 and is raised
by services whose readiness genuinely takes longer to settle.

`release.yml` callers are triggered by their own tag push and must widen the token grant
themselves — the default is read-only, and a reusable workflow cannot request more than its caller
holds:

```yaml
name: Release
on:
  push:
    tags:
      - "v*.*.*"

permissions:
  contents: write

jobs:
  release:
    uses: damian1000/.github/.github/workflows/release.yml@main
    with:
      build-command: clean build
```

`automerge.yml` replaces each repository's `dependabot-automerge.yml`:

```yaml
name: Auto-merge
on:
  pull_request_target:
    branches: [main]
jobs:
  auto-merge:
    uses: damian1000/.github/.github/workflows/automerge.yml@main
    secrets: inherit
```
