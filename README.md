# Cardbe Plugin API

This repository owns the single WIT contract consumed by Cardbe and independent
plugins: [wit/plugin.wit](wit/plugin.wit), package `cardbe:plugin@1.0.0`, world
`plugin`. The API is not released yet; rebuild guests and Host bindings from
one reviewed revision. Old draft components, including the JSON-query/changeset
contract, are not compatible with this typed ABI; Manifest changes cannot fix it.
There is no Plugin kind or Manifest world selection.

Cardbe pins this repository as a submodule. Commit/review API changes here first,
then update Cardbe's gitlink. API/package/storage schema versions are independent;
keep released API versions immutable once releases begin.

## Plugin package and manifest

The public package format is JSON metadata plus a WebAssembly component built for the `plugin` world. These JSON rules complement WIT; they do not change the WIT ABI. Cardbe validates manifests and instance configuration at the host boundary.

A local package directory contains `manifest.json` and `component.wasm` together. A ZIP package must contain exactly one `manifest.json` and a sibling `component.wasm`; both may be inside one directory. The current installer accepts ZIP archives up to 20 MiB with at most 256 entries, manifests up to 64 KiB, and components up to 16 MiB. Duplicate entry names, unsafe paths, backslash paths, and symbolic links are rejected. Other archive files are not installed; do not rely on bundled translation or resource files being available.

### Manifest fields

| Field | Type / default | Contract and current host validation |
| --- | --- | --- |
| `id` | Required string | Stable plugin identity, 1–100 ASCII characters from `a-z`, `0-9`, `-`, `_`. Windows device names `con`, `prn`, `aux`, `nul`, `com1`–`com9`, and `lpt1`–`lpt9` are reserved. |
| `name` | Required localized text | Display name; use meaningful text in both supported languages. |
| `version` | Required string | Plugin release version. Use stable `x.y.z`; in-place updates require three unsigned 64-bit integers without leading zeros and a strictly greater numeric version. Prerelease and build suffixes are not supported for updates. |
| `api_version` | Required string | Currently exactly `1.0.0`, matching the WIT API consumed by the component. |
| `storage_schema_version` | Required unsigned 32-bit integer | Currently exactly `1`. Independent of plugin release and WIT versions; it is not permission to access or migrate the host database. |
| `description` | Localized text / `null` | Optional display description. |
| `domains` | String array / `[]` | At most 16 unique, nonempty lowercase ASCII hostnames. Each is at most 253 bytes; dot-separated labels are 1–63 bytes, use `a-z`, `0-9`, `-`, and cannot start or end with `-`. Supply hostnames only, without scheme, port, path, or wildcards. Runtime URL and public-address checks still apply. |
| `allow_custom_domains` | Boolean / `false` | Allows the user to approve domains beyond the declared list; it does not grant automatic network access. |
| `settings` | Setting field array / `[]` | At most 32 fields with unique keys; see below. |
| `error_codes` | String array / `[]` | At most 64 declared guest error codes. Each is 1–64 ASCII characters from `A-Z`, `0-9`, `_`. |
| `log_codes` | String array / `[]` | At most 64 declared diagnostic codes, using the same code format. |

Localized text is a string, `null`, or an object mapping locale IDs to strings. Cardbe currently displays `en` and `zh-TW`, selecting the active locale and falling back to `en`, then an empty string. Use both locales for names, descriptions, field labels, and option labels. These display fields do not define guest error translations or override core application messages. The current manifest has no error translation resource field.

### Setting declarations and instance values

Each field requires `key`, `label` (localized text), and `type`. Optional `description` uses the same localized-text format (string, locale-to-string object, or `null`) and defaults to `null`. Supply `en` and `zh-TW` explanations of the field, expected format, and relevant limits. Cardbe shows a question-mark button beside the field label when the active-locale description (with the usual English fallback) is nonblank; clicking or keyboard-activating it opens the explanation. Descriptions are plain text, support line breaks, and never enter instance configuration or guest `config-json`. Older manifests without descriptions remain valid. This is host package metadata, not a WIT API change. Keys use the same format and reserved-name rules as plugin IDs. `required` defaults to `false`; `options` defaults to `[]`; `min`, `max`, and `default` may be omitted or `null`.

