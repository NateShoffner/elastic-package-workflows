# elastic-package workflows

Reusable GitHub Actions workflows that lint, test and release a single Elastic
integration package, built on
[`NateShoffner/elastic-package-action`][action].

| Workflow | Reference |
| --- | --- |
| CI | `NateShoffner/elastic-package-workflows/.github/workflows/ci.yml@v1` |
| Release | `NateShoffner/elastic-package-workflows/.github/workflows/release.yml@v1` |

Both use the automatic `GITHUB_TOKEN` only, so no `secrets: inherit`.

## CI

Lints and builds the package, then runs whichever test types it defines.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    # Weekly, so the slow tests still catch upstream stack drift on a quiet repo.
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
      # Scheduled runs always take the slow tier; manual runs ask for it.
      stack-tests: ${{ github.event_name == 'schedule' || inputs.stack-tests == true }}
```

**The triggers stay in the caller.** A reusable workflow cannot declare its own
`on:`, so `push`, `pull_request`, `schedule` and `workflow_dispatch` all belong
in the calling workflow, along with any dispatch inputs. Nothing here reads
`github.event_name`; pass the decision in through `stack-tests`. Write
`inputs.stack-tests == true` rather than the bare reference, which evaluates to
null on events that carry no dispatch inputs.

| Input | Default | Description |
| --- | --- | --- |
| `package` | *(required)* | Path to the package directory from the repository root. Use `.` when the package is the repository root. |
| `stack-tests` | `false` | Also run the test types needing a full stack. Off by default because booting Elasticsearch, Kibana, Fleet Server and an agent takes several minutes. |
| `runs-on` | `ubuntu-24.04` | Runner label used by every job. |
| `elastic-package-version` | `v0.126.4` | `elastic-package` release to install, with or without the leading `v`. |

Outputs: `test-types` (detected types, e.g. `pipeline policy system`) and
`package-artifact` (name of the artifact holding the built zip).

## Release

Checks the tag against the manifest, runs the tests, builds the zip, attests it
with [`actions/attest-build-provenance`][attest] and publishes a release with
notes rendered from `changelog.yml`.

```yaml
name: Release

on:
  push:
    tags: ['v*']

# A called workflow cannot request more than its caller grants, so these must be
# here: contents to create the release, id-token and attestations to sign the
# provenance attestation.
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

The tag pattern in `on.push.tags` and `tag-prefix` have to agree. For unprefixed
tags, use `tags: ['[0-9]*']` and `tag-prefix: ''`.

| Input | Default | Description |
| --- | --- | --- |
| `package` | *(required)* | Path to the package directory from the repository root. |
| `tag-prefix` | `v` | Prefix stripped from the tag to get the version. Pass `''` for unprefixed tags. A tag not starting with the prefix fails the release. |
| `stack-tests` | `true` | Gate the release on the test types needing a full stack. On by default: a tagged release is rare enough to be worth the extra minutes. |
| `runs-on` | `ubuntu-24.04` | Runner label used by every job. |
| `elastic-package-version` | `v0.126.4` | `elastic-package` release to install. |
| `tag` | *(the triggering ref)* | Tag to release. Set it to release a tag other than the triggering ref, or to exercise the workflow from a branch with `dry-run`. |
| `dry-run` | `false` | Run every check, build the zip and render the notes, but create neither the release nor the attestation. |
| `attest` | `true` | Attach a provenance attestation to the zip. Turn it off where artifact attestations are unavailable, such as private repositories outside GitHub Enterprise Cloud. |

Outputs: `version`, the released version without the tag prefix.

The version is verified against `manifest.yml` before anything is published, and
a changelog with no entry for it fails the release. A test job that was never
scheduled, because the package has no tests of that tier or because
`stack-tests` is off, does not block the release; one that ran and failed does.

## How test types are detected

Nothing here carries a hardcoded list. `detect` scans the package for
`_dev/test/<type>` directories, at package level and under `data_stream/*/`, and
runs exactly what it finds, split by what each runner needs from the stack:

| Test type | Job | Stack it needs |
| --- | --- | --- |
| `static` | `light-tests` | None. |
| `pipeline` | `light-tests` | Elasticsearch only (`--services elasticsearch`). |
| `asset`, `policy`, `script`, `system` | `stack-tests` | Elasticsearch, Kibana, Fleet Server and an agent. |

So adding policy tests to a package is enough to get them running; no workflow
change is needed. Each job boots its stack once and runs every detected type
against it, continuing past a failure so one run reports every broken type. A
directory that is not one of the six `elastic-package test` subcommands is
ignored with a warning, so a typo like `_dev/test/sytem` shows up in the summary
rather than failing as an unknown subcommand.

`elastic-package` runs asset and static tests without needing a config
directory, so those two are only scheduled when the package actually has a
`_dev/test/asset` or `_dev/test/static` directory.

### Data streams that cannot be system tested

There is deliberately no `data-streams` input. `elastic-package` discovers system
tests from `data_stream/<stream>/_dev/test/system/test-*-config.yml`: a data
stream with no such file has no system tests, and one that has a file can opt out
explicitly, reported as skipped rather than failed.

```yaml
skip:
  reason: No docker-based mock exists for the aws-s3 input.
  link: https://github.com/owner/repo/issues/42
```

For a package like `backblaze_b2`, where only the `bucket` data stream has a
mock, `elastic-package test system` already does the right thing across the whole
package, and the reason travels with the data stream instead of a workflow input.

## Self-test

`.github/workflows/self-test.yml` runs both workflows against
`tests/fixtures/packages/fixture_logs`, a minimal package kept here, and asserts
on their outputs. The fixture has real pipeline tests and a system test skipped
with a reason, which exercises the full-stack tier without shipping a mock
service. The release leg runs with `dry-run: true`, so `gh release create` and
the attestation are the two steps it does not cover. It needs the repository's
default workflow permissions to allow writes.

## Notes

- Pin to a tag (`@v1`) rather than `@main`, so a change here cannot break a
  release in an integration repository without a deliberate bump.
- Third-party actions are pinned to full commit SHAs with the version in a
  trailing comment, since other repositories execute these workflows.
  `NateShoffner/elastic-package-action` stays on `@v1`, which it also documents
  as its own reference.
- Built zips land in `build/packages/` at the root of the repository checkout,
  not inside the package directory. That is where `elastic-package` puts them,
  and why the artifact and attestation paths look the way they do.
- The two workflows duplicate their test jobs rather than sharing them: a
  reusable workflow runs against the *caller's* checkout, so there is no shared
  file to factor the steps into, and chaining `release.yml` to `ci.yml` through a
  relative path resolves differently depending on who is calling.

[action]: https://github.com/NateShoffner/elastic-package-action
[attest]: https://github.com/actions/attest-build-provenance
