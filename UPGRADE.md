# OpenClaw Upgrade Runbook

This fork uses `.openclaw-version` as the single source of truth for the OpenClaw version. The Dockerfile reads from this file at build time. To upgrade, change the file and commit — Railway rebuilds automatically.

## The happy path

1. New OpenClaw release ships
2. The auto-PR workflow opens a PR in this fork ("Bump OpenClaw to X.Y.Z")
3. Read the linked release notes
4. Merge the PR (only if it's a stable release, not a pre-release/beta)
5. Wait ~9 min for Railway to build and pass healthcheck
6. Verify: ask Maitreya `run: openclaw --version`

## Manual upgrade (if automation isn't enough)

1. Backup `/data/` first. Ask Maitreya:
```
   create a tar backup of /data/.openclaw and /data/workspace* in /data/
```
2. Edit `.openclaw-version` in this repo to the target version
3. Commit to `main`
4. Wait for Railway rebuild
5. Verify with Maitreya: `run: openclaw --version`

## Pre-release versions

The auto-PR workflow filters out versions with a `-` suffix (e.g. `2026.5.3-1`, `2026.6.0-beta`). These are pre-releases on the `beta` dist-tag and should not run in production. The workflow only opens PRs for stable releases on the `latest` dist-tag.

## Known constraints

- **Healthcheck timeout: 600s** in `railway.toml`. Do not lower this. Plugin init and `openclaw doctor --fix` together take ~5.5 min on first boot after upgrade.
- **Node version: 24-bookworm** in `Dockerfile`. OpenClaw 2026.4.x+ requires Node 24+.
- **Build cache invalidation:** Railway sometimes redeploys cached images instead of rebuilding. If a version bump doesn't take effect, force a fresh commit (any change) to invalidate the cache.

## Verification commands

Always use these to verify the running version, in order of reliability:

```
run: openclaw --version
```

```
run: which openclaw && cat /usr/local/lib/node_modules/openclaw/package.json | grep version
```

**Do not** trust `/data/.openclaw/VERSION` — it's written inconsistently and may be stale.

## If a version bump breaks production

Roll back by editing `.openclaw-version` to the previous version and committing. Railway will rebuild with the older version. The persistent volume `/data/` is untouched by this — agent state and config survive the rollback.

If `/data/` itself got corrupted by the upgrade attempt, restore from the most recent tarball backup in `/data/openclaw-data-backup-*.tar.gz`.

## Rollback target

The last known-good versions:
- `2026.5.2` (current, working as of May 4 2026)
- `2026.4.23` (previous, working — Node 24 required)
- `2026.4.15` (original — Node 22 was sufficient)
