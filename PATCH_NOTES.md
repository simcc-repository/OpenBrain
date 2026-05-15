# SIMCC fork patch notes

This fork (`simcc-repository/OpenBrain`, branch `simcc-main`) carries local modifications
on top of upstream `srnichols/OpenBrain` (default branch `master`).

## Why this fork exists

The upstream `openrouter` embedder provider in `src/embedder/openrouter.ts` hardcodes
`https://openrouter.ai/api/v1` as the API base URL. We route embedding and metadata-
extraction traffic through our self-hosted **LiteLLM gateway** (on `192.168.50.148:4000`)
so calls are cost-tracked in Langfuse and use our existing virtual-key management. This
fork adds an `OPENROUTER_BASE_URL` env var so the provider can be pointed at LiteLLM
(or any OpenAI-compatible endpoint).

We also adjusted `docker-compose.yml` for our deployment topology on `192.168.50.148`:
- Postgres is internal-only (`expose:` not `ports:`) — host port 5432 is shared.
- The `api` service is attached to the external Docker network `litellm-langfuse_default`
  so `litellm:4000` resolves from inside the container.

## Patches in this branch

| # | File | Purpose |
|---|---|---|
| 1 | `src/embedder/openrouter.ts` | Add `OPENROUTER_BASE_URL` env override (default unchanged) |
| 2 | `docker-compose.yml` | Postgres internal-only, attach `api` to `litellm-langfuse_default`, switch from `env_file` to explicit `environment:` block (Portainer compatibility) |
| 3 | `db/init.sql` + `db/migrations/00{1,2}*.sql` | Change `VECTOR(768)` → `VECTOR(1024)` to match bge-m3 embedding dimensions |

A captured diff lives under `patches/0001-add-openrouter-base-url.patch` for
durability — regenerate after any rebase with `git format-patch -1 <patch-commit> -o patches/`.

## Applying upstream updates

```bash
cd OpenBrain
git fetch upstream
git checkout simcc-main
git rebase upstream/master      # NOTE: upstream default branch is "master", not "main"
# Resolve any conflicts in src/embedder/openrouter.ts or docker-compose.yml.
# Verify the OPENROUTER_BASE_URL override is still in place.
git push --force-with-lease origin simcc-main
```

Then in Portainer → openbrain stack → **Pull and redeploy**.

## Drift checks after each rebase

1. `git log simcc-main ^upstream/master` should show **only** our patch commits — nothing else.
2. `grep -rn 'openrouter\.ai' src/` should return **zero** hits inside `src/embedder/openrouter.ts`
   (matches in `docs/` or `setup.{sh,ps1}` are fine — they're documentation).
3. `grep -n 'OPENROUTER_BASE_URL' src/embedder/openrouter.ts` should match.
4. `docker-compose.yml` should NOT publish postgres on host port 5432 and SHOULD attach `api`
   to `litellm-langfuse_default`.

## Retire the fork when upstream adds OPENROUTER_BASE_URL natively

If upstream ever adds the same env var (consider opening a PR to make this happen),
delete `simcc-main` and track `upstream/master` directly. The compose-file overrides
can move into a small `docker-compose.override.yml` instead of being committed to the fork.
