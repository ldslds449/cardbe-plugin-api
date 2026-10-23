# Cardbe Plugin API

This repository owns the single WIT contract consumed by Cardbe and independent
plugins: [wit/plugin.wit](wit/plugin.wit), package `cardbe:plugin@1.2.0`, world
`plugin`. The API is not released yet; rebuild guests and Host bindings from
one reviewed revision. Old draft components, including the confirmation flag and JSON-query/changeset
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
| `version` | Required string | Plugin release version in stable `x.y.z` form: three unsigned 64-bit integers without leading zeros. Installation rejects other formats, including prerelease and build suffixes; in-place updates require a strictly greater numeric version. |
| `api_version` | Required string | `1.1.0` or `1.2.0`, matching the WIT API consumed by the component. `1.2.0` only adds task priority, completion, and archive/restore functions, so `1.1.0` components keep working. Components built for the unreleased `1.0.0` draft are not compatible: `1.1.0` changed the `task` record (see Board operations). |
| `manifest_version` | Unsigned 32-bit integer / `1` | Manifest format and semantics. This host supports `1` and `2`; other values and non-integers are rejected. Omission retains legacy v1 behavior. Authentication objects in `http_credentials[].auth` require `2`. Independent of release, WIT, and storage versions. |
| `storage_schema_version` | Required unsigned 32-bit integer | Currently exactly `1`. Independent of plugin release and WIT versions; it is not permission to access or migrate the host database. |
| `description` | Localized text / `null` | Optional display description. |
| `domains` | String array / `[]` | At most 16 unique, nonempty lowercase ASCII hostnames. Each is at most 253 bytes; dot-separated labels are 1–63 bytes, use `a-z`, `0-9`, `-`, and cannot start or end with `-`. Supply hostnames only, without scheme, port, path, or wildcards. Runtime URL and public-address checks still apply. |
| `allow_custom_domains` | Boolean / `false` | Allows the user to approve domains beyond the declared list; it does not grant automatic network access. |
| `http_credentials` | Credential rule array / omitted | Optional host-managed Bearer or API-key header rules for fixed domains or Base URL settings, scoped by API path. At most 64 rules. An explicit `[]` disables automatic authentication; omission or `null` retains legacy behavior. Requires a host implementing these rules; see below. |
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

Defaults may be JSON scalars, not arrays or objects, and must match the field type: `text` strings up to 4,096 bytes, `number` values within the declared bounds, booleans, `select` values among the declared options, and integer `board`/`column` IDs. Manifest parsing rejects mismatched defaults; because board and column IDs are device-specific, saving an instance still validates them against the target board. Cardbe's settings form applies non-secret, non-null defaults; the backend does not automatically fill missing required configuration. Omit optional values instead of sending `null`. Unknown configuration keys are rejected.

Never provide a non-null secret default or embed credentials in metadata. Secret values are limited to 8,192 UTF-8 bytes without control characters other than tab, and stored separately by the host. A previously saved secret that cannot be sent as an HTTP header value fails the run with `INVALID_CREDENTIAL` before execution; re-enter it. Plugins cannot read secrets or supply arbitrary authentication headers. With explicit `http_credentials`, a package can declare multiple secret fields and assign them to different hosts and API paths. Without these rules (or with `null`), legacy validation allows at most one secret field and requires exactly one declared domain; its token is sent only when that original domain is the sole approved host. Legacy credentials never go to a custom host.

Declared domains are requests for permission. The instance's approved domains determine actual access; an unapproved declared domain, or an eligible custom domain, can pause a run for user approval. HTTPS, public-address validation, DNS pinning, and the 16-domain limit still apply after approval.

### HTTP credential rules

Connection grants and credential rules are independent. Approving a host permits
network access to that entire host; it does not give every request a token.
`http_credentials` is host package metadata and does not change the WIT API
or storage schema `1`. Packages using it require a host that implements these
rules: older hosts can ignore unknown metadata, so do not distribute such packages
to them.

Each rule contains:

