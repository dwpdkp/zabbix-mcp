# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Repository Overview

This is an MCP (Model Context Protocol) server that exposes tools for interacting with the
Zabbix API — hosts, host groups, templates, items, triggers, problems, events, history/trends,
users, proxies, maintenance periods, actions, media types, graphs, discovery rules, SLAs,
services, scripts, user macros, and configuration import/export.

**This repo is a fork, not an upstream original.** Check `git remote -v` before assuming
upstream behavior:
- `origin` → `https://github.com/dwpdkp/zabbix-mcp.git` (Doug's fork, what's checked out here)
- `upstream` → `https://github.com/mhajder/zabbix-mcp.git` (original author, Mateusz Hajder)

The fork carries **security hardening not present upstream** — confirmation gates on
destructive/privilege-changing tools (see "Fork-Specific Confirmation Gates" below) — plus a
few bug fixes (e.g. `count_output=True` returning `len(str)` instead of the actual count) and
new tools (`action_create`, `action_update`, `action_delete`). Don't assume upstream docs,
issues, or PyPI releases reflect what's actually in this checkout. `pyproject.toml` still
publishes as `zabbix-mcp` (same PyPI name as upstream) — this fork is **not currently
republished under a different package name**, so `uvx zabbix-mcp` from PyPI pulls upstream's
code, not this fork's. Only `uv run zabbix-mcp` from this cloned directory runs the fork.

## Development Commands

### Setup
```bash
uv sync --group dev            # Install deps + dev tooling (pytest, ruff, ty, prek, commitizen, typos)
uv run prek install            # Install pre-push/commit-msg/pre-commit hooks (prek, not pre-commit)
```

### Running the Server
```bash
uv run zabbix-mcp              # Entry point defined in pyproject.toml [project.scripts]
uv run -m zabbix_mcp.server    # Direct module execution
```

### Code Quality
```bash
uv run ruff check .            # Lint
uv run ruff format .           # Format
uv run ty check .              # Type checking (Astral's ty)
uv run pytest                  # Run tests
uv run pytest --cov=src/       # With coverage
```

**Note on test coverage**: `tests/` currently contains only `test_utils.py` — there is no
test coverage for `zabbix_tools.py` (the 56-tool, ~4900-line file that does almost all the
work) or `zabbix_client.py`. Treat this as a real gap, not an oversight to route around;
changes to tool behavior are effectively unverified by CI beyond linting/type-checking.

### Environment Setup
Copy `.env.example` to `.env` (or set the same vars via the consuming `.mcp.json`'s `env`
block — see "Consumption: Homelab Mac vs. Work Laptop" below).

**Required:**
- `ZABBIX_URL` — Zabbix API endpoint, e.g. `https://zabbix.example.com/api_jsonrpc.php`
- Auth — **either** `ZABBIX_TOKEN` (preferred, Zabbix 5.4+) **or** `ZABBIX_USER` +
  `ZABBIX_PASSWORD` (older versions)

**Connection:**
- `ZABBIX_VERIFY_SSL` (default `true`)
- `ZABBIX_TIMEOUT` (default `30` seconds)
- `ZABBIX_SKIP_VERSION_CHECK` (default `false`)

**Safety/behavior:**
- `READ_ONLY_MODE` (default `false`) — disables all write operations (create/update/delete)
  server-wide
- `DISABLED_TAGS` — comma-separated list to hide whole tool categories, e.g.
  `DISABLED_TAGS=host,user,maintenance`
- `LOG_LEVEL` (default `INFO`)

**Rate limiting:**
- `RATE_LIMIT_ENABLED` (default `false`), `RATE_LIMIT_MAX_REQUESTS` (default `60`),
  `RATE_LIMIT_WINDOW_MINUTES` (default `1`)

**Tool search transform (optional, for large tool catalogs):**
- `TOOL_SEARCH_ENABLED` (default `false`) — when true, `list_tools` returns two synthetic
  tools instead of the full catalog: `search_tools` (bm25 or regex) + `call_tool`
- `TOOL_SEARCH_STRATEGY` (`bm25` | `regex`), `TOOL_SEARCH_MAX_RESULTS` (default `5`)

**Sentry (optional, requires the `sentry` extra — `uv sync --extra sentry`):**
- `SENTRY_DSN` and friends (`SENTRY_TRACES_SAMPLE_RATE`, `SENTRY_SEND_DEFAULT_PII`,
  `SENTRY_ENVIRONMENT`, `SENTRY_RELEASE`, `SENTRY_PROFILE_*`, `SENTRY_ENABLE_LOGS`) — see
  `.env.example` for the full list. Handled in `src/zabbix_mcp/sentry_init.py`.

