# elastic-package workflows

Reusable workflows that lint, test and release an Elastic integration package,
built on [`NateShoffner/elastic-package-action`][action]. Both use the automatic
`GITHUB_TOKEN`, so no secrets.

## CI

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 6 * * 1'
  workflow_dispatch:
    inputs:
      stack-tests:
        description: Also run the tests that need a full stack (slow)
        type: boolean
        default: false

permissions:
  contents: read

jobs:
  ci:
    uses: NateShoffner/elastic-package-workflows/.github/workflows/ci.yml@v1
    with:
      package: packages/my_package
      stack-tests: ${{ github.event_name == 'schedule' || inputs.stack-tests == true }}
```

**Triggers stay in the caller.** A reusable workflow cannot declare its own
`on:`, so the schedule and any dispatch inputs live in your workflow, and the
decision reaches this one through `stack-tests`. Write `inputs.stack-tests ==
true`: the bare reference is null on events that carry no dispatch inputs.

| Input | Default | Description |
| --- | --- | --- |
| `package` | *(required)* | Package directory from the repository root. `.` if it is the root. |
| `stack-tests` | `false` | Also run the tests that need a full stack. Slow. |
| `runs-on` | `ubuntu-24.04` | Runner label. |
| `elastic-package-version` | `v0.126.4` | Release to install. |

Outputs `test-types` and `package-artifact`.

## Release

Verifies the tag against `manifest.yml`, runs the tests, builds the zip, attests
it with [`actions/attest-build-provenance`][attest] and publishes a release with
notes from `changelog.yml`. A test job that was never scheduled does not block
the release; one that ran and failed does.

```yaml
name: Release

on:
  push:
    tags: ['v*']

# A called workflow cannot request more than its caller grants.
permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  release:
    uses: NateShoffner/elastic-package-workflows/.github/workflows/release.yml@v1
    with:
      package: packages/my_package
```

`on.push.tags` and `tag-prefix` have to agree; for unprefixed tags use
`tags: ['[0-9]*']` with `tag-prefix: ''`.

| Input | Default | Description |
| --- | --- | --- |
| `package` | *(required)* | Package directory from the repository root. |
| `tag-prefix` | `v` | Stripped from the tag to get the version. `''` for unprefixed tags. |
| `stack-tests` | `true` | Gate the release on the full-stack tests. |
| `runs-on` | `ubuntu-24.04` | Runner label. |
| `elastic-package-version` | `v0.126.4` | Release to install. |
| `tag` | *(triggering ref)* | Tag to release. |
| `dry-run` | `false` | Build and render notes, but publish nothing. |
| `attest` | `true` | Attach a provenance attestation. Off where attestations are unavailable. |

Outputs `version`.

## Test types

`detect` scans for `_dev/test/<type>` directories, at package level and under
`data_stream/*/`, and runs what it finds, so adding policy tests to a package
needs no workflow change. `pipeline` and `static` run in a job that boots
Elasticsearch alone or nothing at all; `asset`, `policy`, `script` and `system`
run in one that boots the full stack behind `stack-tests`. Each job boots once
and continues past a failure, so a run reports every broken type. A directory
that is not an `elastic-package test` subcommand warns instead of becoming an
unknown subcommand.

The full-stack tier outgrows a private repository's 2-core runner, where
Elasticsearch fails its healthcheck before Kibana starts. Point `runs-on` at a
larger one.

There is deliberately no `data-streams` input. `elastic-package` finds system
tests per data stream, and a stream that cannot be tested opts out in its own
config, reported as skipped rather than failed:

```yaml
skip:
  reason: No docker-based mock exists for the aws-s3 input.
  link: https://github.com/owner/repo/issues/42
```

## Notes

- Pin to `@v1`, not `@main`.
- Built zips land in `build/packages/` at the repository root, not in the
  package directory, because that is where `elastic-package` writes them.
- Third-party actions are SHA-pinned; `elastic-package-action` stays on `@v1`.
- `self-test.yml` runs both workflows against the fixture in `tests/`, and needs
  default workflow permissions set to write.
- The two workflows duplicate their test jobs: a reusable workflow runs against
  the caller's checkout, so there is no shared file to factor the steps into.

[action]: https://github.com/NateShoffner/elastic-package-action
[attest]: https://github.com/actions/attest-build-provenance
