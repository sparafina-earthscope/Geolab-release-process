# Rolling This Workflow Out to earthscope/Geolab

This repo was a sandbox for prototyping the release-please + GHCR build pipeline before
introducing it to the real repo, [earthscope/Geolab](https://github.com/earthscope/Geolab).
This doc is the step-by-step for making that move. It assumes you're comfortable with the
concepts in [release-process.md](release-process.md) — read that first if you haven't.

## Before you start: one decision only you can make

**Resolved: this supplements the GitLab pipeline, it doesn't replace it.** GitHub Actions +
release-please becomes the *dev* pipeline: it decides the semantic version from commit history
and builds a dev image to GHCR tagged with it. `earthscope/Geolab`'s existing `.gitlab-ci.yml` →
AWS ECR pipeline stays the actual production release, and production promotion is a **deliberate,
separate step** — GitLab does not automatically rebuild every time GitHub cuts a release. See
[Promoting a dev release to production via GitLab](#promoting-a-dev-release-to-production-via-gitlab)
below for that process.

**What's the starting version?**
`earthscope/Geolab` already has real, currently-deployed images (timestamp-versioned, on ECR).
There's no natural "1.0.0" here the way there was in the sandbox. Pick a starting version that
makes sense for your team — e.g. `1.0.0` if you're treating this as a fresh semver start, or
something like `2025.1.0` if you want continuity with the existing timestamp scheme. Whatever
you pick, follow the same forcing pattern used in Step 5 below (`Release-As:`) — don't expect an
arbitrary manifest value alone to produce it, because any real commits already in the repo's
history will get counted too.

## Step 1: `.github/CODEOWNERS` already gates this

`earthscope/Geolab`'s `CODEOWNERS` requires `@earthscope/cloud-enablement` to approve any change
under `.github/`. Every workflow file below lives there, so the PR introducing this needs their
review — loop them in early rather than at PR time.

## Step 2: Copy the workflow files, mostly as-is

Copy these three files from this repo to `earthscope/Geolab` unchanged:

- `.github/workflows/release-please.yml`
- `.github/workflows/build-push.yml`
- `.github/workflows/enforce-breaking-change-policy.yml`

No path adaptation needed for `build-push.yml`/`enforce-breaking-change-policy.yml` — both repos
use `geolab-base/Dockerfile` at the same relative path, and `build-push.yml`'s
`IMAGE_NAME: ${{ github.repository }}/geolab-base` computes correctly for any repo without
editing (it already handles the lowercase-name gotcha we found — see the fixes list below).

## Step 3: Copy and adapt `release-please-config.json` / `.release-please-manifest.json`

Copy `release-please-config.json` as-is (package path `geolab-base` matches). For
`.release-please-manifest.json`, set the starting version to whatever you decided in the
"decisions" section above:

```json
{
  "geolab-base": "0.0.0"
}
```

Using `0.0.0` (rather than your real target version) here is deliberate — see Step 5.

## Step 4: Add `geolab-base/VERSION` and update the Dockerfile

`earthscope/Geolab`'s Dockerfile has no version handling at all yet. Add the file:

```
geolab-base/VERSION
```
containing your target starting version (e.g. `1.0.0`), and add this line to
`geolab-base/Dockerfile` (right after the `LABEL` block is a reasonable spot, matching this
repo's layout):

```dockerfile
COPY --chown=jovyan:jovyan VERSION /opt/VERSION
```

Verified directly against the real `pangeo/base-image:latest` that `earthscope/Geolab` uses:
the runtime user is `jovyan` (uid 1000), and neither `/opt` nor `/var` is writable by that user
by default — only the `--chown` on the file itself makes it writable, without loosening the
directory. Don't skip the `--chown`.

Do **not** add a `geolab-base/CHANGELOG.md` — leave it absent. release-please creates it fresh
from real commit history the first time it opens a Release PR; a hand-seeded placeholder just
creates a duplicate-heading mess to clean up later (we hit this in the sandbox).

## Step 5: Force the real starting version

With the manifest at `0.0.0` and real commit history already in `earthscope/Geolab`, an ordinary
push would compute *some* version, but not necessarily the one you actually want as the
baseline. Force it explicitly, the same way this repo's baseline was set:

```bash
git commit --allow-empty -m 'chore: set initial release version' \
  -m 'Release-As: 1.0.0'
```
(substitute your actual target version for `1.0.0`). Push this to `main` *after* the workflow
files and config from Steps 2–4 are already in place — release-please needs its config to exist
before it can act on this commit.

## Step 6: Repo settings (do this before pushing Step 5)

Two settings must be in place, or the pipeline silently does nothing or fails partway through —
both bit us in the sandbox in ways that took real debugging to trace:

1. **Create a PAT and store it as the `RELEASE_PLEASE_TOKEN` secret.** `release-please.yml`
   authenticates with this instead of the default `GITHUB_TOKEN`. This isn't optional: GitHub
   does not let a release/tag created via the default `GITHUB_TOKEN` trigger other workflows,
   which would mean `build-push.yml`'s `release: published` trigger never fires — confirmed
   empirically in the sandbox (a real GitHub Release existed and `build-push.yml` had never once
   run). Create a fine-grained PAT scoped to this repo with **Contents: Read/Write** and
   **Pull requests: Read/Write**, then:
   ```bash
   gh secret set RELEASE_PLEASE_TOKEN --repo earthscope/Geolab
   ```
2. **Enable "Allow GitHub Actions to create and approve pull requests."** Settings → Actions →
   General → Workflow permissions, on `earthscope/Geolab`. Without it, release-please can build
   the Release PR's commit but fails at the last step with
   `GitHub Actions is not permitted to create or approve pull requests`. Can also be set via:
   ```bash
   gh api -X PUT repos/earthscope/Geolab/actions/permissions/workflow \
     -f default_workflow_permissions=read \
     -F can_approve_pull_request_reviews=true
   ```

## Step 7: Copy the docs, with names/URLs updated

Copy `docs/release-process.md` and `docs/release-process-getting-started.md`. Update every
reference to this sandbox's repo (`sparafina-earthscope/Geolab-release-process`) to
`earthscope/Geolab`. Everything else — the conventional-commit rules, the "what release-please
doesn't see" caveats, the `feat!:` quoting warning — applies unchanged.

## Step 8: Test before trusting it

Don't assume it works — verify, the same way each piece was verified in the sandbox:

1. Push the workflow files and config on a branch, open a PR, confirm `actionlint` and the
   breaking-change-policy check both pass.
2. After merging to `main`, confirm the `release-please` workflow runs and — once Step 5's
   forcing commit lands — opens a Release PR proposing your target starting version.
3. Merge that PR. Confirm a real GitHub Release and tag get created.
4. Confirm `build-push.yml` actually runs off that `release: published` event (not just
   `workflow_dispatch`) — this is the one that silently didn't work before the PAT was in place.
5. Pull the resulting image from GHCR and check `/opt/VERSION` inside it matches the release tag.

## Promoting a dev release to production via GitLab

Everything above produces a dev image in GHCR and a version decided by release-please. Getting
that same version built and pushed to AWS ECR for production is a **separate, deliberate
action** — not automatic.

**Confirmed:** promotion is triggered by opening and merging a merge request on the GitLab-side
copy of the repo. That merge is what kicks off `.gitlab-ci.yml`'s pipeline. This also confirms a
GitHub↔GitLab mirror exists — an MR can't be opened otherwise.

**Confirmed: it currently publishes with a timestamp, disconnected from `geolab-base/VERSION`.**
Merging the MR as things stand today does **not** produce a matching version — it produces
whatever `USE_TIMESTAMP_VERSION: "true"` generates, unrelated to the GitHub release. For
"release the same version" to actually be true, `.gitlab-ci.yml` needs a real change:

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
isn't — the shared `earthscope/infrastructure/gitlab-ci` template isn't visible to me, so I
don't know its actual variable name for "use this explicit version," or whether it supports
reading a version file directly. Options to raise with `@earthscope/cloud-enablement`, roughly
in order of how well they'd fit this repo's existing setup:

- A variable that points at a version file (e.g. `VERSION_FILE_PATH: "geolab-base/VERSION"`) —
  best fit, since that file is already the single source of truth release-please maintains.
- A variable that takes an explicit version string (e.g. `IMAGE_VERSION`) — would need something
  in the merge request itself (a CI/CD variable set on the MR, a commit convention) to carry the
  version through to the pipeline.
- Deriving the version from the git ref, if the pipeline could be made tag-triggered instead of
  MR-triggered — a bigger process change than what's described here, since promotion is
  confirmed to go through an MR, not a tag push.

This is a change to a CODEOWNERS-gated file — `@earthscope/cloud-enablement` needs to make or
approve it either way. Treat the variable block above as a starting point for that conversation,
not a change to make unilaterally.

Once `.gitlab-ci.yml` is actually publishing from the repo's version instead of a timestamp, the
shape of the promotion process is:

1. A GitHub release exists (e.g. `v1.2.0`) and has already been dev-tested via the GHCR image.
   `geolab-base/VERSION` in that commit already reads `1.2.0`.
2. Someone decides it's ready for production and opens a merge request bringing that commit (or
   the equivalent state) into the GitLab-side repo.
3. Merging it triggers `.gitlab-ci.yml`, which builds `geolab-base` from that content and pushes
   to AWS ECR.
4. Confirm the resulting ECR image tag is `1.2.0`, not a timestamp, before considering it
   promoted — verify this the first few times rather than assuming the fix took effect.

This keeps the two systems cleanly separated: GitHub/release-please is the source of truth for
*what version something is and what changed*; GitLab is the source of truth for *what's actually
running in production*, and a human decides when those become the same thing.

## Bugs already fixed in the files you're copying — don't reintroduce them

Copying the current versions of these files gets you these fixes for free. If anything gets
hand-retyped instead of copied, watch for:

- **Lowercase image names.** `build-push.yml` lowercases the computed image name in a shell
  step before tagging — `ghcr.io` rejects uppercase repo names, and `github.repository`
  preserves the real repo's casing.
- **`include-component-in-tag: false`** in `release-please-config.json`. Without it, release
  tags are `geolab-base-v1.0.0` instead of `v1.0.0`, which breaks `docker/metadata-action`-style
  semver detection (not used here, but breaks it if anyone reintroduces that pattern) — and any
  hand-written git-ref parsing.
- **Reading `geolab-base/VERSION` directly** in `build-push.yml`, rather than parsing it back out
  of the git tag name. This is what guarantees the image tag and the release version are always
  identical, and sidesteps the tag-format issue above entirely.
