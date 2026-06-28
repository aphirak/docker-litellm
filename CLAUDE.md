# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Docker packaging of the [LiteLLM](https://github.com/BerriAI/litellm) proxy — a single OpenAI-compatible API gateway in front of 100+ LLM providers. This repo is **not** the LiteLLM source; LiteLLM itself is pulled from PyPI (`litellm[proxy]`) at image build time. The repo's own code is two bash scripts plus packaging/CI. For upstream LiteLLM behavior, check the upstream project, not here.

## Commands

There is no compile/lint/test toolchain in-tree. Everything happens through Docker. The same checks CI runs:

```bash
# Lint shell scripts (the main "test" you run locally)
SHELLCHECK_OPTS="-e SC1090,SC1091" shellcheck **/*.sh

# Build the image
docker build -t litellm-test .

# Validate the compose file
docker compose -f docker-compose.yml config

# Run the full stack
cp litellm.env.example litellm.env   # edit keys, then:
docker compose up -d && docker logs litellm
```

Exercising the proxy after `docker build -t litellm-test .` (mirrors `proxy_test.yml`):

```bash
docker run --name t -v litellm-test-data:/etc/litellm -p 4000:4000/tcp -d litellm-test
docker exec t curl -sf http://127.0.0.1:4000/health/liveliness
docker exec t litellm_manage --showkey
docker exec t litellm_manage --listmodels
docker exec t litellm_manage --addmodel openai/gpt-4o --key sk-... --alias my-gpt4
```

No way to run a "single test" — `proxy_test.yml` is a matrix of full container runs keyed by `test_id`
(`no-env`, `with-env`, `with-master-key`, `with-mcp`, `with-shared-vol`, `with-litellm-shared`).
To reproduce one locally, replicate that `test_id`'s `docker run` block from `.github/workflows/proxy_test.yml`.

## Architecture

Two bash scripts carry all the logic. Read both before changing behavior.

**`run.sh`** — container entrypoint (`CMD`), runs as PID 1.
- Refuses to run outside a container (checks `/.dockerenv`, `/proc/1/sched`, etc.).
- Sources `/litellm.env` if bind-mounted (takes precedence over `--env-file`).
- Reads `LITELLM_*` env vars, sanitizes each through `nospaces`/`noquotes`, validates port/log-level/DB-URL/host.
- Master key: uses `LITELLM_MASTER_KEY` if set, else reads `/etc/litellm/.master_key`, else generates one. Always synced to the file (chmod 600).
- Generates `/etc/litellm/config.yaml` **only if absent** — existing config is preserved across restarts so `model_list` survives. The `mcp_servers:` block, however, is rewritten on **every** start from `LITELLM_MCP_URL` (injected or removed).
- On first run only (gated by `/etc/litellm/.initialized`), auto-adds models for any provider key present (`gpt-4o`/`gpt-4o-mini`, `claude-3-6-sonnet`, etc.) by appending to config.yaml.
- Ollama aliases are re-ensured on every start (not just first run): `ollama/llama3.2:3b` (legacy) and `ollama-chat/llama3.2:3b` (chat-native, `supports_function_calling`).
- Starts `litellm --config ... &` in the background, waits on `/health/liveliness`, then `wait`s on the PID. A `trap cleanup INT TERM` forwards SIGTERM to the litellm child for graceful shutdown.

**`manage.sh`** — symlinked to `/usr/local/bin/litellm_manage`, invoked via `docker exec`.
- Models/MCP servers are edited by **rewriting `config.yaml` directly** (Python + PyYAML), then triggering a restart via `_restart_proxy` → `kill -TERM 1`. PID 1 is `run.sh`; its trap exits cleanly and Docker's `restart: always` brings it back with the new config. **This is the mechanism — there is no hot reload; config changes require the container to restart.**
- Virtual keys (`--createkey`/`--listkeys`/`--deletekey`) are different: they hit the running proxy's REST API (`/key/generate` etc.) with the master key, and require a DB (gated by the `.db_configured` marker).
- `--showkey`/`--getkey`/model edits work without the proxy running; key operations and `--listmodels` require it (`check_server`).

**Config/state lives in the `/etc/litellm` volume**, not in the image: `config.yaml`, `.master_key`, `.initialized` (first-run marker), `.db_configured`, `.port`. Backing up the volume preserves the key and models.

### Inter-script contract (keep these in sync)

These constants/markers are referenced by both `run.sh` and `manage.sh` and by CI — changing one side silently breaks the others:
- File paths/markers: `/etc/litellm/{config.yaml,.master_key,.port,.initialized,.db_configured}`.
- The `kill -TERM 1` restart depends on `run.sh` being PID 1 with the `cleanup` trap installed.
- The container-detection guard is duplicated in both scripts.

### Passing data to embedded Python

Both scripts embed Python heredocs for YAML/JSON edits and **pass all values through environment variables** (`_MN`, `_P`, `_AK`, `_AB`, `_SFC`, `_URL`, `_KEY`, ...) rather than string-interpolating into the script — this avoids shell-quoting/injection issues. Preserve this pattern when adding fields.

### Compose stack

`docker-compose.yml` runs three services: `litellm-init` (one-shot, runs `scripts/litellm-init.sh` to seed the Postgres password into the shared `litellm-secrets` volume), `db` (postgres:18), and `litellm`. `litellm-init.sh` generates a random 32-char password for fresh volumes but uses the legacy literal `litellm` password when it detects existing Postgres data (backward compat). The proxy reads the password via `LITELLM_POSTGRES_PASSWORD_FILE` and, in `run.sh`, URL-encodes it into a `postgresql://...@db:5432/litellm` URL.

### Self-hosted-stack integration

`run.sh` auto-reads provider keys from shared volumes when mounted (part of [self-hosted-ai-stack](https://github.com/hwdsl2/self-hosted-ai-stack)): `/var/lib/ollama-shared/.api_key`, `/var/lib/mcp-shared/.api_key`, and writes its own master key to `/var/lib/litellm-shared/.api_key`. Don't remove these volume probes without checking the stack.

## CI

Pushes to `main` touching the image/scripts run `main.yml` → `shellcheck` + `proxy_test` → `buildx` (multi-arch amd64/arm64 to Docker Hub + Quay). `test.yml` runs shellcheck + proxy_test for compose/script-only changes. `cron.yml` rebuilds when the `python:3.12-slim` base updates. All jobs are gated on `github.repository_owner == 'hwdsl2'`, so they're no-ops on forks. `proxy_test.yml` is the behavioral contract — if you change `run.sh`/`manage.sh` semantics, update its assertions.

## Conventions

- Scripts must pass `shellcheck` with only `SC1090,SC1091` excluded.
- When behavior changes, update `README.md`, `litellm.env.example`, and the compose example — and the matching translated READMEs (`README-zh.md`, `README-zh-Hant.md`, `README-ru.md`) when user-facing.
- Never commit master keys, provider API keys, or secrets (see `CONTRIBUTING.md`).
