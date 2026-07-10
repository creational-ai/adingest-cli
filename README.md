# adingest

Single-file CLI for the [ad-ingest](https://api.creational.ai/adingest) service. Fires ingests, polls jobs, renders a 9-category verdict per source.

* `adingest ingest 2026-05-12 --poll` — POST + poll + verdict-classified table, in one command
* `adingest range 2026-05-10..2026-05-12 --poll` — backfill loop, one job per day
* `adingest status <JOB_ID>` — re-render the table for an existing job
* `adingest upgrade` — atomic self-replace via service manifest + sha256-verified bytes from this mirror
* `adingest fetch-only 2026-05-12 --json` — pipeable canonical revenue rows on stdout, **zero writes** (no BQ, no GA4, no dedup); validate a caller's config with no destination credentials
* 9-category verdict classifier (`✅ healthy-fresh`, `🚨 silent-drop`, `✅ fetch-only`, …) — one-glance health per source and per app-platform rollup
* PEP 723 single-file Python script — [uv](https://docs.astral.sh/uv/) resolves inline deps on first run; no `pip install`, no virtualenv

## Table of Contents

- [Getting started](#getting-started)
- [Configuration](#configuration)
- [`adingest ingest`](#adingest-ingest)
- [`adingest range`](#adingest-range)
- [`adingest status`](#adingest-status)
- [`adingest fetch-only`](#adingest-fetch-only)
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
adingest, version 0.2.2
```

Point it at a service — `adingest init <slug>` bootstraps `~/.adingest/<slug>/config.toml` + `.env` with template placeholders; edit those two files (see [Configuration](#configuration) below for the layout):

```bash
$ adingest init dev
Active project: dev

$ ${EDITOR:-vi} ~/.adingest/dev/config.toml    # set [ingest_api].url + [[apps]]
$ ${EDITOR:-vi} ~/.adingest/dev/.env           # set INGEST_API_TOKEN=<bearer>
```

First use — stub mode exercises the full wire path without upstream credentials. ⚠ Stub adapters register only when the service runs with `env="dev"` — point `[ingest_api].url` at a local dev server (e.g. `http://localhost:8000`) for this flow; `--stub` errors against the production mount:

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

The CLI reads config from `~/.adingest/` — outside any project repo. Bootstrap a new project with `adingest init <slug>`; switch the active one with `adingest use <slug>`; inspect with `adingest projects` / `current` / `config show`.

```
~/.adingest/
├── config.toml                # global defaults (service URL + bearer token)
├── state.toml                 # active-project pointer (mutated by `adingest use`)
└── <slug>/
    ├── config.toml            # per-project structural config (BQ, LP, apps[])
    └── .env                   # per-project secrets (mode 0600; CLI auto-sources)
```

Walkup commands (the `ls` / `cd` / `pwd` / `mkdir` for projects):

```bash
$ adingest projects             # list configured projects (* marks active)
* hexario
  staging

$ adingest use staging          # switch active project (persists across shells)
Active project: staging  (was: hexario)

$ adingest current              # what's active right now
staging

$ adingest init <slug>          # bootstrap a new project; sets active if no current active

$ adingest config show          # render the resolved config for the active project
```

A minimal per-project `config.toml`:

```toml
[ingest_api]
url   = "https://api.creational.ai/adingest"   # the /adingest mount path is required; also settable globally in ~/.adingest/config.toml
token = "${INGEST_API_TOKEN}"           # env-interpolated from ~/.adingest/<slug>/.env

[[apps]]
app_id   = "com.acme.app"
platform = "ios"
```

And the matching `~/.adingest/<slug>/.env` (mode 0600 — CLI warns on looser perms):

```bash
INGEST_API_TOKEN=<bearer token from the service operator>
```

**Selection precedence** (highest wins): `--project <slug>` flag → `ADINGEST_PROJECT` env var → `~/.adingest/state.toml` active → error with a remediation hint. `state.toml` is read once at process startup and cached for that process's lifetime — a concurrent `adingest use <other>` in another shell can't redirect an in-flight ingest. State writes are atomic (`fsync` + `os.replace`); a crash mid-`use` never leaves the file half-written.

**Parallel runs**: two `adingest` instances can run in parallel from any cwd:

```bash
$ adingest --project hexario ingest 2026-05-27 & \
  adingest --project staging ingest 2026-05-27 &
```

Use the always-explicit `--project <slug>` form for cron / scripted invocations so interactive `use` of another slug never redirects them.

Every subcommand accepts `--url`, `--token`, and `--project <slug>` (overrides cascade). `ingest` / `range` / `fetch-only` additionally accept `--apps FILE` (a JSON file path that overrides the per-project TOML `[[apps]]` list).

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
| `--apps FILE` | JSON file with `apps[]`; overrides the per-project TOML `[[apps]]` list |
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

## `adingest fetch-only`

Run the same faithful pipe as `ingest` — adapter fetch → validation → match remap — but **stop before the sinks**: no BigQuery write, no GA4 fan-out, no dedup query. The day's canonical revenue rows come back as pipeable JSON on stdout; counters and verdicts go to stderr. Two operator wins: pull a day of rows without provisioning any destination, and validate a caller's config (LP creds, per-app `match` remap, row validity) end-to-end with **zero writes and zero destination credentials**.

It **always polls** (rows only exist at the terminal job, so a fire-and-forget fetch-only is meaningless) — there is no `--poll` flag.

```bash
$ adingest fetch-only --project hexario 2026-07-06 --json > rows.json        # rows to a file
$ adingest fetch-only --project hexario 2026-07-06 --json | jq 'length'      # count the day's rows
$ adingest fetch-only --project hexario 2026-07-06                           # human summary, no rows dump
```

> **⚠ Local-only until `core-job-registry-rds` lands.** The service's in-process job registry can silently lose the deliverable on the multi-instance production gateway — an unpinned instance may not hold the job the CLI polls back. Point `fetch-only` at a **local dev service** until the registry is externalized: a bare `--project hexario` resolves to `https://api.creational.ai/adingest` (prod) via the per-slug config, so pass `--url http://localhost:8000` to force local routing (and `--stub` to exercise the wire path without upstream LP creds).

### Output contract (the piping surface)

`--json` splits the two streams so redirection stays clean:

- **stdout** — a bare JSON array of the day's rows (all sources concatenated; each row self-describes via its `source` field). `> rows.json` and `| jq …` both work with zero non-JSON contamination.
- **stderr** — the same counters envelope `ingest --json` emits (verdicts merged), **minus the rows** (`rows_by_source` is stripped so the array is not duplicated into the envelope). `2>counters.json | jq` yields machine-readable verdicts — the only programmatic verdict channel in this mode, since stdout is rows-only by contract.

```bash
$ adingest fetch-only --project hexario --url http://localhost:8000 --stub 2026-07-06 --json \
    > rows.json 2> counters.json
$ jq 'length' rows.json                       # row count
$ jq '.sources[].verdicts' counters.json      # ["fetch-only"] on a healthy slug
```

`--json --quiet` is **byte-clean**: the stderr envelope is suppressed too, so stdout carries pure rows and **stderr is empty**. Use this in cron/script pipelines that treat any stderr output as a failure signal:

```bash
$ adingest fetch-only --project hexario --url http://localhost:8000 --stub 2026-07-06 --json --quiet \
    | jq 'length'
```

Without `--json`, human mode prints a per-source counters summary plus a rows-per-`(platform, app_id)` breakdown derived CLI-side from the returned rows — Use Case 2's routing check at a glance — and **never dumps the rows**:

```
Job ing_01KS7…  status=complete  1.9s
Source: levelplay  fetched=1240  invalid=0  verdict=✅ fetch-only
┏━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━┓
┃ platform ┃ app_id                       ┃ rows ┃
┡━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━┩
│ android  │ com.mochibits.hexario.google │ 812  │
│ ios      │ com.mochibits.hexario.apple  │ 428  │
└──────────┴──────────────────────────────┴──────┘
```

The writer-boundary columns (`skipped_dedup` / `inserted` / `orphan` / `%uid`) and the `GA4:` summary line are omitted in fetch-only mode — they would all read as a misleading `0` in a mode that provably never writes or publishes.

### Exit codes (stricter than `ingest`)

| Terminal status | `fetch-only` exit | stdout |
|---|---|---|
| `complete` (incl. empty day) | 0 | rows array (`[]` on an empty day — empty is a valid answer, not an error) |
| `partial` (some sources failed) | non-zero | **surviving sources' rows still emitted** — `> rows.json \|\| alert` gets both the salvageable data and the alarm |
| `failed` (all sources failed) | non-zero | `[]` |
| non-terminal poll timeout | non-zero | `[]` |

This is deliberately **stricter than `ingest` / `range`**, which exit `0` even on a `failed` job (their alarm is the `🚨` verdict tier in the rendered table, not the exit code). `fetch-only` makes the exit code the alarm so `> rows.json || alert` pipelines work directly. Unifying `ingest`'s exit behavior is out of scope — the divergence is per-verb by design.

> Under `--json --quiet`, a `partial`/`failed` run exits non-zero with **no in-band diagnostic** — the stderr envelope that names the failed source is suppressed. The exit code is the only alarm; re-run without `--quiet` to see which source failed.

### Flags

| Flag | Behavior |
|------|----------|
| `--source NAME` | Source name (default: `levelplay`) |
| `--stub` | Alias for `--source stub` — no upstream creds required (dev service only) |
| `--apps FILE` | JSON file with `apps[]`; overrides the per-project TOML `[[apps]]` list |
| `--url URL` / `--token TOKEN` | Override `INGEST_API_URL` / `INGEST_API_TOKEN` |
| `--json` | Rows array → stdout, counters envelope → stderr |
| `--quiet` | Suppress the stderr envelope (with `--json` → byte-clean stdout) |

A **single ISO date only** — a `START..END` range string is rejected with a `UsageError` pointing at `range` (fetch-only has no multi-day loop). For multiple days, loop and merge the arrays:

```bash
$ for d in 2026-07-01 2026-07-02 2026-07-03; do
    adingest fetch-only --project hexario --url http://localhost:8000 --stub "$d" --json --quiet
  done | jq -s 'add'          # concatenate each day's array into one
```

> **Retention caveat.** fetch-only reads the same LevelPlay ILR source as `ingest` — LP retains only ~30 days of impression-level revenue. Dates older than that window return an upstream error/empty (surfaced as `adapter_fetch_failed`, not a silent `[]`); deep history still comes from a LevelPlay dashboard export. fetch-only is a *tap before the sinks*, not a history store.

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
| ✅ | `fetch-only` | A `fetch-only` job that fired neither `empty-upstream` nor `invalid-heavy` — the healthy fetch-only outcome: rows fetched and valid, nothing written by design. |

> The tier emoji (`✅` / `⚠️` / `🚨`) is render-only — it shows in the table output. The JSON `verdicts` field carries just the bare name (e.g. `["healthy-fresh"]`).

> **fetch-only mode.** On a `fetch-only` job only the two *pre-sink* triggers evaluate — `empty-upstream` and `invalid-heavy` (both computed from `rows_fetched` / `rows_invalid`, which fetch-only populates for real). The sink-boundary triggers (`healthy-fresh`, `healthy-dedup`, `partial-dedup`, `writer-orphan-heavy`, `ga4-orphan-heavy`, `silent-drop`) are suppressed — they are meaningless with no writer or publisher, and `silent-drop` in particular would false-positive on *every* successful fetch-only run (`rows_inserted == 0` by construction). When neither pre-sink trigger fires, the verdict is `fetch-only` ✅; when a caller's match rules drop rows the operator still sees `invalid-heavy` (Use Case 2's broken-config signal). The `fetch-only` slug deliberately breaks the `healthy-*` naming pattern — verb↔wire↔verdict name parity wins.

`--json` emits the same verdicts under `sources[<src>].verdicts: [...]` and `sources[<src>].by_app_platform[<key>].verdicts: [...]`. Pipe into `jq` for cron alerting — anything not in the healthy set is an alert:

```bash
adingest ingest "$(date -u -v-1d +%F)" --poll --json \
  | jq -e '[.sources[].verdicts[]] - ["healthy-fresh", "healthy-dedup", "fetch-only"] | length > 0' \
  > /dev/null \
  && echo "ALERT: non-healthy verdict — page oncall"
```

> `fetch-only` is in the healthy set so the *same* recipe works whether it polls an `ingest` or a `fetch-only` job — without it, every successful fetch-only run would false-page. One channel difference: for a `fetch-only` job the verdict envelope rides **stderr** (stdout is rows-only), so feed the recipe the envelope stream — `adingest fetch-only … --json 2>&1 >/dev/null | jq -e …`.

## `adingest upgrade`

Atomic self-replace. The CLI asks the service which version it should be running, then fetches commit-SHA-pinned bytes from this mirror, sha256-verifies them, and `mv`s them over its own binary.

```bash
$ adingest upgrade
Resolving server at https://api.creational.ai/adingest...
Already at 0.2.2
```

When a newer version is available:

```bash
$ adingest upgrade
Resolving server at https://api.creational.ai/adingest...
Server recommends 0.3.0; you are at 0.2.2
Fetching https://raw.githubusercontent.com/creational-ai/adingest-cli/abc123.../adingest
Verifying sha256... ok
Replacing /home/user/.local/bin/adingest...
Upgraded 0.2.2 -> 0.3.0
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
