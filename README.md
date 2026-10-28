# opencode2api — patch branch

> This branch carries our local production patches. It is **not** upstream.
> For the original documentation, see `main` or upstream
> `jasonxu114514/opencode2api`.

## Base

Upstream tag `v1.4.5` + local commits (see `git log --oneline v1.4.5..patch`).

## Why these patches exist

The free tier meters quota per project id and egress, resetting daily at
00:00 UTC (upstream issue #156). Once a session burns its bucket, every
remaining exit answers `429 Free usage limit reached` for the same identity —
while a fresh identity on any exit returns 200. Verified on our box: one
pinned burned session failed 8 legs in a row; fresh sessions succeeded first
leg, 8/8.

Three facts shaped the design:

- The cursor start is deterministic per session (FNV affinity hash), so the
  same session always walks the same first exit. "Same exit always 429s" is
  affinity, not a broken exit.
- `#pool` nodes are never cooled by design (`MarkFailure` skips them,
  `Next()` returns them regardless of cooldown), so cooling a proxy name
  would be a no-op. The bucket is bound to the **identity**, not the exit.
- Randomizing the project id alone is not enough; the session must change
  too (upstream #156 proved 0/8 → 8/8 only when both change).

## Patches

### 1. Per-leg rotation on quota 429 (sesrot1)

- `internal/gateway/abuse.go`: new `isFreeUsageLimit` — status 429 carrying
  the "Free usage limit" message. Other 429s (edge rate limits, empty bodies)
  keep the old behavior.
- `internal/gateway/upstream.go` (`doAnonymousUpstream`): when a leg draws a
  quota 429, the next exit is tried under a freshly minted session
  (`CanonicalSessionID` over a random signal, so the timestamp stays inside
  the upstream acceptance window) and a fresh project id.
- Logged as `anonymous_session_rotated`. Anonymous lane only.

### 2. Pre-rotation of burned sessions (sesrot2)

- `internal/gateway/gateway.go`: new `burnedSessions` — sessions that
  recently drew a quota 429, remembered for 15 minutes (mutex-guarded,
  capped at 512 entries with expiry-first/random eviction).
- `doAnonymousUpstream` entry: a request carrying a burned session starts on
  a fresh identity immediately instead of spending its first leg re-proving
  the bucket is burned. Logged as `anonymous_session_prerotated`.
- The burned identity is recorded *before* the replacement is minted, so the
  memory always points at the identity that actually failed. Healthy sessions
  are untouched: normal affinity and prompt-cache behavior is preserved.

## Build & deploy (box VPS 192.129.200.200)

```bash
docker build -t oc2:v1.4.5-sesrotN --build-arg VERSION=v1.4.5-sesrotN /root/oc2-patch
docker tag oc2:v1.4.5-sesrotN ghcr.io/jasonxu114514/opencode2api:v1.4.5
docker stop oc2 && docker rm oc2
docker compose -f /root/oc2/docker-compose.yml up -d
```

The retag keeps `docker-compose.yml` untouched (`ghcr.io/...:v1.4.5`), so the
autoupdate timer keeps working (`already latest`, no-op). The next upstream
release pull replaces the patched image — that is the intended handoff.

Rollback: the pre-patch image `oc2:v1.4.5` is retained on the box.
`docker tag oc2:v1.4.5 ghcr.io/jasonxu114514/opencode2api:v1.4.5` + recreate.

## Verify

```bash
curl -s http://127.0.0.1:8086/healthz   # version v1.4.5-sesrotN, keys, proxies
docker logs oc2 2>&1 | grep 'anonymous_session_prerotated'   # pre-rotations firing
docker logs oc2 2>&1 | grep 'anonymous_session_rotated'      # leg rotations firing
```

Expected after sesrot2: a request with a burned session finishes in 1 attempt
(~3s) instead of 429-then-200 in 2 attempts (~6s); requests with fresh
sessions finish in 1 attempt.

## Rebase policy

Every upstream release: rebase `patch` onto the new tag, rebuild, redeploy,
verify as above, and append the sbc deployment record (`b60ae47a…`).

## Related

- Upstream issue #156 + PR #157 (probes/Playground identity) — complementary,
  not overlapping: they fix the probes and the bench, these patches fix live
  traffic.
- Deployed on box VPS only (`oc2` container). truenas untouched.
- Box source tree: `/root/oc2-patch` (`.bak-sesrotN` files are the pre-patch
  backups). Vault holds the removed zen keys (`oc2 zen key …`).