| Type | Saved non-secret configuration value |
| --- | --- |
| `text` | String up to 4,096 UTF-8 bytes; required text cannot be whitespace-only. |
| `number` | Finite JSON number within inclusive `min` and `max`, when supplied. |
| `boolean` | JSON boolean; `false` is a valid required value. |
| `select` | String equal to one of the declared option values. |
| `board` | Integer equal to the configured instance's target board ID. |
| `column` | Integer identifying an existing column on the target board. |
| `secret` | Supplied separately to the host; never included in configuration or `config-json`. |

An option requires a nonempty string `value` (at most 256 UTF-8 bytes) and a localized `label`. Each field may declare at most 100 options, with unique values. Numeric bounds must be finite and `min <= max` when both exist.

Defaults may be JSON scalars, not arrays or objects. Choose defaults that satisfy the field's type, options, and bounds: manifest parsing does not fully validate defaults against their field type, but saving an instance validates its values. Cardbe's settings form applies non-secret, non-null defaults; the backend does not automatically fill missing required configuration. Omit optional values instead of sending `null`. Unknown configuration keys are rejected.

At most one `secret` field is supported, and it requires exactly one declared domain. Never provide a non-null secret default or embed credentials in metadata. Secret values are limited to 8,192 UTF-8 bytes and stored separately by the host. The current HTTP capability uses the secret as a Bearer token only for the declared domain when the instance has exactly one approved domain. Custom or additional approved domains do not receive that token. Plugins cannot read the secret or select arbitrary authentication headers.

Declared domains are requests for permission. The instance's approved domains determine actual access; an unapproved declared domain, or an eligible custom domain, can pause a run for user approval. HTTPS, public-address validation, DNS pinning, and the 16-domain limit still apply after approval.

### Example manifest

This example declares one credentialed origin and a destination column selected by the user. Build a matching component separately; this metadata alone is not an executable plugin.

```json
{
  "id": "example-importer",
  "name": { "en": "Example importer", "zh-TW": "範例匯入插件" },
  "version": "1.0.0",
  "api_version": "1.0.0",
  "storage_schema_version": 1,
  "description": { "en": "Import items into a board.", "zh-TW": "將項目匯入看板。" },
  "domains": ["api.example.com"],
  "allow_custom_domains": false,
  "error_codes": ["SOURCE_UNAVAILABLE"],
  "log_codes": ["ITEMS_READ"],
  "settings": [
    {
      "key": "token",
      "label": { "en": "Access token", "zh-TW": "存取權杖" },
      "type": "secret",
      "required": true
    },
    {
      "key": "destination",
      "description": { "en": "Column receiving imported items on the target board.", "zh-TW": "目標看板中用來接收匯入項目的欄位。" },
      "label": { "en": "Destination column", "zh-TW": "目標欄位" },
      "type": "column",
      "required": true
    },
    {
      "key": "include_closed",
      "label": { "en": "Include closed items", "zh-TW": "包含已關閉的項目" },
      "type": "boolean",
      "default": false
    }
  ]
}
```

The guest receives configuration such as `{"destination": 42, "include_closed": false}`, where `42` must be an actual column ID on the target board. It never receives `token`; the host adds authentication to eligible HTTP requests.

### Versions and updates

Keep the three version fields independent: `version` identifies a plugin release, `api_version` selects the public execution API, and `storage_schema_version` declares the package's supported storage schema. The current host accepts API `1.0.0` and schema `1`; an in-place update must retain plugin ID, API version, and storage schema version.

Changes to domains, custom-domain permission, or setting declarations require instance review before execution resumes. Changing the credential domain or removing/renaming the secret field also requires credential review. Publishing a new plugin release does not update the WIT package or the host's database schema. Runtime, permissions, persistence, and validation remain host responsibilities.

## Contract

