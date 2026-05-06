# Releasing

This project uses [release-please](https://github.com/googleapis/release-please) to automate version bumps, changelog generation, and GitHub Release creation. Two release tracks are supported:

- `master` → stable releases (config: `release-please-config.json`, manifest: `.release-please-manifest.json`)
- `beta` → beta releases (config: `release-please-config-beta.json`, manifest: `.release-please-manifest-beta.json`)

## Workflows

| Workflow      | Trigger                          | Purpose                                |
|---------------|----------------------------------|----------------------------------------|
| `ci.yml`      | `pull_request`, `push` master/beta | Lint always; build only on PRs.        |
| `_lint.yml`   | reusable                         | `golangci-lint` over the whole repo.   |
| `_build.yml`  | reusable (accepts `ref` input)   | Build CLI (`mage cross`) + Android AAR (`mage mobile`). Uploads `cli-binaries` and `android-aar` artifacts. |
| `release.yml` | `push` master/beta, `workflow_dispatch` | Run release-please. On a release commit, build and publish artifacts to the GitHub Release. |

## Standard release flow

1. Land conventional-commit changes (`feat:`, `fix:`, `feat!:` for breaking) into `master` or `beta`.
2. On each push, `release.yml` runs `release-please`. It opens or updates a **Release PR** titled `chore(<branch>): release X.Y.Z` containing the version bump and changelog.
3. Continue merging more `feat:`/`fix:` PRs — the Release PR is amended automatically.
4. When ready to ship, merge the Release PR. On that push:
   - `release-please` creates the git tag and an empty GitHub Release.
   - `build` job compiles CLI + Android AAR.
   - `publish` job downloads artifacts, generates `SHA256SUMS`, and uploads everything to the Release.

Commit types that do not bump the version (e.g. `ci:`, `chore:`, `docs:`, `refactor:`) will not trigger a release.

## Manual rebuild for an existing tag

If `release.yml` finishes `release-please` but `build` or `publish` fail (GitHub API timeout, runner outage, etc.), the tag and Release exist without artifacts. Re-running `release-please` will not help — it considers the version released and produces no outputs.

Recover by dispatching `release.yml` with the `tag` input:

```sh
gh workflow run release.yml -r <branch> --field tag=v1.0.1
```

- `<branch>` can be any branch that contains the workflow file (typically the same branch the release was cut from).
- The `release-please` job is skipped when `tag` is set.
- `build` checks out the tag (`actions/checkout` with `ref: ${{ inputs.tag }}`) and produces artifacts.
- `publish` uploads to the existing Release with `--clobber`, so re-running is safe and idempotent.

## Notes

- Default workflow `permissions: {}` — only the jobs that need write access (`release-please`, `publish`) declare it explicitly.
- `SHA256SUMS` is generated over `dist/cli/*` and `dist/android/*` and uploaded alongside the binaries.
- `_build.yml` globs `build/olcrtc*` for the Android upload, capturing both `olcrtc.aar` and `olcrtc-sources.jar`.
- Concurrency on `ci.yml` cancels in-progress runs for the same ref to save minutes.
