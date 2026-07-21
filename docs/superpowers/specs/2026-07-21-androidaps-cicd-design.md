# AndroidAPS fork CI/CD design

**Date:** 2026-07-21
**Repo:** `funman300/AndroidAPS-old-fork` (fork of `nightscout/AndroidAPS`)
**Status:** approved, pending implementation

## Context

This fork already carries a mature release pipeline in `.github/workflows/`:

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `aaps-ci.yml` | dispatch | Build any upstream tag, JDK chosen from `jdk-map.json` |
| `branch-ci.yml` | dispatch | Build the current branch, any variant |
| `pr-ci.yml` | dispatch | Build an arbitrary upstream PR |
| `cherry-pick-ci.yml` | dispatch | Build with upstream commits cherry-picked |
| `auto-release.yml` | dispatch / from sync | Signed APKs published as Gitea releases for Obtainium |
| `sync-upstream.yml` | daily 06:00 UTC | Merge upstream, trigger releases for new tags |
| `keystore-export.yml` | dispatch | Export the signing keystore |
| `cleanup-workflow-runs.yml` | `workflow_call` | Prune old runs |

The gap is not release automation — it is **verification**. Every build workflow
above is `workflow_dispatch`-only. Nothing runs automatically on a commit.

The practical consequence: `sync-upstream.yml` merges upstream into `master` and
pushes it every day, unattended. If that merge produces code that does not
compile or breaks a unit test, nothing notices until someone manually dispatches
a build — potentially days later, with several merges stacked on top of the
breakage.

The repo has exactly one branch, `master`, and it is public. Upstream is
`nightscout/AndroidAPS`. The root `build.gradle.kts` applies
`org.jlleitschuh.gradle.ktlint` (v14.0.1) to all projects, so a `ktlintCheck`
task already exists. `runtests.sh` is
`./gradlew -Pcoverage -PfirebaseDisable testFullDebugUnitTest`.

## Goals

- Fail loudly, automatically, when a commit or an upstream sync breaks the build.
- Reuse the tooling already in the tree rather than adding new static analysis.
- Verify the APK is correctly signed before it is published to Obtainium users.
- Make release notes carry enough information to spot an anomalous build.

## Non-goals

- Detekt or any static analyser not already applied in the build.
- Instrumented / device tests.
- Migrating any workflow to Gitea Actions. Releases already land on Gitea; builds
  stay on GitHub Actions, which is free for this public repo.
- Restructuring the existing dispatch-driven build workflows.

## Design

### 1. `.github/workflows/verify.yml` (new)

Triggers: `push` to `master`, `pull_request`, and `workflow_call` (so
`sync-upstream.yml` can invoke it). Concurrency group per ref with
`cancel-in-progress: true`.

Every job shares the same preamble, matching what `branch-ci.yml` already does so
the fork has one way of setting up a build:

- `actions/checkout@v5`
- `JAVA_VERSION=$(jq -r '.default' .github/jdk-map.json)` into `$GITHUB_ENV`
- `actions/setup-java@v5` with `distribution: temurin`, `java-version:
  ${{ env.JAVA_VERSION }}`, `cache: gradle`
- `chmod +x gradlew`

Deliberately reusing `actions/setup-java`'s `cache: gradle` rather than
introducing `gradle/actions/setup-gradle`: a second caching mechanism in the same
repo would be a divergence with no benefit here.

Gradle invocations carry the same JVM tuning `branch-ci.yml` uses
(`-Dorg.gradle.jvmargs="-Xmx8g -XX:+UseParallelGC -Xss1024m"`,
`-Dkotlin.daemon.jvm.options="-Xmx2g"`,
`-Dkotlin.compiler.execution.strategy="in-process"`), since that configuration is
already known to fit a GitHub-hosted runner for this project.

Three parallel jobs:

| Job | Command |
| --- | --- |
| `ktlint` | `./gradlew ktlintCheck` |
| `unit-tests` | `./gradlew -PfirebaseDisable testFullDebugUnitTest` |
| `assemble` | `./gradlew -PfirebaseDisable :app:assembleFullDebug :wear:assembleFullDebug` |

`unit-tests` uploads `**/build/reports/tests/**` as an artifact with
`if: failure()`. `assemble` does not upload the APK — `branch-ci.yml` already
covers producing an installable build on demand.

No job touches `secrets.KEYSTORE_SET` or any other secret. Debug variants are
signed with the standard debug key, so `pull_request` runs from forks are safe by
construction and need no `pull_request_target` workaround.