| Field | Contract |
| --- | --- |
| `destination` | Either `{ "type": "domain", "domain": "api.example.com" }`, with the exact hostname listed in `domains`, or `{ "type": "base_url", "setting": "base_url" }`, referencing a `text` setting. |
| `path_prefix` | Required path prefix, relative to the Base URL path or to the fixed host root. `/` covers the complete base path. |
| `bearer_secret` | Optional key of a declared `secret` setting. Omission or `null` explicitly sends no token. |
| `auth` | Optional host-managed authentication: `{ "type": "bearer", "secret": "token" }` or `{ "type": "header", "name": "X-API-Key", "secret": "api_key" }`. Requires a host supporting API-key header rules. Do not combine with `bearer_secret`, even if either field is `null`. |

Set the package's top-level `"manifest_version": 2` when using an `auth` object.
Manifest v1 still supports existing `bearer_secret` rules; manifest v2 supports
both legacy rules and the new format. Unknown versions are rejected on install,
update, save, connection approval, and execution preparation. `auth: null` is
equivalent to omission and can also be used in v1. Changing manifest version
requires instance review; credential re-entry is required when credential rules
or destinations change, not solely because the manifest version changes.
This field cannot retroactively make published older hosts reject unknown
metadata: distribute v2 packages only to hosts known to support v2.

`bearer_secret` remains supported for existing packages. `auth: null`, or omission
of both authentication fields, sends no credential for that rule. A header rule
sends the secret value unchanged; a Bearer rule adds `Authorization: Bearer`.
Secret values remain in the host's credential store and are never included in
guest configuration. Enter the raw token or API key, without a `Bearer` prefix.
Authentication rules apply to both GET and POST, using the same exact host/port
grants, path boundaries, longest-prefix selection, and no-fallback behavior.

Header names must be valid HTTP field names with at most 128 ASCII bytes.
Host-managed and transport headers are reserved: `Authorization`, `Host`,
`Cookie`, `Cookie2`, `Content-Length`, `Connection`, `Keep-Alive`,
`Proxy-Authorization`, `Proxy-Authenticate`, `Proxy-Connection`, `TE`, `Trailer`,
`Transfer-Encoding`, `Upgrade`, `Accept`, `Content-Type`, `User-Agent`, and
names beginning with `Sec-` (case-insensitive). Secret values must be valid HTTP
header values, with at most 8192 UTF-8 bytes; CR/LF and other forbidden controls
are rejected on save and before DNS/HTTP. Header names are displayed separately
from secret field labels in consent settings. Changing the authentication type,
header name, referenced secret, or destination requires review and secret re-entry.
General request-header customization, Basic Auth, cookies, and OAuth refresh are
not part of this metadata. No WIT or storage-schema change is required.

For an API using `X-API-Key`, declare an `api_key` secret setting and use:

```json
{
  "destination": { "type": "base_url", "setting": "base_url" },
  "path_prefix": "/tasks",
  "auth": { "type": "header", "name": "X-API-Key", "secret": "api_key" }
}
```

For a URL-setting destination, enter the Base URL once in instance configuration.
Its host must be listed in `domains` unless `allow_custom_domains` is `true`.
The URL must use HTTPS with a nonzero port (443 by default), without user information, query, or fragment,
and satisfy the existing public-host validation. Valid nonempty defaults for
these settings are also checked during manifest parsing. An absent or empty
optional URL setting leaves its rules inactive; required fields retain the normal
configuration validation. The host still needs explicit connection approval.

Prefixes use ASCII letters, digits, `/`, `-`, `.`, `_`, and `~`, with at most
2048 bytes. They cannot contain wildcards, dot segments, duplicate slashes, or
a trailing slash except `/`. Base URL paths use the same character restriction
with an optional single trailing slash. Raw dot segments, encoded paths, and
duplicate slashes are rejected before URL normalization. With `https://custom.example.com/api`, a prefix
of `/tasks` covers `/api/tasks` and its descendants, including `/api/tasks/123`.
It does not cover `/api/tasks-other`. Matching is case-sensitive, query strings
do not select credentials, and the longest matching prefix wins regardless of
manifest order.

For example, add this fragment to a complete manifest with the usual identity
and version fields:

```json
{
  "domains": ["fixed.example.com"],
  "allow_custom_domains": true,
  "settings": [
    { "key": "base_url", "type": "text", "label": "API base URL", "required": true },
    { "key": "api_token", "type": "secret", "label": "API token", "required": true },
    { "key": "admin_token", "type": "secret", "label": "Admin token" }
  ],
  "http_credentials": [
    {
      "destination": { "type": "base_url", "setting": "base_url" },
      "path_prefix": "/tasks",
      "bearer_secret": "api_token"
    },
    {
      "destination": { "type": "base_url", "setting": "base_url" },
      "path_prefix": "/tasks/public"
    },
    {
      "destination": { "type": "base_url", "setting": "base_url" },
      "path_prefix": "/admin",
      "bearer_secret": "admin_token"
    },
    {
      "destination": { "type": "domain", "domain": "fixed.example.com" },
      "path_prefix": "/tasks",
      "bearer_secret": "api_token"
    }
  ]
}
```

Use the existing locale objects for setting labels and descriptions in production
packages. The guest receives `base_url` but never either token. The host attaches
the assigned Bearer token to matching GET and POST requests. `/tasks/public`
and its descendants explicitly have no token despite the broader `/tasks` rule.
Unmatched requests also have no token. A missing optional secret never falls back
to a broader rule, another secret, or legacy authentication. `http_credentials: []`
disables automatic authentication even if secret settings exist.

Duplicate destinations with the same prefix are rejected in the manifest.
Different destinations resolving to the same host and prefix are rejected when
saving configuration or preparing a run. On hosts with token rules, ambiguous
request paths (encoded routing separators, double encoding, control characters,
semicolons, or duplicate slashes) are rejected before DNS/HTTP. Public DNS checks,
pinning, no redirects/proxies, and connection-grant limits still apply.

The settings editor displays resolved credential hosts and paths separately from
connection checkboxes. Network configuration changes preserve board consent.
Changing an existing instance's Base URL host removes the new host's grant on save,
even if it had previously been approved; a fresh run asks for approval again.
Changing only the path on the same host retains the host grant. Configuration
saves invalidate pending requests; pending records contain only host, run ID, and
trigger, never the URL, query string, or credentials.

### Example manifest

This example declares one credentialed origin and a destination column selected by the user. Build a matching component separately; this metadata alone is not an executable plugin.

