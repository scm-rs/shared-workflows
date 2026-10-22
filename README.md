# SCM-RS shared workflows

Reusable GitHub Actions workflows for SCM-RS projects.

## Releases and crates.io publishing

The release workflow checks that the tag matches the package versions, generates
release notes from Conventional Commits, creates a GitHub release, and publishes
the workspace's publishable crates.

Release-note generation expects a `.convco` repository under the calling
repository's owner, containing `.versionrc`.

### Set up Trusted Publishing

Trusted Publishing lets GitHub Actions authenticate to crates.io through OIDC
without storing a long-lived crates.io API token.

For each crate published by the workflow:

1. The crate must already exist on crates.io. Its initial publication requires an
   API token.
2. As a crate owner, open the crate's **Settings → Trusted Publishing** and add a
   GitHub publisher:

   | Field | Value |
   |---|---|
   | Repository owner | The owner of the repository calling this workflow, for example `scm-rs` |
   | Repository name | The calling repository's name |
   | Workflow filename | The caller's release workflow filename, normally `release.yaml` |
   | Environment | `release` |

   Use the **calling repository**, not `shared-workflows`. Enter the filename
   only—not `.github/workflows/release.yaml`.

3. In the calling GitHub repository, create an environment named **`release`**, if
   it does not already exist.

For a workspace publishing multiple crates, configure a Trusted Publisher for each
crate.

### Configure the calling workflow

For a crate-only repository with an existing `ci.yaml` workflow, use:

```yaml
name: release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write
  id-token: write

jobs:
  ci:
    uses: ./.github/workflows/ci.yaml

  release:
    needs:
      - ci
    uses: scm-rs/shared-workflows/.github/workflows/release.yaml@main
    # no CARGO_REGISTRY_TOKEN: publishing to crates.io uses trusted publishing
    # (OIDC) via the "release" environment. Passing the token would take
    # precedence in the shared workflow and bypass it.
```

`contents: write` permits creating the GitHub release. `id-token: write` permits
OIDC authentication.

Do not forward `CARGO_REGISTRY_TOKEN` or use `secrets: inherit` for Trusted
Publishing. Also ensure the `release` environment does not provide a secret with
that name: a supplied token takes precedence over OIDC.

### Migrate from an API token

1. Configure the crates.io Trusted Publisher and GitHub environment.
2. Remove `CARGO_REGISTRY_TOKEN` forwarding from the calling workflow.
3. Merge the change before creating the next normal version tag.
4. Verify that:
   - **Authenticate with crates.io (trusted publishing)** succeeds rather than
     being skipped.
   - **Publish** succeeds.
   - The new version is available on crates.io.

   A GitHub release alone does not confirm registry publication: it is created
   before the publish job.

5. After successful verification, enable **Require trusted publishing for all new
   versions** in the crate's settings. This rejects publishing through ordinary
   API tokens.
6. Remove obsolete GitHub token secrets and revoke the old API token on crates.io.

Deleting a GitHub secret does **not** revoke the underlying token. Likewise,
requiring Trusted Publishing does not revoke it. If a token is shared with other
projects, migrate those projects before revoking it.

Do not reuse or move an existing release tag to test the migration.

### Protect releases

Consider adding required reviewers and deployment restrictions to the `release`
environment. For the tag-triggered example above, allow the appropriate release
tags, such as `v*`; a restriction allowing only branch `main` would not match those
tag deployments.

## Renovate dependency proposals

The reusable Renovate workflow provides shared defaults for Cargo libraries with
one root `Cargo.toml`. It proposes dependency updates; it never approves or merges
pull requests. Existing dependency CI, MSRV, features, review and merge-queue
requirements remain in place.

### Use the defaults

Add `.github/workflows/renovate.yaml` to the calling repository:

```yaml
name: Renovate

on:
  workflow_dispatch:
  schedule:
    - cron: '17 3 * * 1'
  pull_request:
    paths:
      - renovate.json
      - .github/workflows/renovate.yaml

permissions: {}

jobs:
  renovate:
    uses: scm-rs/shared-workflows/.github/workflows/renovate.yaml@main
    permissions:
      contents: write
      pull-requests: write
      issues: write
      statuses: write
      checks: read
```

No local Renovate configuration or forwarded secrets are needed for the defaults.
The workflow uses the calling repository's scoped `GITHUB_TOKEN`, explicitly
checks out this public repository for `.github/renovate-runner.json`, and targets
only the caller. It is not an organization-wide writer.

The caller grants the maximum permissions needed for production; the shared PR
validation job reduces these to `contents: read` and passes no token into its
read-only validator containers. Production runs only on the caller's default
branch for the Monday 03:17 UTC schedule or a manual dispatch. Landing the caller
workflow on the default branch enables those runs; there is no separate activation
variable. `@main` and the shared configuration checkout deliberately follow
`main`, like the other shared workflows: reviewed central changes affect future
runs in enrolled repositories. They are not an immutable release snapshot.

Shared defaults include:

- Cargo-only updates for the root manifest; nested manifests are disabled.
- Range-aware proposals, no automerge, and no standalone lockfile maintenance.
  Normal updates can still update a tracked `Cargo.lock`.
- A Dependency Dashboard, a three-day minimum release age, and limits of two open
  and one new PR per hour.
- Individual root dependency updates that do not re-enable held dependencies,
  with a warning for potentially breaking pre-1.0 minor changes.
- The same digest-pinned Renovate image for production and strict validation.
  Scripts, plugins, custom post-upgrade commands and automatic discovery are off.

This default is not complete coverage for virtual workspaces or nested packages.
Review those layouts before enrolling them; do not blindly re-enable dependencies
held by existing configuration.

### Add repository-specific configuration

An optional root `renovate.json` overrides shared defaults using Renovate's native
configuration merging. Keep local dependency holds, exclusions and compatibility
notes there. No `extends` entry or copy of the runner configuration is needed.
The shared safety settings still prevent automerge, non-default base branches and
standalone lockfile maintenance, and keep the `renovate-ci/` branch prefix.

For example:

```json
{
  "ignorePaths": ["schemas/**", "tests/fixtures/**"],
  "prBodyNotes": ["Review generated-model and serialization compatibility."]
}
```

List all intended paths when overriding `ignorePaths`: that array replaces the
shared value rather than appending to it. Before enrolling an existing Renovate
setup, review its complete configuration and preserve its holds. The sample PR
filter and optional-file validation cover root `renovate.json`; if maintaining a
different native configuration filename, include it in the caller's PR filter and
extend validation accordingly before adopting this integration.

Strict validation checks the shared global configuration and any root
`renovate.json`, not registry lookup coverage, dependency compatibility or writer
success. Changes to this repository's runner or workflow validate the proposed
shared configuration, not the old copy from `main`.

### Repository prerequisites and first run

Before merging a caller workflow:

1. Confirm ownership and that no competing dependency writer targets the repo.
2. Enable Issues for the Dependency Dashboard and allow the shared workflow and
   pinned actions under the repository's Actions policy.
3. Enable **Allow GitHub Actions to create and approve pull requests**. Despite
   that setting's name, this workflow does not approve or merge PRs. Default token
   permissions may remain read-only; production requests its permissions per job.
4. Ensure required checks do not depend on the path-filtered configuration job.

After merging, run **Renovate** manually on the default branch. Check the actual
Dashboard and any eligible dependency PR, not just the run's green status. A PR
is not guaranteed when no update meets the policy. For bot-created dependency PRs,
approve workflow execution when GitHub requests it, then verify actual Rust CI on
the current dependency head. Execution approval, human dependency review and the
merge queue are separate steps. If no current-head CI run is available, resolve
that before merging; a default-branch run does not validate the dependency PR.

Remove or disable the caller workflow to stop future scheduled runs. This does
not undo branches, PRs or issues already created, or necessarily cancel a run
already in flight.