**Open question, to be resolved on the first run rather than guessed:** whether
upstream keeps `master` ktlint-clean. If `ktlintCheck` returns a wall of
pre-existing violations, the job will be narrowed to files changed in the
push/PR instead of committing formatting churn across a fork that must stay
mergeable with upstream. The full-repo check is the starting assumption because
upstream applies the same plugin.

### 2. `.github/workflows/sync-upstream.yml` (modified)

The daily sync gains verification of what it just pushed.

- The `sync` job gains an output `changed` — `true` when the merge actually moved
  `master`, `false` on a no-op sync.
- A new job `verify` with `needs: sync`, `if: needs.sync.outputs.changed == 'true'`,
  `uses: ./.github/workflows/verify.yml`.
- A new job `report-failure` with `needs: [sync, verify]`, `if: failure()`, which
  opens an issue titled `Upstream sync broke the build` containing the merged
  upstream range and a link to the failed run. If an open issue with that title
  already exists, it comments on it instead of opening a duplicate — consecutive
  daily syncs against a still-broken `master` must not produce a new issue every
  morning.

This is the whole of "post-sync verification": one call into `verify.yml`, not a
second parallel build workflow.

The existing "detect new upstream tags and trigger release builds" step is
untouched. Note that it dispatches `auto-release.yml` for a new upstream *tag*,
which builds that tag from upstream rather than the merged fork `master`, so
gating it on `verify` would be gating the wrong thing.

### 3. `.github/workflows/auto-release.yml` (modified)

Two additions between the existing "Rename APKs with version" and "Create Gitea
release and upload APKs" steps.

**Verify the signature before publishing.** A new step resolves the newest
installed build-tools directory
(`APKSIGNER=$ANDROID_HOME/build-tools/$(ls "$ANDROID_HOME/build-tools" | sort -V | tail -1)/apksigner`,
failing if that path is not executable) and runs `apksigner verify --print-certs`
on every APK about to be uploaded, failing the job unless:

- the APK verifies, and
- the signing certificate SHA-256 matches a repository *variable*
  `RELEASE_CERT_SHA256`.

If `RELEASE_CERT_SHA256` is unset, the step prints the observed fingerprint and
fails with instructions to set it. This is a one-time setup cost and is the point
of the check: Obtainium refuses to update an app whose signing key changed, so
publishing an APK signed with the wrong key silently strands every installed
client. The variable is not a secret — a public certificate fingerprint.

**Enrich the release body.** The release body today is a single sentence. It
becomes a short table listing, per APK: filename, size, size delta against the
same-named asset on the previous Gitea release (fetched via
`GET $BASE/releases?limit=1` before creating the new one; rendered as `n/a` when
there is no prior release or no matching asset), and SHA-256. Retains the
existing "Automated build of AAPS `<version>` from upstream nightscout/AndroidAPS"
line above the table.

An unexplained size jump is the cheapest available signal that a build picked up
something it should not have.

## Testing

- `verify.yml`: dispatched manually first via a temporary `workflow_dispatch`
  trigger to observe real runtimes and settle the ktlint question, then wired to
  `push`/`pull_request`. Correctness of the failure path is checked with a
  throwaway branch containing a deliberate compile error and a deliberately
  failing assertion, confirming `assemble` and `unit-tests` fail independently
  and the test report artifact uploads.
- `sync-upstream.yml`: dispatched manually. Verified that a no-op sync skips the
  `verify` job, and that a sync with changes runs it. The `report-failure` path
  is checked by pointing a scratch copy of the workflow at a branch known not to
  build, confirming one issue is opened and a second run comments rather than
  duplicating.
- `auto-release.yml`: dispatched for an already-released upstream tag with the
  Gitea publish step temporarily disabled, confirming the signature check passes
  against the real keystore, that it fails when `RELEASE_CERT_SHA256` is set to a
  wrong value, and that the rendered release body is correct including the `n/a`
  no-prior-release case.

## Risks

- **Runtime and cost.** Three parallel jobs on every push to a repo that receives
  a daily upstream merge. The repo is public, so GitHub Actions minutes are free;
  the cost is wall-clock feedback time only.
- **ktlint on upstream code.** Covered above — narrow to changed files if the
  full check is not clean on arrival.
- **Cache thrash.** A daily upstream merge invalidates much of the Gradle cache,
  so post-sync runs will be slow. Accepted; it is the run that most needs to
  happen.
- **`firebaseDisable`.** Assumed sufficient to build without `google-services.json`,
  since `runtests.sh` relies on it. If `assembleFullDebug` still requires the
  file, the `assemble` job will write a stub `google-services.json` the way
  upstream CI does, rather than putting a real one in secrets.