**Transport:**
- `MCP_TRANSPORT` (`stdio` default | `sse` | `http`)
- `MCP_HTTP_HOST` (default `127.0.0.1`; the Docker image overrides this to `0.0.0.0`),
  `MCP_HTTP_PORT` (default `8000`), `MCP_HTTP_BEARER_TOKEN` (optional, HTTP/SSE auth)

## Architecture

- **Entry point**: `src/zabbix_mcp/server.py` — builds the `FastMCP` instance
  (`mcp = FastMCP(...)`), wires the bearer-token auth provider (`StaticTokenVerifier`) when
  `MCP_HTTP_BEARER_TOKEN` is set, initializes Sentry, and registers tools via
  `zabbix_tools.register_tools(mcp, config)`. `main()` in this module is the
  `[project.scripts]` entry point.
- **Tool definitions**: `src/zabbix_mcp/zabbix_tools.py` — single large module,
  `register_tools(mcp, config)` registers **56 `@mcp.tool(...)` decorated async functions**,
  each tagged (e.g. `tags={"zabbix", "host", "read-only"}`) for `DISABLED_TAGS` filtering and
  read-only-mode gating.
- **Zabbix client / config parsing**: `src/zabbix_mcp/zabbix_client.py` — builds
  `ZabbixConfig`/transport config from environment variables (`get_zabbix_config_from_env`,
  `get_transport_config_from_env`), wraps `zabbix_utils` (async) for the actual JSON-RPC calls.
- **Models**: `src/zabbix_mcp/models.py` — Pydantic `ZabbixConfig` and related types.
- **Utilities**: `src/zabbix_mcp/utils.py`
- **Sentry init**: `src/zabbix_mcp/sentry_init.py` — no-ops if `SENTRY_DSN` unset.

### Tool Catalog (56 tools, by category)

| Category | Tools |
|---|---|
| API info | `api_version` |
| Host | `host_get`, `host_create`, `host_update`, `host_delete`* |
| Host group | `hostgroup_get`, `hostgroup_create`, `hostgroup_update`, `hostgroup_delete`* |
| Template | `template_get`, `template_create`, `template_update`, `template_delete`* |
| Item | `item_get`, `item_create`, `item_update`, `item_delete` |
| Trigger | `trigger_get`, `trigger_create`, `trigger_update`, `trigger_delete` |
| Problem/Event | `problem_get`, `event_get`, `event_acknowledge` |
| History/Trend | `history_get`, `trend_get` |
| User | `user_get`, `user_create`*, `user_update`*, `user_delete`* |
| Proxy | `proxy_get`, `proxy_create`, `proxy_update`, `proxy_delete` |
| Maintenance | `maintenance_get`, `maintenance_create`, `maintenance_update`, `maintenance_delete` |
| Action | `action_get`, `action_create`, `action_update`, `action_delete`* |
| Media type | `mediatype_get` |
| Graph/Discovery | `graph_get`, `discoveryrule_get`, `drule_get`, `itemprototype_get` |
| SLA/Service | `sla_get`, `service_get` |
| Script | `script_get`, `script_execute`* |
| User macro | `usermacro_get`, `usermacro_create`, `usermacro_delete` |
| Configuration | `configuration_export`, `configuration_import`* |

`*` = requires `confirm=true` (fork-specific gate, see below). Full per-tool descriptions are
in `README.md` under "Available Tools" — cross-check that section against
`src/zabbix_mcp/zabbix_tools.py` before trusting it blindly if the two ever drift (56 tools
counted directly via `grep -c "@mcp.tool" src/zabbix_mcp/zabbix_tools.py` as of `v0.5.2` +
fork commits through `43e275c`).

### Fork-Specific Confirmation Gates

Independent of `READ_ONLY_MODE`, these tools require an explicit `confirm=true` argument on
every call, even when write operations are otherwise enabled: `script_execute`, `user_create`,
`user_update`, `user_delete`, `host_delete`, `hostgroup_delete`, `template_delete`,
`action_delete`, `configuration_import`. Calling without `confirm=true` returns an error dict
(`{"error": "SECURITY: ... was not run. ..."}`) rather than touching the Zabbix API.
`user_update` has a second, narrower gate: setting `type=3` (Zabbix super admin) also requires
`confirm_privilege_escalation=true` on the same call — a routine profile edit cannot
accidentally grant super-admin. This is a code-level safety net layered on top of, not a
replacement for, `READ_ONLY_MODE`. See `README.md` → "Security & Safety Features" and
`src/zabbix_mcp/zabbix_tools.py` (search `confirm:` for the `Annotated[...]` parameter
definitions and the `if not confirm:` guard blocks) for exact behavior per tool.