`run(context)` returns `result<run-output, plugin-error>`. Context carries
instance/run/board IDs, board snapshot version, trigger (manual/schedule),
configuration JSON and opaque cursor. Each run has a fresh Wasmtime Store.
There is no whole-board JSON input: use scoped, typed board functions and paginated `list-tasks`.

Output can combine external-task upserts, a typed changeset and cursor.
`require-confirmation` requests review; it cannot authorize automatic writes.
Host device-local instance consent chooses automatic apply versus preview for
all operations, with current board permissions and validation in both cases.
An empty output supports HTTP/reporting-only work. Read-only output may still
update its local checkpoint. Return the input cursor to preserve it; none clears
it only on successful commit. Function signatures and data types are defined
in [wit/plugin.wit](wit/plugin.wit); the execution semantics are described below.

External tasks use stable `external-key`, title, description, URL, state, labels,
updated-at and destination column ID. Omitted tasks are not deleted. Host merges
through existing source ownership/user overrides, and keeps source mappings and
cursor local to this instance/device. Plugins never receive SQLite connections,
app paths, host exceptions or host credentials.

## HTTP and diagnostics

Both GET and POST use the same host capability. HTTPS port 443 only; grants are
at most 16 exact hostnames, never paths, wildcards or arbitrary subdomains.
Only Manifest domains (or declared custom-domain capability) are eligible.
An unapproved eligible hostname stops the run before DNS/HTTP and before board,
source or cursor writes; approval starts a fresh run with the original trigger
and waits for execution capacity. Public DNS validation and
pinning, no redirects/proxies and bounded cancellation/timeouts remain enforced.

`http-post(url, content-type, body)` accepts a nonempty valid header value of
at most 256 bytes and a body of at most **10 MiB (10 * 1024 * 1024 bytes)**.
POST bodies total at most 32 MiB per run, within the 100-request cap. Guest
cannot supply arbitrary headers. Host attaches bearer credentials only for
the original sole manifest domain when it is also the sole approved domain;
custom/multiple destinations do not receive that credential.

HTTP statuses, including non-2xx, are responses; guest must inspect them.
Policy/transport failures return safe codes and fail the entire run even when
guest catches them. Response headers are allowlisted and bodies bounded.
The HTTP capability has no retry loop; later scheduled/manual runs can issue
POST again. A timeout can leave its remote outcome unknown. External effects
occur during execution, before board confirmation, and cannot be rolled back by
local rejection/cancellation. Choose read-only endpoints when planning, or use
remote idempotency for mutations.

`host.log(code,count)` only accepts manifest-allowlisted bounded diagnostic
codes, never arbitrary content or secrets. Plugin errors likewise use declared
codes and a bounded retry hint. Host failures stay in their safe error namespace.
No WASI filesystem/network/process/environment/clock capabilities are linked.

## Host limits

| Resource | Limit |
| --- | --- |
| Component / linear memory | 16 MiB / 128 MiB |
| Run duration / fuel | 60 seconds / 500,000,000 fuel |
| Configuration / cursor | 64 KiB / 4 KiB |
| HTTP requests per run | 100 |
| POST body per request / total per run | 10 MiB / 32 MiB |
| HTTP response body / total per run | 2 MiB / 32 MiB |
| General operations / source upserts | 200 / 2,000 |
| Combined changeset / resulting active columns | 8 MiB / 8 MiB |
| Pending previews | 1 per instance, 8 per app |
| Recorded logs | 100 |
| External key / title / description | 512 / 4,096 / 262,144 bytes |
| URL / state / updated-at | 4,096 / 64 / 128 bytes |
| Labels / each label | 100 / 256 bytes |

Query limits and explorer search behavior are described below.
These are Host policies, not promises encoded in WIT. Strings use UTF-8 bytes.

## Board operations and consistency

The typed board interface provides list-boards, get-board, list-columns,
get-column, get-task and list-tasks. Each success includes the same opaque
snapshot version for the invocation. Access is scoped to the instance's one
consented board; list-boards returns only that board. Query errors use the WIT
host-error enum. No SQLite handle, path, caller-provided board scope or clock
is exposed. Archived tasks have no column or position because archives do not
retain original columns. Dates are u64 epoch milliseconds; unsupported larger
stored values are rejected without truncation.

