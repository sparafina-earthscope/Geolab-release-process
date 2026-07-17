# Release Process: Conventional Commits + release-please

This document describes how semantic versioning and changelog generation work for this repository, using [Conventional Commits](https://www.conventionalcommits.org/) and [release-please](https://github.com/googleapis/release-please).

## Why this approach

Version bumps and changelog entries are derived automatically from commit messages instead of being written by hand or guessed from a diff. The version number and the changelog entry both come from what a contributor typed at commit time, so changelog entries describe the actual change instead of a generic placeholder.

## Conventional Commits

Every commit that should affect the version or changelog follows this format:

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types that affect versioning

| Type | Version bump | When to use it |
|---|---|---|
| `fix:` | patch (`0.1.x`) | Bug fix, security patch, or version pin for a package |
| `feat:` | minor (`0.x.0`) | Adding or removing a package, tool, or capability from the image |
| `feat!:` or a `BREAKING CHANGE:` footer | major (`x.0.0`) | Base image swap, or any other change that breaks compatibility |

### Types that do not affect versioning

`chore:`, `docs:`, `ci:`, `refactor:`, `test:`, `style:` These are recorded in git history but do not trigger a version bump or changelog entry.

### Examples for this repo

```text
feat(geolab-base): add scipy to environment.yml

feat(geolab-base): remove seisbench due to failed dependency resolution

fix(geolab-base): pin numpy to 1.26.4 to fix build failure

feat!: bump base image from python:3.11 to python:3.12

BREAKING CHANGE: base image major version changed; downstream images must rebuild

chore: update .gitignore
```

### Scope

The optional `(scope)` names the part of the repo affected, e.g. `(geolab-base)`, `(build-push)`. It's for readability in the changelog and does not affect the version bump logic.

### Making commits with the git CLI

A single-line `fix:` or `feat:` commit needs nothing special:

```bash
git commit -m "fix(geolab-base): pin numpy to 1.26.4 to fix build failure"

git commit -m "feat(geolab-base): add scipy to environment.yml"
```

For a commit with a body (extra explanation) but no footer, stack `-m` flags, the first becomes the subject line, each additional one becomes its own paragraph in the body:

```bash
git commit -m "fix(geolab-base): pin numpy to 1.26.4" \
  -m "The 1.26.5 release broke the build on arm64 runners."
```

For a **breaking change**, add a `BREAKING CHANGE:` footer as its own `-m` paragraph, and mark the type with `!`:

```bash
git commit -m "feat!: bump base image from python:3.11 to python:3.12" \
  -m "BREAKING CHANGE: base image major version changed; downstream images must rebuild"
```

For longer messages, skip `-m` and let git open your editor (respects `core.editor` / `$EDITOR`), which is easier for multi-paragraph bodies and footers:

```bash
git commit
```

```text
feat(geolab-base): add scipy to environment.yml

Needed for the gnss notebooks added in this PR.

Refs: CRO-431
```

A `git commit --amend` works the same way if you need to fix a type/scope on the most recent commit before pushing. Re-run `git commit --amend` and edit the message.

## Step-by-step: how release-please works

1. **Contributor commits using Conventional Commits.** Every PR merged to `main` contains commits following the format above.

2. **release-please runs on every push to `main`.** It scans all commits since the last release tag and classifies them by type.

3. **release-please opens or updates a "Release PR".** This PR's diff is just two things:
   - The version file (e.g. `VERSION`) bumped to the next semantic version.
   - `CHANGELOG.md` with a new section listing every `feat:`/`fix:`/breaking
     commit since the last release, grouped and described using the commit
     messages themselves.

No new code changes are introduced by this PR. It only proposes the version/changelog update. It stays open and keeps updating itself as more commits land on `main`, until someone merges it.

4. **A maintainer reviews and merges the Release PR** when ready to cut a release. This is the one manual/human step in the process, nothing is auto-committed to `main` directly.

5. **On merge, release-please tags the release.** It creates a git tag (e.g. `v1.3.0`) and a corresponding GitHub Release with the changelog entry as the release notes.

6. **The GitHub Release publish event triggers the build/push pipeline.** `build-push.yml` already listens for `release: types: [published]` and already tags Docker images using `type=semver,pattern={{version}}` and `type=semver,pattern={{major}}.{{minor}}` in its `docker/metadata-action` step. Once step 5 happens, this fires automatically and the image gets published as `1.3.0`, `1.3`, and `latest` (for the default branch). No changes needed to the existing Docker build/push logic.

## One-time setup required

To enable this process, the repository needs:

- A `release-please-config.json` at the repo root configuring `release-type: simple` (appropriate for a non-npm non-language-specific repos) and pointing at the version file to track.
- A `.release-please-manifest.json` tracking the current version per path.
- A new workflow, e.g. `.github/workflows/release-please.yml`, running the `googleapis/release-please-action` on push to `main`.