# DRAFT: GeoLab Release Playbook

> **Status: draft.** This consolidates [release-process.md](release-process.md),
> [release-process-getting-started.md](release-process-getting-started.md),
> [rollout-to-geolab.md](rollout-to-geolab.md), and
> [geolab-release-playbook.md](geolab-release-playbook.md) into one systematic reference. Nothing
> here has been re-verified beyond what those source docs already verified — this is a
> reorganization, not new research. See the end of this doc for what's still open.

## Contents

1. [Overview](#1-overview)
2. [Quick start](#2-quick-start)
3. [Conventional commit reference](#3-conventional-commit-reference)
4. [Writing commits correctly](#4-writing-commits-correctly)
5. [Reverting a change](#5-reverting-a-change)
6. [What release-please does not see](#6-what-release-please-does-not-see)
7. [How the pipeline works end-to-end](#7-how-the-pipeline-works-end-to-end)
8. [One-time setup reference](#8-one-time-setup-reference)
9. [Rolling this out to earthscope/Geolab](#9-rolling-this-out-to-earthscopegeolab)
10. [Promoting a dev release to production via GitLab](#10-promoting-a-dev-release-to-production-via-gitlab)
11. [`pangeo/base-image` update cadence](#11-pangeobase-image-update-cadence)
12. [Open questions](#12-open-questions)

## 1. Overview

This repo (`GeoLab-release-process`) is a sandbox that prototypes a full release pipeline for
`geolab-base`, intended to be rolled out to the real repo,
[earthscope/Geolab](https://github.com/earthscope/Geolab). The pipeline has two halves:

- **Dev (GitHub):** contributors write [Conventional Commits](https://www.conventionalcommits.org/).
  [release-please](https://github.com/googleapis/release-please) reads them, proposes a semantic
  version + changelog via a standing "Release PR," and — once merged — a GitHub Actions workflow
  builds and pushes a dev image to GHCR tagged with exactly that version.
- **Production (GitLab):** `earthscope/Geolab` already has a separate, pre-existing GitLab CI
  pipeline that builds and pushes to AWS ECR. This playbook does not replace it — production
  promotion is a deliberate, separate step, described in
  [§10](#10-promoting-a-dev-release-to-production-via-gitlab).

Version bumps and changelog entries are derived automatically from commit messages instead of
being written by hand or guessed from a diff — the version number and changelog entry both come
from what a contributor typed at commit time, so changelog entries describe the actual change
instead of a generic placeholder.

## 2. Quick start

For anyone who just needs to know what to type, in three sentences:

1. When you commit, your message's **type** (`fix:`, `feat:`, `feat!:`) says what kind of change
   it is — that's the only part that drives versioning.
2. release-please reads those messages and keeps a standing "Release PR" up to date with the
   next version + changelog. You never hand-edit a version number or `CHANGELOG.md`.
3. Merging that PR cuts a real release, which automatically builds and publishes the dev image
   with the matching version tag.

Cheat sheet:

```text
Fixed something, or upgraded/downgraded/pinned a package? -> fix: describe it
Added or removed a package?                               -> feat: describe what changed
Swapped the base image? (the ONLY thing that's major)      -> feat!: describe it, plus a
                                                               BREAKING CHANGE: footer
Just cleanup/docs/CI?                                       -> chore: / docs: / ci:
                                                                (won't appear in changelog)
```

Terms: **semver** = version numbers shaped `MAJOR.MINOR.PATCH`; **release-please** = the bot that
reads commits and proposes the version/changelog; **GHCR** = GitHub Container Registry, where dev
images get pushed (`ghcr.io/...`).

See [§3](#3-conventional-commit-reference) for the full rules, and [§4](#4-writing-commits-correctly)
before typing a `!` at an interactive shell prompt.

## 3. Conventional commit reference

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

release-please only reads the commit message's type prefix — it never inspects the diff, so *how*
a package was installed doesn't affect its classification. Adding a package via `apt.txt`,
`environment.yml`, `requirements.txt`, or by building it from source in a Dockerfile `RUN` step
are all equally "adding a package" (`feat:`, minor). Don't confuse "building a tool from its
source repository" with a base image swap — only changing the Dockerfile's `FROM` line qualifies
for `feat!:`.

Other changes to `geolab-base/Dockerfile` that are neither a package change nor a base image swap
— e.g. editing `ENV`/`LABEL`/`ARG` defaults, `CMD`, or the `start` entrypoint script — aren't
explicitly covered by the table above either. Classify these by what they actually do: a bug fix
or behavior correction is `fix:`, a new capability is `feat:`. There's no separate rule for
"Dockerfile plumbing."

### Types that do not affect versioning

`chore:`, `docs:`, `ci:`, `refactor:`, `test:`, `style:` — recorded in git history but don't
trigger a version bump or changelog entry.

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

The optional `(scope)` names the part of the repo affected, e.g. `(geolab-base)`, `(build-push)`.
It's for changelog readability and does not affect the version bump logic.

**Tip:** if you're not sure whether something is a `fix` or a `feat`, ask "did I add something
new, or repair something that was broken/wrong?" New = `feat`. Repaired = `fix`.

## 4. Writing commits correctly

A single-line `fix:`/`feat:` commit needs nothing special:

```bash
git commit -m "fix(geolab-base): pin numpy to 1.26.4 to fix build failure"
git commit -m "feat(geolab-base): add scipy to environment.yml"
```

For a commit with a body but no footer, stack `-m` flags — the first becomes the subject line,
each additional one becomes its own paragraph:

```bash
git commit -m "fix(geolab-base): pin numpy to 1.26.4" \
  -m "The 1.26.5 release broke the build on arm64 runners."
```

For a **breaking change**, add a `BREAKING CHANGE:` footer and mark the type with `!`:

```bash
git commit -m 'feat!: bump base image from python:3.11 to python:3.12' \
  -m "BREAKING CHANGE: base image major version changed; downstream images must rebuild"
```

### The `!` quoting gotcha

`!` triggers history expansion in interactive bash/zsh, and **double quotes don't protect against
it** — `"feat!: ..."` fails interactively with `zsh: illegal modifier:` (zsh parses `!:` as a
history-event modifier). This only bites interactive shells; it's not an issue in scripts/CI.
Ways to avoid it, in order of preference:

- **Single quotes — preferred.** `git commit -m 'feat!: ...'`. Verified safe and correct.
- **Skip `-m`, use your editor.** Plain `git commit` opens `$EDITOR`, sidestepping shell quoting
  entirely. Good fallback if you're ever unsure.
- **Don't use `\!` inside double quotes.** It avoids the shell error, but leaves a literal
  backslash in the actual commit message — verified directly: `git commit -m "feat\!: test"`
  produces the commit message `feat\!: test`, backslash and all. release-please's parser won't
  recognize that as a breaking change.
- Disabling history expansion for the whole session (`set +H` in bash, `unsetopt banghist` in
  zsh) also works, but it's a blunt, session-wide setting — not worth it for one commit.

For longer messages, skip `-m` and let git open your editor (respects `core.editor`/`$EDITOR`),
easier for multi-paragraph bodies and footers:

```bash
git commit
```
```text
feat(geolab-base): add scipy to environment.yml

Needed for the gnss notebooks added in this PR.

Refs: CRO-431
```

A `git commit --amend` works the same way for fixing a type/scope on the most recent commit
before pushing.

## 5. Reverting a change

`git revert` produces a message like `Revert "feat(geolab-base): add jq via apt.txt"`. That's not
valid Conventional Commits syntax, and release-please's parser can't read it — it's silently
dropped with no error. In practice, reverting a `feat:`/`fix:` commit does **not** undo its
effect: the original commit is still counted in full, so the next release still bumps the version
and still lists the reverted change in `CHANGELOG.md`, as if it had shipped.

Rewriting the revert with valid syntax (e.g. `revert(geolab-base): add jq via apt.txt`, with a
`This reverts commit <sha>.` footer) doesn't fix this either. release-please recognizes `revert:`
as its own type with its own `### Reverts` changelog section, but that does not cancel out or
remove the original commit's entry or version bump — the changelog ends up showing *both* the
original change and its revert, and the version still bumps as though the original change stuck.

**release-please has no built-in way to treat a revert as a net no-op.** If a change needs to be
fully retracted before its effect on versioning/the changelog:

- **Best, if the commit hasn't been released yet:** remove it from history (`git rebase`/`reset`)
  before it's ever picked up into a Release PR, rather than reverting it forward.
- **If it's already in an open, unmerged Release PR:** edit that PR's branch directly to drop the
  change, instead of adding a revert commit to `main`.
- **If it's already been released:** don't rely on `git revert`'s default message — write the
  undo as an ordinary new commit (`fix:`/`feat:` as appropriate to what the undo actually does).

## 6. What release-please does not see

release-please is driven entirely by git commit messages. It never inspects a diff, a GitHub repo
setting, a secret, or anything else that isn't a git commit. Three consequences are easy to miss:

**Only commits that touch `geolab-base/` count.** This repo's package path is `geolab-base` (see
`release-please-config.json`). A commit is only considered for versioning/changelog purposes if
its diff touches a file under `geolab-base/` — regardless of its type prefix. A
`fix:`/`feat:`/`feat!:` commit that only touches `.github/workflows/*.yml`,
`release-please-config.json`, root `README.md`, `.gitlab-ci.yml`, `CODEOWNERS`, or `docs/*.md` has
zero effect on the version; it's silently excluded, the same as a `chore:` commit would be.
Verified directly against this repo: a `fix(build-push): ...` commit that only touched
`build-push.yml` was skipped with "No user facing commits found." If you're changing CI/release
automation rather than the image, `chore:`/`ci:`/`docs:` is still the right type — just don't
expect a `fix:`/`feat:` on those files to bump anything either way.

**Squash-merging would break this model.** release-please reads whatever commits actually exist
in `main`'s history. If a PR is squash-merged, only the single squashed commit's message is ever
seen — the individual commits inside that PR are discarded from history entirely. This repo
doesn't currently squash-merge feature work, but if that changes, it's the squash commit's
message (often defaulted to the PR title) that needs to follow Conventional Commits.

**It doesn't read the diff at all.** Everything past the type prefix is decorative to
release-please — it doesn't know or care whether a commit added a package or rewrote the whole
Dockerfile. See [§5](#5-reverting-a-change) for how far this goes: a revert doesn't cancel
anything out, because release-please has no notion of "this undoes that."

## 7. How the pipeline works end-to-end

1. **Contributor commits using Conventional Commits.** Every PR merged to `main` contains
   commits following the format above.
2. **release-please runs on every push to `main`.** It scans all commits since the last release
   tag and classifies them by type.
3. **release-please opens or updates a "Release PR."** This PR's diff is just two things: the
   version file (`geolab-base/VERSION`) bumped to the next semantic version, and
   `geolab-base/CHANGELOG.md` with a new section listing every `feat:`/`fix:`/breaking commit
   since the last release, described using the commit messages themselves. No new code changes
   are introduced by this PR — it stays open and keeps updating itself as more commits land on
   `main`, until someone merges it.
4. **A maintainer reviews and merges the Release PR** when ready to cut a release. This is the
   one manual/human step in the process; nothing is auto-committed to `main` directly.
5. **On merge, release-please tags the release.** It creates a git tag (e.g. `v1.3.0`) and a
   corresponding GitHub Release with the changelog entry as the release notes.
6. **The GitHub Release publish event triggers the build/push pipeline.** `build-push.yml`
   listens for `release: types: [published]`, checks out the repo at that release, reads the
   version straight out of `geolab-base/VERSION` (which release-please just wrote as part of the
   merged Release PR), and tags the dev image with exactly that value. Reading the file directly
   instead of re-deriving the version from the git tag name is what guarantees the image tag and
   the package release are always the same version.

## 8. One-time setup reference

To enable this process, a repository needs:

- `release-please-config.json` at the repo root, `release-type: simple` (appropriate for a
  non-npm, non-language-specific repo), pointing at the version file to track
  (`version-file: "VERSION"`), with `include-component-in-tag: false` (see the "bugs already
  fixed" list in [§9](#9-rolling-this-out-to-earthscopegeolab)).
- `.release-please-manifest.json` tracking the current version per path.
- `.github/workflows/release-please.yml`, running `googleapis/release-please-action` on push to
  `main`, authenticated with a PAT (not the default token — see [§9](#9-rolling-this-out-to-earthscopegeolab)).
- `.github/workflows/build-push.yml`, triggered by `release: types: [published]`, reading
  `geolab-base/VERSION` and pushing to GHCR.
- `geolab-base/Dockerfile` with `COPY --chown=<runtime-user>:<runtime-user> VERSION /opt/VERSION`.

## 9. Rolling this out to earthscope/Geolab

### Decisions already made

**This supplements the GitLab pipeline, it doesn't replace it.** GitHub Actions + release-please
becomes the *dev* pipeline; `earthscope/Geolab`'s existing GitLab CI → AWS ECR pipeline stays the
actual production release, promoted via a deliberate, separate step (see
[§10](#10-promoting-a-dev-release-to-production-via-gitlab)).

**Starting version:** `earthscope/Geolab` already has real, currently-deployed images
(timestamp-versioned, on ECR) — there's no natural "1.0.0" the way there was in this sandbox.
Pick a starting version that makes sense for the team (e.g. `1.0.0` for a fresh semver start, or
something like `2025.1.0` for continuity with the existing timestamp scheme), and force it
explicitly via `Release-As:` (Step 5 below) — an arbitrary manifest value alone won't produce it,
because real commits already in the repo's history get counted too.

### Steps

1. **`.github/CODEOWNERS` already gates this.** It requires `@earthscope/cloud-enablement` to
   approve any change under `.github/`. Every workflow file below lives there — loop them in
   early rather than at PR time.

2. **Copy the workflow files, mostly as-is:** `.github/workflows/release-please.yml`,
   `.github/workflows/build-push.yml`, `.github/workflows/enforce-breaking-change-policy.yml`. No
   path adaptation needed — both repos use `geolab-base/Dockerfile` at the same relative path,
   and `build-push.yml`'s `IMAGE_NAME: ${{ github.repository }}/geolab-base` computes correctly
   for any repo without editing.

3. **Copy and adapt `release-please-config.json` / `.release-please-manifest.json`.** Copy the
   config as-is (package path `geolab-base` matches). For the manifest, set the starting version
   to `0.0.0` deliberately (not the real target — see Step 5):
   ```json
   { "geolab-base": "0.0.0" }
   ```

4. **Add `geolab-base/VERSION` and update the Dockerfile.** `earthscope/Geolab`'s Dockerfile has
   no version handling yet. Add `geolab-base/VERSION` containing the target starting version, and
   add this line to the Dockerfile:
   ```dockerfile
   COPY --chown=jovyan:jovyan VERSION /opt/VERSION
   ```
   Verified directly against the real `pangeo/base-image:latest` that `earthscope/Geolab` uses:
   the runtime user is `jovyan` (uid 1000), and neither `/opt` nor `/var` is writable by that
   user by default — only the `--chown` on the file itself makes it writable, without loosening
   the directory. Don't skip it.

   Do **not** add `geolab-base/CHANGELOG.md` — leave it absent; release-please creates it fresh
   from real commit history the first time it opens a Release PR. A hand-seeded placeholder just
   creates a duplicate-heading mess to clean up later (hit this in the sandbox).

5. **Force the real starting version.** With the manifest at `0.0.0` and real commit history
   already present, an ordinary push would compute *some* version, but not necessarily the right
   one:
   ```bash
   git commit --allow-empty -m 'chore: set initial release version' \
     -m 'Release-As: 1.0.0'
   ```
   (substitute the actual target version). Push *after* the workflow files and config from Steps
   2–4 are in place.

6. **Repo settings — do this before pushing Step 5.** Two settings must be in place, or the
   pipeline silently does nothing or fails partway through:
   - **Create a PAT and store it as the `RELEASE_PLEASE_TOKEN` secret.** `release-please.yml`
     authenticates with this instead of the default `GITHUB_TOKEN`. Not optional: GitHub does not
     let a release/tag created via the default token trigger other workflows, which means
     `build-push.yml`'s `release: published` trigger never fires — confirmed empirically in the
     sandbox (a real GitHub Release existed and `build-push.yml` had never once run). Create a
     fine-grained PAT scoped to the repo with **Contents: Read/Write** and
     **Pull requests: Read/Write**, then `gh secret set RELEASE_PLEASE_TOKEN --repo earthscope/Geolab`.
   - **Enable "Allow GitHub Actions to create and approve pull requests."** Settings → Actions →
     General → Workflow permissions. Without it, release-please fails at the last step with
     `GitHub Actions is not permitted to create or approve pull requests`. Also settable via:
     ```bash
     gh api -X PUT repos/earthscope/Geolab/actions/permissions/workflow \
       -f default_workflow_permissions=read \
       -F can_approve_pull_request_reviews=true
     ```

7. **Copy the docs, with names/URLs updated.** Update every reference to this sandbox's repo
   (`sparafina-earthscope/Geolab-release-process`) to `earthscope/Geolab`. The rules themselves
   apply unchanged.

8. **Test before trusting it:**
   1. Push the workflow files/config on a branch, open a PR, confirm `actionlint` and the
      breaking-change-policy check both pass.
   2. After merging to `main`, confirm `release-please` runs and — once Step 5's forcing commit
      lands — opens a Release PR proposing the target starting version.
   3. Merge that PR. Confirm a real GitHub Release and tag get created.
   4. Confirm `build-push.yml` actually runs off the `release: published` event (not just
      `workflow_dispatch`) — this is the one that silently didn't work before the PAT was in
      place.
   5. Pull the resulting image from GHCR and check `/opt/VERSION` inside it matches the release
      tag.

### Bugs already fixed in the files being copied — don't reintroduce them

- **Lowercase image names.** `build-push.yml` lowercases the computed image name in a shell step
  before tagging — `ghcr.io` rejects uppercase repo names, and `github.repository` preserves the
  real repo's casing.
- **`include-component-in-tag: false`** in `release-please-config.json`. Without it, release tags
  are `geolab-base-v1.0.0` instead of `v1.0.0`, which breaks `docker/metadata-action`-style semver
  detection and any hand-written git-ref parsing.
- **Reading `geolab-base/VERSION` directly** in `build-push.yml`, rather than parsing it back out
  of the git tag name. This guarantees the image tag and the release version are always
  identical, and sidesteps the tag-format issue above entirely.

## 10. Promoting a dev release to production via GitLab

Everything above produces a dev image in GHCR and a version decided by release-please. Getting
that same version built and pushed to AWS ECR for production is a **separate, deliberate action**
— not automatic.

**Confirmed:** promotion is triggered by opening and merging a merge request on the GitLab-side
copy of the repo. That merge is what kicks off `.gitlab-ci.yml`'s pipeline. This also confirms a
GitHub↔GitLab mirror exists — an MR can't be opened otherwise.

**Confirmed: it currently publishes with a timestamp, disconnected from `geolab-base/VERSION`.**
Merging the MR as things stand today does **not** produce a matching version — it produces
whatever `USE_TIMESTAMP_VERSION: "true"` generates, unrelated to the GitHub release. For "release
the same version" to actually be true, `.gitlab-ci.yml` needs a real change:

```yaml
variables:
  CONTAINER_REGISTRY_PLATFORM: "AWS-PUB"
  DOCKERFILE_RELPATH_IS_IMAGE_NAME: "true"
  GITLAB_HOSTED_RUNNER_SIZE: "saas-linux-medium-amd64"
  USE_TIMESTAMP_VERSION: "false"
  # Proposed — exact variable name unconfirmed, needs @earthscope/cloud-enablement:
  # something like IMAGE_VERSION or VERSION_FILE_PATH, pointed at geolab-base/VERSION
```

Turning off `USE_TIMESTAMP_VERSION` is the certain part of this suggestion. What replaces it
isn't — the shared `earthscope/infrastructure/gitlab-ci` template isn't visible from this repo,
so its actual variable name for "use this explicit version" (or whether it supports reading a
version file directly) is unknown. Options to raise with `@earthscope/cloud-enablement`, roughly
in order of fit:

- A variable that points at a version file (e.g. `VERSION_FILE_PATH: "geolab-base/VERSION"`) —
  best fit, since that file is already the single source of truth release-please maintains.
- A variable that takes an explicit version string (e.g. `IMAGE_VERSION`) — would need something
  in the merge request itself (a CI/CD variable, a commit convention) to carry the version
  through.
- Deriving the version from the git ref, if the pipeline could be made tag-triggered instead of
  MR-triggered — a bigger process change, since promotion is confirmed to go through an MR.

This is a change to a CODEOWNERS-gated file — `@earthscope/cloud-enablement` needs to make or
approve it either way.

Once `.gitlab-ci.yml` is actually publishing from the repo's version instead of a timestamp, the
shape of the promotion process is:

1. A GitHub release exists (e.g. `v1.2.0`) and has already been dev-tested via the GHCR image.
   `geolab-base/VERSION` in that commit already reads `1.2.0`.
2. Someone decides it's ready for production and opens a merge request bringing that commit into
   the GitLab-side repo.
3. Merging it triggers `.gitlab-ci.yml`, which builds `geolab-base` from that content and pushes
   to AWS ECR.
4. Confirm the resulting ECR image tag is `1.2.0`, not a timestamp, before considering it
   promoted — verify this the first few times rather than assuming the fix took effect.

This keeps the two systems cleanly separated: GitHub/release-please is the source of truth for
*what version something is and what changed*; GitLab is the source of truth for *what's actually
running in production*, and a human decides when those become the same thing.

## 11. `pangeo/base-image` update cadence

**It is not on a regular cycle — updates are entirely activity-driven, not scheduled.** Verified
directly against the source repo's (`pangeo-data/pangeo-docker-images`) GitHub Actions workflows
and the actual Docker Hub push history (627 tags, 507 distinct builds since March 2020).

**Why it's not scheduled:**
- `Build.yml` (rebuilds the image) triggers only on `push: branches: [master]` — purely reactive
  to merged commits.
- `Publish.yml` (creates the human-readable CalVer tags like `2026.06.04`) triggers only on
  `push: tags: ['*']` — someone has to manually push a git tag to cut a "release."
- The *only* cron in the repo is `WatchCondaForge.yml`, running daily (`0 0 * * *`) — but it
  doesn't rebuild anything itself. It checks conda-forge for a new `pangeo-notebook` metapackage
  version and, if found, opens a PR for a human maintainer to review and merge. A human merge is
  still required before any image actually rebuilds.

**What the actual history shows:**
- 507 distinct builds over ~6.25 years — averages to roughly one every 4–5 days, but that average
  is misleading. The real distribution is extremely bursty: 111 builds landed within an hour of
  another build, another 137 within a day (active-development clusters), separated by long quiet
  gaps.
- Gaps as long as 62–108 days show up between builds, especially recently.
- **The pace has slowed substantially.** In 2020–2022, updates landed roughly weekly. In the last
  12–18 months, gaps between updates have been 89, 91, and 108 days — closer to once a quarter
  now.

**Practical takeaway:** since the digests this repo has pinned (`04bb14b`, `04681e6`) don't track
`:latest`, and since the real `earthscope/Geolab` repo still uses `pangeo/base-image:latest`
unpinned, there's no fixed cadence to plan a "check for updates" process around. Checking monthly
would be more than sufficient given the current pace, but that's not guaranteed to hold — it
depends entirely on upstream maintainer activity, not a schedule.

## 12. Open questions

Things this playbook flags as unresolved rather than assumed, carried over from the source docs:

- **What variable (if any) does the `earthscope/infrastructure/gitlab-ci` shared template
  support for an explicit version**, to replace `USE_TIMESTAMP_VERSION: "true"`? Needs
  `@earthscope/cloud-enablement`. See [§10](#10-promoting-a-dev-release-to-production-via-gitlab).
- **What's the actual starting version for `earthscope/Geolab`?** Needs a team decision before
  Step 5 of the rollout in [§9](#9-rolling-this-out-to-earthscopegeolab) can run.
- **Should `geolab-base/Dockerfile` pin `pangeo/base-image` to a digest instead of tracking
  `:latest`?** Not decided either way here — [§11](#11-pangeobase-image-update-cadence) only
  establishes that there's no update schedule to plan around, not what this repo should do about
  it.
- **This document's own status.** It's a draft consolidating four separate docs. Once reviewed,
  decide whether it replaces them (with the originals removed or marked superseded) or continues
  to exist alongside them.