list-tasks uses the existing explorer filtering/search and OFFSET pagination.
Choose status all to include active and archived tasks; limit is 1..100.
Stop on a short/empty page. Search retains its candidate/result caps; use an
empty query to enumerate all tasks. Search option integers are 1..10,000;
query text is at most 4 KiB. Each returned native payload is budgeted internally
at 2 MiB, totaling 32 MiB and at most 1,000 calls per run. Invalid arguments do
not poison subsequent calls. Version hashing is O(N) once per invocation;
deep pages and searches retain the existing explorer costs.

A changeset contains a snapshot version and typed operation variants. Column
references are numeric IDs or temporary keys from an earlier create-column.
Forward references are rejected. Host allocates IDs, preserving them between
preview and commit. New tasks append; their keys are available in the native
preview, and they can be positioned in a later changeset. Task updates replace
all fields while preserving identity: copy the queried task before editing.
Explicit task moves require a destination with custom sorting. External upserts
preserve user overrides, managed labels and tombstones; repeated upserts run in
order. Their optional column reference can target a temporary column. The tasks
convenience list is appended after the general operations and uses numeric IDs.

Host validates and simulates all output before writing. Missing targets reject
the whole changeset; a target removed after the snapshot makes the output stale.
Host rechecks grants, board edit permission, snapshot version and source mapping
preconditions before committing. Projection, source links, cursor, one local
Loro delta and run records commit in one SQLite transaction without partial
success. This is local atomicity: P2P delivery is asynchronous, HTTP effects
cannot be rolled back, and there is no durable retry/idempotency key or Undo.
Archives, settings and templates are preserved. Checklist limits are 1,000 entries
with unique nonempty IDs (128 bytes) and text 4 KiB; labels must be unique and
recurrence intervals are 1..10,000. Existing application date/text bounds apply.

Review is optional. User consent allow_automatic_changes defaults to false;
when enabled and the guest returns require-confirmation false, valid output is
committed without displaying or storing a pending review. A guest may request
confirmation on any run. Read-only/no-op output does not require review.
Pending previews are volatile: one per instance, eight per app, invalidated by
configuration/grant saves, disabling, package updates, removal or restart.
Scheduled runs pause while awaiting confirmation; review does not re-run the
guest. There is no action registry, durable job queue or automatic stale retry.
Board create/rename, cross-board mutation, deletion and archive/restore are not
currently exposed.

## Local execution history and lifecycle

Accepted guest logs are buffered in memory as code/count and stored with a
normally returned guest's run record as level info. Records are device-local
JSON payloads in SQLite cardbe_plugin_data (kind run, id run-id). Each instance
retains its latest 50 runs, with at most 100 declared diagnostics per run.
Guest errors, traps, host request failures and cancellation do not currently
preserve the buffered guest logs; they still record safe execution status/errors.
This is execution history, not a streaming text log or synchronized board data.
Removing an instance also removes its run history and source/cursor bookkeeping,
while leaving created board tasks intact.

An actively running instance blocks its removal and removal/update of its package.
Update confirmation returns PLUGIN_UPDATE_BUSY; removal currently returns
INVALID_ARGUMENT. Cancel first and wait for the active run to finish before
retrying. A prepared update may be inspected before activation, but activation
cannot replace a running component. Awaiting-confirmation output is no longer
executing; removal/update invalidates its pending changeset. Disabling an instance
cancels its active run. No expired Wasm execution is resumed.

## Validation

Generate bindings with `path: "wit", world: "plugin"`. Wasm components are
static contracts; Host installation/update checks actual imports/exports.
CI parses/encodes WIT with wasm-tools 1.259.0 and validates the binary:

```sh
wasm-tools component wit wit --wasm --output cardbe-plugin-api.wasm
wasm-tools validate cardbe-plugin-api.wasm
```

For each contract change, review Host/guest bindings, adapters and fixtures
together, rebuild and test an actual component (including failures/cursor
rollback), and update documentation and consumer pins. Production plugins build
and release in their own repositories; Cardbe builds only its test fixture.