### Read-Only Mode vs. Tag Filtering vs. Confirmation Gates

Three independent, stackable safety layers — don't conflate them:
1. `READ_ONLY_MODE=true` — hides/blocks *all* write tools server-wide.
2. `DISABLED_TAGS=host,user,...` — hides entire tool categories regardless of read/write.
3. Confirmation gates (above) — per-call opt-in for the 9 highest-risk tools, active even
   when `READ_ONLY_MODE=false` and the tag isn't disabled.

## Docker Builds — Two Dockerfiles, Different Purposes

- **`Dockerfile`** — builds the native MCP server image (`ghcr.io/mhajder/zabbix-mcp` upstream;
  this fork would publish to `ghcr.io/dwpdkp/zabbix-mcp` if/when its own `publish.yml` runs
  against `origin`). Multi-stage: `python:3.14-alpine3.24` builder installs deps via
  `uv sync --locked --all-extras --no-dev --no-editable`, runtime stage copies the `.venv`,
  runs as non-root `appuser` (uid/gid 1000), `ENTRYPOINT ["zabbix-mcp"]`. Defaults
  `MCP_TRANSPORT=http`, `MCP_HTTP_HOST=0.0.0.0`, `MCP_HTTP_PORT=8000`, exposes 8000.
  Healthcheck only fires when `MCP_TRANSPORT=http` (checks the port with `nc`); stdio-mode
  containers report healthy unconditionally.
- **`Dockerfile.mcpo`** — same builder stage, but additionally `uv pip install mcpo` and the
  final `CMD` wraps the server with **mcpo** (MCP-to-OpenAPI proxy), which exposes a REST/
  OpenAPI surface (`/openapi.json`) instead of raw MCP JSON-RPC — this is the variant meant
  for **Open WebUI** tool integration (see `home/MCP-Servers/CLAUDE.md`-adjacent homelab AI
  stack notes; Open WebUI consumes tools as OpenAPI, not native MCP). If `MCPO_API_KEY` is
  set, mcpo runs with `--api-key`; otherwise unauthenticated. Healthcheck polls
  `http://127.0.0.1:8000/openapi.json` via `wget`.

Neither Dockerfile is currently in use for Doug's own MCP consumption — both homelab Mac and
work-laptop `.mcp.json` entries run this server via local `uv run zabbix-mcp` against the
cloned repo directory (see below), not via Docker. The Dockerfiles matter if this ever gets
deployed as a standing service (e.g. behind Open WebUI/mcpo on the homelab) rather than
spawned per-Claude-Code-session.

## Consumption: Homelab Mac vs. Work Laptop

This single codebase talks to **two separate, unrelated Zabbix instances** depending on which
machine's `.mcp.json` is loaded — same server code, different `env` block.

**Homelab Mac** (`/Users/dougpearson/Downloads/Dev-Stuff/.mcp.json`, `zabbix` entry):
```json
"zabbix": {
  "command": "/Users/dougpearson/.local/bin/uv",
  "args": ["--directory", "<this repo path>", "run", "zabbix-mcp"],
  "env": {
    "ZABBIX_URL": "http://10.0.1.103/zabbix/api_jsonrpc.php",
    "ZABBIX_TOKEN": "<token>",
    "PATH": "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"
  }
}
```
Token-based auth against the homelab Zabbix instance (10.0.1.103). Credential tracked in
enigma key `22169` per the top-level `Dev-Stuff/CLAUDE.md` "Zabbix MCP Server (Homelab)"
section — **note the token is currently hardcoded directly in `.mcp.json`'s `env` block
rather than pulled from enigma at invocation time**; if `.mcp.json` is ever shared or
committed, that token is exposed in plaintext.

**Work laptop** — per the machine-context table in the user's global `~/.claude/CLAUDE.md`,
the work laptop's own `.mcp.json` points this same server at the **production** Zabbix
instance (`10.100.0.45`) using **user/password** auth (`mcp-service` account, enigma key
`22177` on that machine — enigma key numbers are not shared between machines, so `22177`
means something different on the Mac). This document was written from the Mac checkout only;
the work laptop's exact `.mcp.json` block was not read to produce this doc — verify it
directly on that machine rather than assuming it mirrors the Mac's structure.

