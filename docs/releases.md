# Releases and crates.io publishing

The release workflow checks that the tag matches the package versions, generates
release notes from Conventional Commits, creates a GitHub release, and publishes
the workspace's publishable crates.

Release-note generation expects a `.convco` repository under the calling
repository's owner, containing `.versionrc`.

## Set up Trusted Publishing

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

## Configure the calling workflow

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

## Migrate from an API token

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

## Protect releases

Consider adding required reviewers and deployment restrictions to the `release`
environment. For the tag-triggered example above, allow the appropriate release
tags, such as `v*`; a restriction allowing only branch `main` would not match those
tag deployments.
