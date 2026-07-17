# Release Process — The Basics

This doc explains how version numbers and the changelog get created for this repository (GeoLab). It's written for someone new to the team — no prior GitHub Actions or release-tooling experience assumed.

For the full technical reference, see [release-process.md](release-process.md). This page is the short version.

## The big picture, in three sentences

1. When you commit code, you write your commit message in a specific format that says *what kind* of change it is (a fix? a new feature? something that breaks compatibility?).
2. A bot reads those commit messages and figures out the next version number and changelog entry for you — you never edit a version number or `CHANGELOG.md` by hand.
3. When that version is officially released, our Docker image build pipeline automatically builds and publishes it with the right version tag.

That's it. Your only job as a developer is **step 1: write good commit messages**. Everything after that is automated.

## A few terms you'll see

- **Semantic versioning (semver):** version numbers shaped like `MAJOR.MINOR.PATCH`, e.g. `1.4.2`. Each part increases for a different reason (see table below).
- **Changelog:** a running list of what changed in each version, kept in `CHANGELOG.md`.
- **release-please:** the GitHub bot/tool that reads commit messages and proposes the next version + changelog for us.
- **GHCR:** GitHub Container Registry — where our built Docker images get pushed to (`ghcr.io/...`).

## Step 1: Write your commit message in the right format

Every commit message starts with a **type**, a colon, then a short
description:

```text
<type>: <description>
```

The type tells release-please what to do with the version number:

| You write... | Version changes like... | Use it when... |
|---|---|---|
| `fix: ...` | `0.1.0` → `0.1.1` | You fixed a bug, patched a security issue, or pinned a broken package version |
| `feat: ...` | `0.1.0` → `0.2.0` | You added or removed something — a package, a tool, a capability |
| `feat!: ...` | `0.1.0` → `1.0.0` | You swapped the base image. This is the *only* thing that gets a major bump in this repo. |
| `chore:`, `docs:`, `ci:`, `refactor:`, `test:` | *(no change)* | Anything else — cleanup, docs, CI tweaks. These don't show up in the changelog. |

You can optionally add a scope in parentheses to say *where* the change happened, e.g. `fix(geolab-base): ...`. It's just for readability.

### Real examples

```bash
git commit -m "fix(geolab-base): pin numpy to 1.26.4 to fix build failure"

git commit -m "feat(geolab-base): add scipy to environment.yml"

git commit -m "feat!: bump base image from python:3.11 to python:3.12" \
  -m "BREAKING CHANGE: base image major version changed; downstream images must rebuild"
```

That first line (`fix: ...` / `feat: ...`) is the only part that matters for versioning. Everything else — a longer explanation, a ticket reference — can go on the lines below it, and release-please will just carry it along as extra detail.

**Tip:** if you're not sure whether something is a `fix` or a `feat`, ask yourself: "did I add something new, or repair something that was broken/wrong?" New = `feat`. Repaired = `fix`.

## Step 2: Open your PR like normal

Nothing changes about how you open and merge pull requests. Just make sure the commits in it use the format above. If a PR has multiple commits with different types (some `fix`, some `feat`), that's fine elease-please looks at all of them.

## Step 3: What happens after you merge (you don't have to do this part)

1. release-please notices your commits landed on `main`.
2. It opens (or updates) a standing pull request called something like `chore: release 1.3.0`. You'll see this PR appear/update automatically don't be surprised by it, you don't need to write it yourself.
3. This PR is not adding any code — it only contains the updated version file and a new `CHANGELOG.md` entry, written using your commit messages.
4. When someone on the team is ready to actually cut a release, they merge that PR.
5. Merging it creates an official GitHub Release and a git tag like
   `v1.3.0`.
6. That Release triggers our existing image build workflow (`build-push.yml`), which builds and pushes the Docker image to GHCR tagged `1.3.0`, `1.3`, and `latest`.

You don't need to touch `VERSION`, `CHANGELOG.md`, or the Docker tags yourself — writing your commit message correctly in Step 1 is what drives all of this.

## Cheat sheet

```text
Fixed something?          -> fix: describe the fix
Added or removed a package? -> feat: describe what changed
Swapped the base image?   -> feat!: describe it, plus a BREAKING CHANGE: footer
Just cleanup/docs/CI?     -> chore: / docs: / ci: (won't appear in changelog)
```

## Status in this repo

This process is live: `release-please-config.json`, `.release-please-manifest.json`, and the `release-please` and `build-push` workflows are all wired up. Commits landing on `main` are picked up automatically as described above.