## Critical Gotcha: FastMCP Version Pin — Likely Stale for This Repo

The top-level `Dev-Stuff/CLAUDE.md` documents a homelab-wide gotcha: *"FastMCP 3.0.x breaks
tool registration with Claude Code... pin `fastmcp==2.14.5` and `zabbix-mcp==0.2.0` in
`.mcp.json`, invoked via `uvx --with fastmcp==2.14.5 --with zabbix-mcp==0.2.0 ...`."*

**That pin does not match this repo's current state and is not how this repo is actually
invoked:**
- `pyproject.toml` requires **`fastmcp>=3.0.0,<4`** (not `2.14.5`) — `zabbix_mcp/server.py`
  imports `fastmcp.server.transforms.search.BM25SearchTransform` and
  `fastmcp.server.middleware.rate_limiting.SlidingWindowRateLimitingMiddleware`, both FastMCP
  3.x-only APIs. Installing `fastmcp==2.14.5` against this codebase would fail outright, not
  just silently misbehave.
- `CHANGELOG.md` v0.3.0 (2026-04-09) explicitly fixes middleware for **FastMCP 3.0
  compatibility**: *"remove deprecated `Component.disable()` calls for FastMCP 3.0
  compatibility (#11)"* — i.e. this project moved *toward* FastMCP 3.x on purpose, upstream.
- The Mac's actual `.mcp.json` entry (above) invokes `uv run zabbix-mcp` **against this local
  cloned directory** — it does not use `uvx --with fastmcp==2.14.5 --with zabbix-mcp==0.2.0`
  at all. `uv run` resolves dependencies from this repo's own `pyproject.toml`/`uv.lock`
  (locked to `fastmcp>=3.0.0,<4`), so the old pin is architecturally irrelevant here even if
  it were still needed.

**Conclusion (best available evidence, not independently reproduced against Claude Code):**
the documented `fastmcp==2.14.5` / `zabbix-mcp==0.2.0` gotcha appears to predate this repo's
current fork-and-local-`uv run` setup, or applied to a different, older invocation method
(possibly a bare `uvx zabbix-mcp` pulling upstream PyPI releases pre-v0.3.0). If tool
discovery breaks again on this server, do not reflexively re-apply the old pin — check
`fastmcp` version compatibility against whatever `fastmcp` release is current at the time,
since this repo is now hard-pinned to the 3.x line by its own imports. **Recommend updating
the "Zabbix MCP Server (Homelab)" section of the top-level `Dev-Stuff/CLAUDE.md`** to either
drop this gotcha or scope it explicitly to a different/older setup — leaving it as-is risks
someone "fixing" a future tool-discovery issue by pinning to a version this codebase cannot
even import.

## Versioning (Commitizen)

Version bumps, tags, and `CHANGELOG.md` are managed by **commitizen**
(`[tool.commitizen]` in `pyproject.toml`), driven by Conventional Commit messages —
`fix:` → patch, `feat:` → minor, breaking change → major (`major_version_zero = true`, so
0.x breaking changes still bump minor, not major, until a 1.0 cut). `tag_format = "v$version"`,
`version_provider = "uv"` (bumps `pyproject.toml` via `uv`), `version_files = ["server.json"]`
(also bumped in lockstep). Current version: **`0.5.2`** (`v0.5.2`, 2026-07-06). Do not hand-edit
version strings in `pyproject.toml` or `server.json`; use `cz bump` (or let the `release.yml`
workflow handle it) so `CHANGELOG.md` stays in sync.

## Testing

```bash
uv run pytest                  # tests/ — currently just test_utils.py, see gap noted above
uv run pytest --cov=src/       # with coverage
```

## Git Workflow / CI

`.github/workflows/`: `pre-commit.yml` (lint/format/type-check on PRs, via `prek` not the
Python `pre-commit` package — see `.pre-commit-config.yaml`'s
`minimum_prek_version`/`default_install_hook_types`), `release.yml` (commitizen bump + tag),
`publish.yml` (PyPI + Docker image publish on tag). `uv run prek install` sets up
`pre-push`/`commit-msg`/`pre-commit` hooks locally to match CI.

---

## Document History

| Date | Author | Changes |
|------|--------|---------|
| 2026-08-12 | Doug Pearson | Initial creation |
