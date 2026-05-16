# adingest

Single-file CLI for the [ad-ingest](https://api.creational.ai) service. Fires ingests, polls jobs, renders an 8-category verdict per source.

* `adingest ingest 2026-05-12 --poll` — POST + poll + verdict-classified table, in one command
* `adingest range 2026-05-10..2026-05-12 --poll` — backfill loop, one job per day
* `adingest status <JOB_ID>` — re-render the table for an existing job
* `adingest upgrade` — atomic self-replace via service manifest + sha256-verified bytes from this mirror
* 8-category verdict classifier (`✅ healthy-fresh`, `🚨 silent-drop`, `⚠️ partial-dedup`, …) — one-glance health per source and per app-platform rollup
* PEP 723 single-file Python script — [uv](https://docs.astral.sh/uv/) resolves inline deps on first run; no `pip install`, no virtualenv

## Table of Contents

- [Getting started](#getting-started)
- [Configuration](#configuration)
- [`adingest ingest`](#adingest-ingest)
- [`adingest range`](#adingest-range)
- [`adingest status`](#adingest-status)
- [Verdict vocabulary](#verdict-vocabulary)
- [`adingest upgrade`](#adingest-upgrade)
- [Authentication and errors](#authentication-and-errors)
- [About this repository](#about-this-repository)
- [License](#license)

## Getting started

Requires Python 3.11+ and [uv](https://docs.astral.sh/uv/) on `PATH`. `uv` installs the script's inline dependencies (`httpx`, `rich`, `click`) on first run — no separate `pip install` step.

Install the latest release into `~/.local/bin/`:

```bash
mkdir -p ~/.local/bin
curl -fsSL https://github.com/creational-ai/adingest-cli/releases/latest/download/adingest \
  -o ~/.local/bin/adingest
chmod +x ~/.local/bin/adingest
adingest --version
```

```
adingest, version 0.1.0
```

Point it at a service:

```bash
export INGEST_API_URL=https://api.creational.ai
export INGEST_API_TOKEN=<bearer token from the service operator>
export APPS_JSON='[{"app_id":"com.acme.app","platform":"ios"}]'
```

First use — stub mode exercises the full wire path without upstream credentials:

```bash
$ adingest ingest 2026-05-12 --stub --poll
Job ing_01KRJ7JA498C40JK1RMA0XHZNC  status=complete  1.9s
Source: stub
┏━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━━━━━┓
┃ Key          ┃ fetched ┃ invalid ┃ skipped_dedup ┃ inserted ┃ orphan ┃ %uid   ┃ verdict          ┃
┡━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━━┩
│ source total │ 3       │ 0       │ 0             │ 3        │ 0      │ 100%   │ ✅ healthy-fresh │
└──────────────┴─────────┴─────────┴───────────────┴──────────┴────────┴────────┴──────────────────┘
GA4: attempted=3  sent=3  skipped_orphan=0  failed=0
```

## Configuration

Three environment variables (or per-invocation flags). The CLI also reads `.env.client` in the current working directory if present.

| Variable | Required | Purpose |
|---|---|---|
| `INGEST_API_URL` | yes | Service base URL (e.g. `https://api.creational.ai` or `http://localhost:8000`) |
| `INGEST_API_TOKEN` | yes | Bearer token issued by the service operator |
| `APPS_JSON` | yes for `ingest` / `range` | JSON array of `{app_id, platform}` objects — the apps to ingest for |

`.env.client` example:

```bash
INGEST_API_URL=https://api.creational.ai
INGEST_API_TOKEN=<bearer>
APPS_JSON='[{"app_id":"com.acme.app","platform":"ios"},{"app_id":"com.acme.app","platform":"android"}]'
```

Every subcommand accepts `--url`, `--token`, and `--apps FILE` (a JSON file path that overrides `APPS_JSON`).

> Credentials live in the service's request body for one call and are dropped after the response. The service is stateless — nothing about your account is persisted on it.

## `adingest ingest`

POST one ingest for a single `EVENT_DATE` (ISO date, `YYYY-MM-DD`). Returns immediately with a `job_id`; pass `--poll` to wait until terminal and render the table.

```bash
$ adingest ingest 2026-05-12 --poll
Job ing_01KRJ7JA498C40JK1RMA0XHZNC  status=complete  4.9s
Source: levelplay
│ source total │ 955     │ 0       │ 0             │ 955      │ 6      │ 99.4%  │ ✅ healthy-fresh │
GA4: attempted=955  sent=949  skipped_orphan=6  failed=0
```

Same date re-run — dedup path:

```bash
$ adingest ingest 2026-05-12 --poll
Job ing_01KRJ7AM15A2Z4C5M91RXW6STF  status=complete  1.8s
Source: levelplay
│ source total │ 955     │ 0       │ 955           │ 0        │ 0      │ —      │ ✅ healthy-dedup │
GA4: attempted=0  sent=0  skipped_orphan=0  failed=0
```

> Re-running the same `event_date` is always safe. The service's per-source dedup tuple turns the second call into a `✅ healthy-dedup` no-op.

| Flag | Behavior |
|------|----------|
| `--poll` | Poll `GET /v1/jobs/{id}` every 2s until terminal (max 120s); render the table |
| `--source NAME` | Source name (default: `levelplay`) |
| `--stub` | Alias for `--source stub` — no upstream creds required |
| `--apps FILE` | JSON file with `apps[]`; overrides `APPS_JSON` env |
| `--url URL` | Override `INGEST_API_URL` |
| `--token TOKEN` | Override `INGEST_API_TOKEN` |
| `--json` | Emit raw JSON (with `verdicts: [...]` merged in at every scope) instead of the table |
| `--quiet` | Suppress non-verdict output |

## `adingest range`

Fire one ingest per day in `START..END` (inclusive). Useful for backfills.

```bash
$ adingest range 2026-05-10..2026-05-12 --poll
Job ing_01KRJ79J5DQPX2YANKD6PWTHMP  status=complete  6.7s
│ source total │ 1,394   │ 0       │ 0             │ 1,394    │ 594    │ 57.4%  │ ✅ healthy-fresh │
GA4: attempted=1394  sent=800  skipped_orphan=594  failed=0

Job ing_01KRJ79T2CS79C98KRHS758TXX  status=complete  4.6s
│ source total │ 1,412   │ 0       │ 0             │ 1,412    │ 378    │ 73.2%  │ ✅ healthy-fresh │
GA4: attempted=1412  sent=1034  skipped_orphan=378  failed=0

Job ing_01KRJ7AM15A2Z4C5M91RXW6STF  status=complete  5.2s
│ source total │ 955     │ 0       │ 0             │ 955      │ 6      │ 99.4%  │ ✅ healthy-fresh │
GA4: attempted=955  sent=949  skipped_orphan=6  failed=0
```

Accepts every flag `ingest` accepts, plus:

| Flag | Behavior |
|------|----------|
| `--continue-on-failure` | Do not abort on the first `failed` job — fire every day in the range, then exit 1 if any failed |

> Default behavior aborts on the first `failed` job. Use `--continue-on-failure` for unattended backfills where you want all days attempted regardless.

## `adingest status`

Re-render the table for an existing `JOB_ID`. Jobs persist for the lifetime of the service process; there is no historical query across restarts.

```bash
$ adingest status ing_01KRJ7JA498C40JK1RMA0XHZNC
Job ing_01KRJ7JA498C40JK1RMA0XHZNC  status=complete  4.9s
Source: levelplay
│ source total │ 955     │ 0       │ 0             │ 955      │ 6      │ 99.4%  │ ✅ healthy-fresh │
GA4: attempted=955  sent=949  skipped_orphan=6  failed=0
```

| Flag | Behavior |
|------|----------|
| `--url URL` / `--token TOKEN` | Override env defaults |
| `--json` | Emit raw JSON (with verdicts) |
| `--quiet` | Suppress non-verdict output |

## Verdict vocabulary

Each source gets exactly one verdict per job, plus per-`(platform, app_id)` verdicts under the `by_app_platform` rollup. Precedence is two-axis: tier (`🚨 > ⚠️ > ✅`), then table order within tier.

| Tier | Verdict | Triggered when |
|---|---|---|
| ✅ | `healthy-fresh` | `rows_inserted == rows_fetched` and `rows_fetched > 0` |
| ✅ | `healthy-dedup` | `rows_skipped_dedup == rows_fetched` and `rows_fetched > 0` |
| ⚠️ | `partial-dedup` | Some rows inserted, some deduped |
| 🚨 | `empty-upstream` | `rows_fetched == 0` — upstream returned nothing (likely a credential or scope problem) |
| 🚨 | `invalid-heavy` | `rows_invalid` is a significant fraction of `rows_fetched` (adapter-drop heavy) |
| 🚨 | `writer-orphan-heavy` | `rows_inserted_orphan` is a significant fraction of `rows_inserted` (BQ landed but no `user_id`) |
| 🚨 | `ga4-orphan-heavy` | `ga4_events_skipped_orphan` is a significant fraction of `ga4_events_attempted` |
| 🚨 | `silent-drop` | Counter-balance violation — `rows_fetched != rows_invalid + rows_skipped_dedup + rows_inserted`. Means a row vanished between fetch and BQ without classification. |

> The tier emoji (`✅` / `⚠️` / `🚨`) is render-only — it shows in the table output. The JSON `verdicts` field carries just the bare name (e.g. `["healthy-fresh"]`).

`--json` emits the same verdicts under `sources[<src>].verdicts: [...]` and `sources[<src>].by_app_platform[<key>].verdicts: [...]`. Pipe into `jq` for cron alerting — anything not in the healthy set is an alert:

```bash
adingest ingest "$(date -u -v-1d +%F)" --poll --json \
  | jq -e '[.sources[].verdicts[]] - ["healthy-fresh", "healthy-dedup"] | length > 0' \
  > /dev/null \
  && echo "ALERT: non-healthy verdict — page oncall"
```

## `adingest upgrade`

Atomic self-replace. The CLI asks the service which version it should be running, then fetches commit-SHA-pinned bytes from this mirror, sha256-verifies them, and `mv`s them over its own binary.

```bash
$ adingest upgrade
Resolving server at https://api.creational.ai...
Already at 0.1.0
```

When a newer version is available:

```bash
$ adingest upgrade
Resolving server at https://api.creational.ai...
Server recommends 0.2.0; you are at 0.1.0
Fetching https://raw.githubusercontent.com/creational-ai/adingest-cli/abc123.../adingest
Verifying sha256... ok
Replacing /home/user/.local/bin/adingest...
Upgraded 0.1.0 -> 0.2.0
```

The flow:

1. `GET <url>/v1/cli/version` with Bearer — returns `{product, server_version, cli_recommended, cli_min_compatible, download_url, cli_sha256}`
2. Abort if `manifest.product != "adingest"` — defends against pointing the CLI at a different Creational product's server
3. Compare `__version__` to `cli_recommended` / `cli_min_compatible`; pick one of four verdicts (equal / local-newer / local-older-compatible / local-older-required)
4. For `older-required`: fetch `download_url` (commit-SHA-pinned, no auth needed since the repo is public), sha256-verify against `cli_sha256`, `chmod +x`, atomic-`mv` into place

> `download_url` is pinned to a commit SHA (40-char hex), not a tag. Mirror tag-moves cannot redirect to different bytes. `cli_sha256` is belt-and-suspenders — if the bytes don't match, the CLI refuses to replace itself.

| Flag | Behavior |
|------|----------|
| `--url URL` | Override `INGEST_API_URL` |
| `--token TOKEN` | Override `INGEST_API_TOKEN` |

## Authentication and errors

Every service endpoint requires `Authorization: Bearer <token>`. The CLI sends this automatically. All 4xx/5xx responses come back as Problem+JSON envelopes (`type` / `title` / `status` / `detail` / `instance` / `next_action`), which the CLI rewrites into a one-line classified error and exits non-zero:

```bash
$ adingest ingest not-a-date --stub --poll
Error: HTTP 422 POST /v1/ — Validation failed for 1 field(s): event_date

$ adingest status ing_DOESNOTEXIST
Error: HTTP 404 GET /v1/jobs/ing_DOESNOTEXIST — Job 'ing_DOESNOTEXIST' not found

$ adingest ingest 2026-05-12 --token wrong
Error: HTTP 401 POST /v1/ — Bearer token does not match the expected value.
```

| Exit | Meaning |
|------|---------|
| 0 | Job reached a terminal status (`complete` / `partial` / `failed`); table rendered with verdict tier |
| 1 | HTTP error (non-2xx, network failure, auth mismatch) rendered as `Error: …`; or `range` saw a `failed` job and `--continue-on-failure` was not set |

| HTTP | Trigger |
|------|---------|
| 401 | Missing `Authorization` header, non-`Bearer` scheme, empty token, or token mismatch |
| 404 | `GET` for an unknown `job_id` |
| 422 | Body validation failure — `errors[]` carries per-field Pydantic detail |

## About this repository

This repo is the **public binary mirror** for the `adingest` CLI. It exists so consumers (and the `adingest upgrade` flow) have a public, commit-SHA-pinnable HTTPS surface to fetch from.

- Holds the stamped CLI binary plus a small set of release artifacts (this README, `LICENSE`, etc.) — nothing else
- Each `vX.Y.Z` tag corresponds to a service release of the same version; the same bytes are attached as the release asset named `adingest`
- `download_url` in the `/v1/cli/version` manifest pins to a commit SHA in this repo — even if a tag is moved, an older `cli_sha256` will refuse to install bytes from the new commit
- Contents are produced by an automated release pipeline; direct edits are not the development workflow

## License

Proprietary — Creational.ai. All rights reserved.
