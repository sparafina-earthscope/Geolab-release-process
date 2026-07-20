# GeoLab Release Playbook

## `pangeo/base-image` update cadence

**It is not on a regular cycle — updates are entirely activity-driven, not scheduled.**

Verified directly against the source repo's (`pangeo-data/pangeo-docker-images`) GitHub Actions
workflows and the actual Docker Hub push history (627 tags, 507 distinct builds since March
2020).

### Why it's not scheduled

- `Build.yml` (rebuilds the image) triggers only on `push: branches: [master]` — purely reactive
  to merged commits.
- `Publish.yml` (creates the human-readable CalVer tags like `2026.06.04`) triggers only on
  `push: tags: ['*']` — someone has to manually push a git tag to cut a "release."
- The *only* cron in the repo is `WatchCondaForge.yml`, running daily (`0 0 * * *`) — but it
  doesn't rebuild anything itself. It checks conda-forge for a new `pangeo-notebook` metapackage
  version and, if found, opens a PR for a human maintainer to review and merge. A human merge is
  still required before any image actually rebuilds.

### What the actual history shows

- 507 distinct builds over ~6.25 years — averages to roughly one every 4–5 days, but that average
  is misleading. The real distribution is extremely bursty: 111 builds landed within an hour of
  another build, another 137 within a day (active-development clusters), separated by long quiet
  gaps.
- Gaps as long as 62–108 days show up between builds, especially recently.
- **The pace has slowed substantially.** In 2020–2022, updates landed roughly weekly. In the last
  12–18 months, gaps between updates have been 89, 91, and 108 days — closer to once a quarter
  now.

### Practical takeaway for this repo

Since the digests this repo has pinned (`04bb14b`, `04681e6`) don't track `:latest`, and since
the real `earthscope/Geolab` repo still uses `pangeo/base-image:latest` unpinned, there's no
fixed cadence to plan a "check for updates" process around. Checking monthly would be more than
sufficient given the current pace, but that's not guaranteed to hold — it depends entirely on
upstream maintainer activity, not a schedule.
