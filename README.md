# Cardbe Plugin API

This repository owns the versioned WIT contract consumed by Cardbe and independent plugins.

- WIT package: `cardbe:plugin@1.0.0`
- World: `importer`

Cardbe pins this repository as a Git submodule. Changes here require updating the submodule pointer in Cardbe after API compatibility review.

## Scope and consumption

[`wit/plugin.wit`](wit/plugin.wit) is the source of truth for the Component Model boundary. This repository contains the contract, not the host runtime, plugin implementations, or a language-specific SDK. Generate host and guest bindings from the same pinned revision and select the `importer` world.

The guest imports `cardbe:plugin/host@1.0.0` and the shared `cardbe:plugin/types@1.0.0` types, and exports `run`. The current host supplies no WASI capabilities; plugins must use `host.http-get` for network access and `host.log` for diagnostics. Do not rely on files, environment variables, clocks, or direct sockets being available.

## Run contract

`run(context)` returns `result<changeset, plugin-error>`. Each invocation gets a fresh runtime store; persist incremental state through the cursor rather than guest memory.

| Input | Meaning |
| --- | --- |
| `instance-id` | Identity of the configured plugin instance. |
| `run-id` | Identity of this invocation. |
| `config-json` | JSON object containing the instance's non-secret configuration; keys are plugin-specific. Credentials stay with the host. |
| `board-json` | JSON array of the target board's columns. Current columns contain `id`, `name`, `color`, `sort_order`, and `tasks`; use column `id` for task destinations. This is not a board object. |
| `cursor` | Opaque checkpoint from the last committed run, or `none`. |

The JSON strings are not structurally typed by WIT. Parse and validate the fields you need, tolerate unrelated fields, and coordinate JSON shape changes with consumers just like typed API changes.

Each returned task contains all fields below; an empty string is a value, not an omitted update.

| Task field | Meaning |
| --- | --- |
| `external-key` | Stable source identity. Namespace it (for example, `service:project:123`) and keep it stable across runs. |
| `title`, `description` | Imported text. Title must contain non-whitespace text. |
| `url` | Source URL, or an empty string. |
| `state` | Source-defined state string; WIT defines no enum or automatic column mapping. |
| `labels` | Imported labels; each must contain non-whitespace text. |
| `updated-at` | Source-defined timestamp string, or an empty string; WIT defines no timestamp format. |
| `column-id` | Existing destination column ID from `board-json`. |

`changeset.tasks` contains upserts, not a full replacement or deletion list. Omitted tasks remain on the board. The current host preserves user overrides and does not recreate deliberately removed imported tasks. Return each external key once; the current host takes the first duplicate and rejects a collision with another instance of the same plugin on the same board.

Tasks and the returned cursor commit atomically. A guest error, trap, cancellation, validation failure, or failed commit does not advance the cursor. On success, `none` clears the checkpoint; return the input cursor to retain it. An empty task list can still update the cursor.

## Host capabilities and errors

`http-get(url)` returns status, filtered headers, and raw body bytes. A non-success HTTP status (such as 404 or 429) is still an `ok` response; plugins must inspect it. Transport, permission, cancellation, and resource failures return diagnostic strings. The current host also fails the run after a failed host request, even if the guest catches that error.

The current Cardbe host allows only approved HTTPS origins on port 443, rejects private addresses and redirects, and attaches credentials itself. It exposes only `link`, `retry-after`, `x-ratelimit-remaining`, `x-ratelimit-reset`, and `content-type` response headers when valid and within bounds. Never put credentials in URLs, cursors, returned task data, or diagnostics.

`log(code, count)` records a declared diagnostic code and a numeric count. Error and log codes must be 1–64 ASCII characters using `A-Z`, `0-9`, or `_`, and appear in the plugin package's `error_codes` or `log_codes` respectively. Unknown logs are discarded; unknown guest errors become `PLUGIN_FAILED`. `retry-after-seconds` is a scheduling hint; the current host clamps it to 60–86,400 seconds.

## Current Cardbe host limits

These are runtime policies, not guarantees encoded in WIT. Hosts may impose stricter limits. String lengths below are UTF-8 bytes.

| Resource | Limit |
| --- | --- |
| Component / linear memory | 16 MiB / 128 MiB |
| Run duration / fuel | 60 seconds / 500,000,000 fuel units |
| Configuration / board JSON | 64 KiB / 2 MiB |
| HTTP requests per run | 100 |
| HTTP body per response / total per run | 2 MiB / 32 MiB |
| Recorded logs per run | 100 |
| Tasks / serialized task output | 2,000 / 8 MiB |
| Cursor | 4,096 bytes |
| External key / title / description | 512 / 4,096 / 262,144 bytes |
| URL / state / updated-at | 4,096 / 64 / 128 bytes |
| Labels per task / each label | 100 / 256 bytes |

## Validation and changes

With `wasm-tools` installed, run from the repository root:

```sh
wasm-tools component wit wit
```

CI parses and encodes the package, then validates the encoded WIT binary. This checks WIT validity; it does not prove host/guest compatibility or JSON behavior.

For each contract change:

1. Review host bindings, guest bindings, fixtures, and JSON consumers together.
2. Keep released package versions immutable for ABI or behavior changes. Select a new version deliberately; record field, function, JSON, and semantic changes in the review. Do not assume adding a required field or import is backward compatible.
3. Regenerate bindings and test an actual guest component against the host, including error paths and cursor rollback.
4. Update this README and coordinate consumer pins. Updating this repository alone does not update consumers or rebuilt plugin binaries.

Documentation-only changes that preserve the contract do not require a package version bump.
