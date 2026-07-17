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
| `fix:` | patch (`0.1.x`) | Bug fix, security patch, or upgrading, downgrading, or pinning an existing package's version |
| `feat:` | minor (`0.x.0`) | Adding or removing a package, tool, or capability from the image |
| `feat!:` or a `BREAKING CHANGE:` footer | major (`x.0.0`) | Base image swap. This is the *only* case that qualifies for a major bump — nothing else in this repo does. |

release-please only reads the commit message's type prefix — it never inspects the diff, so *how* a package was installed doesn't affect its classification. Adding a package via `apt.txt`, `environment.yml`, `requirements.txt`, or by building it from source in a Dockerfile `RUN` step are all equally "adding a package" (`feat:`, minor). Don't confuse "building a tool from its source repository" with a base image swap — only changing the Dockerfile's `FROM` line qualifies for `feat!:`.

Other changes to `geolab-base/Dockerfile` that are neither a package change nor a base image swap — e.g. editing `ENV`/`LABEL`/`ARG` defaults, `CMD`, or the `start` entrypoint script — aren't explicitly covered by the table above either. Classify these by what they actually do: a bug fix or behavior correction is `fix:`, a new capability is `feat:`. There's no separate rule for "Dockerfile plumbing."

### Types that do not affect versioning

`chore:`, `docs:`, `ci:`, `refactor:`, `test:`, `style:` These are recorded in git history but do not trigger a version bump or changelog entry.

### Examples for this repo

```text
feat(geolab-base): add scipy to environment.yml

feat(geolab-base): add nodejs and npm via apt.txt

feat(geolab-base): build and install libmseed from source

feat(geolab-base): remove seisbench due to failed dependency resolution

fix(geolab-base): pin numpy to 1.26.4 to fix build failure

fix(geolab-base): upgrade obspy to 1.5.0 for a bugfix upstream

fix(geolab-base): downgrade pyarrow to 17.0.0 to unblock the build

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

### Reverting a change

`git revert` produces a commit message like `Revert "feat(geolab-base): add jq via apt.txt"`. That's not valid Conventional Commits syntax, and release-please's parser can't read it — it's silently dropped with no error. In practice this means reverting a `feat:`/`fix:` commit does **not** undo its effect: the original commit is still counted in full, so the next release still bumps the version and still lists the reverted change in `CHANGELOG.md`, as if it had shipped.

Rewriting the revert with valid syntax (e.g. `revert(geolab-base): add jq via apt.txt`, with a `This reverts commit <sha>.` footer) doesn't fix this either. release-please recognizes `revert:` as its own type with its own `### Reverts` changelog section, but that does not cancel out or remove the original commit's entry or version bump — the changelog ends up showing *both* the original change and its revert, and the version still bumps as though the original change stuck.

**release-please has no built-in way to treat a revert as a net no-op.** If a change needs to be fully retracted before its effect on versioning/the changelog:

- **Best, if the commit hasn't been released yet:** remove it from history (`git rebase`/`reset`) before it's ever picked up into a Release PR, rather than reverting it forward.
- **If it's already in an open, unmerged Release PR:** edit that PR's branch directly to drop the change, instead of adding a revert commit to `main`.
- **If it's already been released:** don't rely on `git revert`'s default message at all — write the undo as an ordinary new commit (`fix:`/`feat:` as appropriate to what the undo actually does).

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

6. **The GitHub Release publish event triggers the build/push pipeline.** `build-push.yml` listens for `release: types: [published]`, checks out the repo at that release, reads the version straight out of `geolab-base/VERSION` (which release-please just wrote as part of the merged Release PR), and tags the image with exactly that value — e.g. `1.3.0`. Reading the file directly instead of re-deriving the version from the git tag name is what guarantees the image tag and the package release are always the same version.

## What release-please does not see

release-please is driven entirely by git commit messages. It never inspects a diff, a GitHub repo setting, a secret, or anything else that isn't a git commit. Two consequences of that are easy to miss:

**Only commits that touch `geolab-base/` count.** This repo's package path is `geolab-base` (see `release-please-config.json`). A commit is only considered for versioning/changelog purposes if its diff touches a file under `geolab-base/` — regardless of its type prefix. A `fix:`/`feat:`/`feat!:` commit that only touches `.github/workflows/*.yml`, `release-please-config.json`, root `README.md`, `.gitlab-ci.yml`, `CODEOWNERS`, or `docs/*.md` has zero effect on the version; it's silently excluded, the same as a `chore:` commit would be. Verified directly against this repo: a `fix(build-push): ...` commit that only touched `build-push.yml` was skipped with "No user facing commits found." If you're changing CI/release automation rather than the image, `chore:`/`ci:`/`docs:` is still the right type — just don't expect a `fix:`/`feat:` on those files to bump anything either way.

**Squash-merging would break this model.** release-please reads whatever commits actually exist in `main`'s history. If a PR is squash-merged, only the single squashed commit's message is ever seen — the individual commits inside that PR are discarded from history entirely. This repo doesn't currently squash-merge feature work, but if that changes, it's the squash commit's message (often defaulted to the PR title) that needs to follow Conventional Commits, not the commits inside the PR.

More generally: everything past the type prefix is decorative to release-please. It doesn't know or care whether a commit added a package or rewrote the whole Dockerfile — the type you choose is the only thing that matters. See [Reverting a change](#reverting-a-change) above for how far this goes: a revert doesn't cancel anything out, because release-please has no notion of "this undoes that."

## One-time setup required

To enable this process, the repository needs:

- A `release-please-config.json` at the repo root configuring `release-type: simple` (appropriate for a non-npm non-language-specific repos) and pointing at the version file to track.
- A `.release-please-manifest.json` tracking the current version per path.
- A new workflow, e.g. `.github/workflows/release-please.yml`, running the `googleapis/release-please-action` on push to `main`.