```json
{
  "id": "example-importer",
  "name": { "en": "Example importer", "zh-TW": "範例匯入插件" },
  "version": "1.0.0",
  "api_version": "1.2.0",
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

Keep the four version fields independent: `version` identifies a plugin release, `manifest_version` selects package format and semantics, `api_version` selects the public execution API, and `storage_schema_version` declares the package's supported storage schema. The current host accepts manifest versions `1` and `2`, API versions `1.1.0` and `1.2.0`, and storage schema `1`; an in-place update must retain plugin ID and storage schema version. Supported manifest versions may change during an in-place update; the plugin release version must still increase.

Changes to domains, custom-domain permission, setting declarations, or `http_credentials` require instance review before execution resumes. Changing declared credential domains or credential rules, or removing/renaming an existing secret field, also requires credentials to be re-entered. Re-entering one secret never reactivates other old secrets. Publishing a new plugin release does not update the WIT package or the host's database schema. Runtime, permissions, persistence, and validation remain host responsibilities.

## Contract

`run(context)` returns `result<run-output, plugin-error>`. Context carries
instance/run/board IDs, board snapshot version, trigger (manual/schedule),
configuration JSON and opaque cursor. Each run has a fresh Wasmtime Store.
There is no whole-board JSON input: use scoped, typed board functions and paginated `list-tasks`.

Output can combine external-task upserts, a typed changeset and cursor.
After the user authorizes an instance to read and write its selected board,
Host validates and commits output directly for manual and scheduled runs. There
is no change preview or per-run confirmation. Current permissions are rechecked.
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

`http-get`, `http-post`, and `http-send` (API 1.1.0) use the same host capability. HTTPS with a nonzero port (443 by default); grants are
at most 16 exact hostnames, never paths, wildcards or arbitrary subdomains.
Only Manifest domains (or declared custom-domain capability) are eligible.
An unapproved eligible hostname stops the run before DNS/HTTP and before board,
source or cursor writes; approval starts a fresh run with the original trigger
and waits for execution capacity. Public DNS validation and
pinning, no redirects/proxies and bounded cancellation/timeouts remain enforced.

`http-post(url, content-type, body)` accepts a nonempty valid header value of
at most 256 bytes and a body of at most **10 MiB (10 * 1024 * 1024 bytes)**.
Request bodies total at most 32 MiB per run, within the 100-request cap shared
by all HTTP functions.

`http-send(request)` supports GET, POST, PUT, PATCH, and DELETE. Its `headers`
are guest-supplied: at most 32 unique (case-insensitive) names, 32 KiB of names
and values combined, valid HTTP names and control-free values of at most 8192
bytes. `Accept` and `Content-Type` may be set; otherwise `Accept` defaults to
`application/json`. The reserved credential-header names listed above, plus
`Expect`, are rejected, as are header names used by this package's
`http_credentials` header rules, so guests cannot replace host credentials. A
`body` requires a nonempty `Content-Type` of at most 256 bytes and is rejected
for GET. Invalid requests fail the run like other policy errors. With explicit `http_credentials`, Host attaches
Bearer or API-key header credentials only according to the matching host/port/path rule and the instance's
connection grants. Legacy authentication remains restricted to the original sole
manifest domain when it is also the sole approved domain; custom or multiple
destinations never receive that legacy credential. See HTTP credential rules above.

HTTP statuses, including non-2xx and `304 Not Modified`, are responses; guest
must inspect them. Other 3xx statuses are redirects and fail the run.
Policy/transport failures return safe codes and fail the entire run even when
guest catches them, with one exception: a `NETWORK_FAILED` error from
`http-send` (DNS, connection, TLS, or the 15-second per-request timeout) leaves
the run usable, so the guest may skip the item or fall back. Run timeouts,
cancellation, limits, and permission failures remain fatal. Response headers
are allowlisted (`content-type`, `etag`, `last-modified`, `link`, `location`,
`retry-after`, `x-ratelimit-remaining`, `x-ratelimit-reset`) and bodies are
bounded; send `If-None-Match` or `If-Modified-Since` through `http-send` for
conditional requests.
The HTTP capability has no retry loop; later scheduled/manual runs can issue
POST again. A timeout can leave its remote outcome unknown. External effects
occur during execution, before local commit, and cannot be rolled back by
local rejection/cancellation. Use remote idempotency for mutations.

`host.log(code,count)` only accepts manifest-allowlisted bounded diagnostic
codes, never arbitrary content or secrets. API 1.1.0 adds
`host.log-message(level,message)` for free-text diagnostics at `debug`, `info`,
`warn` or `error`. Messages are kept in order with coded logs in the local run
history. The host redacts the instance's loaded credential values as `***`,
replaces control characters other than newline and tab, and truncates each
message at a UTF-8 boundary. They never leave the device, but they are not
encrypted; do not log user content, personal data or response bodies the user
would not expect to keep.
Plugin errors use declared codes and a bounded retry hint, like coded logs.
Host failures stay in their safe error namespace.
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
| Recorded coded logs / text messages | 100 / 200 per run |
| Text message / total text per run | 1,024 / 65,536 bytes |
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

A task has three independent times: optional `start-time` (user-chosen start),
optional `due-time`, and read-only `created-time`. The host stamps the creation
time for new tasks, including plugin-created tasks, and keeps it on updates;
values supplied by the guest are ignored. Do not put the current time in
`start-time` to mean "created now": it is shown to users as the task's start.
Tasks saved before creation times were recorded report their creation time in
`created-time` and no start time. A start or due time at local midnight on this
device is a date without a time of day (a date-only start begins at that
midnight, a date-only due date lasts until the end of that day) unless
`start-at-midnight` or `due-at-midnight` marks it as a chosen 00:00. The flags
are ignored for times that are not local midnight; set them only when the user
means 00:00. A created task's `start-time` may not be later than its `due-time`,
nor may an updated task's when either time changes; a due time at local midnight
on this device is treated as a date-only due date covering that whole day,
unless `due-at-midnight` is set. Unchanged times are accepted as read, because a
date-only due date set in another time zone is not midnight here. Violations
reject the whole changeset.

Task priority (API 1.2.0) is not part of the `task` record, so components built
for 1.1.0 keep working. Read it with `get-task-priority`, which counts toward
the same query budget. Stage changes with `set-task-priority`, referencing an
existing task ID or the key of a create-task in the returned changeset; none
clears the priority. Staged priorities apply in call order after the returned
operations (or after an empty changeset at the run snapshot) and commit in the
same transaction, so a failed or stale run applies none of them. Task updates
always keep the stored priority. A run may stage at most 200 priorities, and
they count toward the changeset's 200 general operations; keys are 1..128 bytes.
`set-task-priority` writes directly and does not track user changes.

To synchronize a source's priority, call `set-external-task-priority` with the
external key of a task returned in `tasks` or upserted in the changeset.
Reported priorities apply after all upserts and follow the same ownership as
titles: the source replaces the priority until the user changes it, and never
afterwards, even if the values match again. Tasks without a report keep their
priority, so 1.1.0 components never change it; none clears it. The last call
for a key wins, a run may report at most 2000 keys (1..512 bytes), and a key
that this run does not upsert rejects the whole changeset.

Archive and restore (API 1.2.0) are staged the same way. `archive-task` takes an
active task ID or the key of a create-task in the returned changeset;
`restore-task` takes an archived task ID (from `get-task` or `list-tasks` with
archived tasks) and a destination column ID or create-column key, and appends
the task to that column. Archive, restore and `set-task-priority` calls share
one queue: they apply in call order after the returned operations, before
reported source priorities, and commit in the same transaction, so a failed or
stale run applies none of them. Together they count toward the 200 general
operations. A target that is missing, or not active (archive) or not archived
(restore) when its call applies, rejects the whole changeset, so a task can be
archived and restored in one run only in that order of calls. Archiving keeps
the task's fields and priority; restoring returns them unchanged. As
completion follows position, archiving completes a task and restoring it to a
column other than the done column reopens it. A recurring
task continues with its next occurrence in its column, as when archived in the
app; query a later run to obtain the new task's ID.

An archived source task is a tombstone: later upserts of its external key do
not update or recreate it, and source priorities reported for it are not
applied. After it is restored, upserts update it again with the usual
ownership. There is no delete; archive instead.

Completion (API 1.2.0) follows a task's position: a task is completed while it
is in the board's done column, or once it is archived. `get-done-column`
returns that column, if any; to complete a task, move it there with
`move-task`, and the host records when. `get-task-completed-time` returns
milliseconds since the Unix epoch when the task entered the done column, or the
archive time of a task completed by archiving; open tasks and tasks with an
unknown time return none. Both count toward the query budget. Task updates keep
the stored completion time.

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
Forward references are rejected. Host allocates IDs during validation. New tasks
append; query them in a later run to obtain IDs for positioning. Task updates replace
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

Valid output commits directly after instance board authorization; manual and
scheduled runs use the same validation and transaction path. Old pending-review
records are cancelled on startup, never applied. There is no action registry,
durable job queue or automatic stale retry.
Board create/rename, cross-board mutation and deletion are not currently
exposed.

## Local execution history and lifecycle

Accepted guest logs are buffered in memory (codes at level info, text messages
at their own level) and stored with the run record, whether the run succeeds,
fails or is cancelled. Records are device-local JSON payloads in SQLite cardbe_plugin_data (kind run,
id run-id). Each instance retains its latest 50 runs, with at most 100 declared
diagnostic codes and 200 text messages per run.
Guest errors, traps, host request failures, timeouts and cancellation keep the
logs recorded before the failure next to the safe execution status/error, so a
plugin can log progress codes to diagnose where a run stopped. Hosts without API
1.1.0 support discard these logs on failure.
This is execution history, not a streaming text log or synchronized board data.
Removing an instance also removes its run history and source/cursor bookkeeping,
while leaving created board tasks intact.

An actively running instance blocks its removal and removal/update of its package.
Update confirmation returns PLUGIN_UPDATE_BUSY; removal currently returns
INVALID_ARGUMENT. Cancel first and wait for the active run to finish before
retrying. A prepared update may be inspected before activation, but activation
cannot replace a running component. Disabling an instance cancels its active run.
No expired Wasm execution is resumed.

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
