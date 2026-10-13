# Architecture

This is a tour of the moving parts. If you're trying to debug a build or extend it, start here.

## Pipeline

```mermaid
flowchart LR
    check --> release --> build --> publish --> prune
```

| Job | File | Runtime | Purpose |
|---|---|---|---|
| `check` | `build.yml` → `scripts/check-updates.sh` | ~15s | Resolves the upstream (and NSS) ref to a commit SHA via `git ls-remote`. On a **scheduled** tick it skips the build when the latest release already records the SHA(s); on **push** (config changed) and **manual** runs it always builds. Emits `upstream_sha`, `nss_sha`, `need`. |
| `release` | `build.yml` | ~5s | Opens one draft release for the build job to upload into, and emits its tag. |
| `build` | `build.yml` → `scripts/prepare-build.sh` | 2-6h | One matrix job per `devices/<name>/` (only `aw1000` today). Checks out the upstream at the pinned SHA, runs `prepare-build.sh` (feeds, `.config`, overlays), sets reproducible-build env, compiles, uploads its images into the draft. |
| `publish` | `build.yml` | ~5s | Publishes the draft once the build succeeded; deletes it (tag included) and fails otherwise. |
| `prune` | `build.yml` → `scripts/prune-releases.sh` | ~10s | Keeps the newest `KEEP` releases of each prefix. Tested in `scripts/tests/prune-releases.test.sh`. |

## Why one release for all images?

There is one matrix job (the AW1000), and its one draft is all-or-nothing:
`publish` releases it only when the build succeeded, so a user never sees a
half-uploaded image set that reads as "this device is not supported". A draft
left behind is never pruned, which is why a failed build deletes it.

## Why split into jobs?

- **One network step** — `check` does all the SHA resolution and "did anything change?" logic in one place, so `build` is pure compile.
- **Re-runnable prune** — if cleanup fails, re-run just that job.
- **Clear failure attribution** — each step is its own red/green node in the Actions UI.

## Why no caching?

1. **Predictability.** Every release is built from a clean tree, so reproducing locally matches CI.
2. **Cache-poisoning surface.** OpenWrt's build dir is 10+ GB; corrupted caches produce broken images that look fine.
3. **Size limits.** OpenWrt builds blow past `actions/cache` limits anyway.

ccache is also disabled (`# CONFIG_CCACHE is not set` in the config) since it's a no-op without persistent storage.

## Reproducibility note

`SOURCE_DATE_EPOCH` is taken from the checked-out upstream commit (`git show -s --format=%ct HEAD`).
The image is intentionally *current* (latest `nss-edma-rework` head each build), so the SHA is pinned
per build but the tip moves as upstream does — that is the documented trade-off for "refresh on build".

## Why an `env:` block instead of a config file?

Every build parameter (upstream, NSS, feeds, target, retention, cron) lives in the `env:` block at
the top of `build.yml`, and the device set is the `build` job's matrix next to it. It is the one
file you edit — no separate YAML to parse, no extra tooling (`yq`) on the runner.

## Why `scripts/` instead of inline shell?

Inline shell in YAML is hostile to shellcheck, testing, and diffing. So all non-trivial logic lives in
`scripts/`, gets `set -euo pipefail`, sources `scripts/lib/log.sh`, and is reachable from `bash`
directly for local debugging.

## Test strategy

`prune-releases.sh` has unit tests — it has a clear input/output contract (JSON list → tags to
delete), is destructive on failure, and previously hid a "keep last N" bug for months.
`check-updates.sh` is exercised by running it directly (it prints the resolution and fails fast on bad
input); the rest orchestrate `make`, which isn't usefully unit-testable, and are guarded by
`actionlint` + `shellcheck`.
