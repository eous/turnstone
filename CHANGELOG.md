# Changelog

All notable changes to turnstone are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [PEP 440](https://peps.python.org/pep-0440/) for
version numbers (`X.Y.Z`, with `X.Y.ZaN` / `bN` / `rcN` for pre-releases).

Three release tracks are maintained:

- **`stable/1.0`** — patch-only (`v1.0.x`)
- **`stable/1.3`** — patch-only (`v1.3.x`)
- **`stable/1.4`** — patch-only (`v1.4.x`)
- **`main`** — experimental (`v1.5.0aN`)

## [Unreleased]

## [1.4.0]

User-visible additions: a full attachment system (images + text documents,
including pre-creation uploads), a unified dashboard composer, a Slack
channel adapter, per-call plan/task model selection with an admin UI, and
provider capability passthrough.

This release introduces two forward-only schema migrations
(`037_workstream_attachments`, `038_workstream_attachments_reserved_at`)
that the server applies automatically on first startup against an
existing 1.3.x database.  Both are additive; no data loss.  See
**Database migrations** below for details.

### Added

- **Workstream attachments** — images (png/jpeg/gif/webp, 4 MiB cap) and
  text documents (any `text/*` MIME, allowlisted application MIMEs, or
  known text extensions; 512 KiB cap; UTF-8 enforced).  Magic-byte image
  sniffing on upload; per-(ws, user) pending cap of 10.  Three-state
  lifecycle (`pending → reserved → consumed`) with reservation tokens
  threaded through `/v1/api/send` so queued multimodal turns can't lose
  files to overlapping sends.  Provider-side translation: Anthropic
  emits native document blocks; OpenAI Chat Completions inlines them as
  escaped `<document>` text blocks; Responses API emits `input_text`
  with the same wrapper. (#356)
- **Attachments at workstream-creation time** —
  `POST /v1/api/workstreams/new` accepts `multipart/form-data` (one
  `meta` JSON field plus 0..N `file` parts).  Files are validated and
  reserved onto the first turn before the dispatch worker fires; failure
  rolls back the fresh workstream so no orphan rows leak.  Web UI
  (new-workstream modal + dashboard composer), Python SDK, and
  TypeScript SDK all gained attachment support.  Cluster routing
  (`/v1/api/route/workstreams/{ws_id}/attachments`) extended to forward
  multipart bodies + preserve upstream headers (CSP, Content-Disposition).
  SDKs auto-generate `ws_id` client-side so cluster-routed callers can
  bind the body to the owning node before it lands. (#362)
- **Slack channel adapter** (Socket Mode) — mirrors the Discord adapter:
  per-user channel sessions via configurable slash command, DM routing,
  SSE event consumption, tool approval buttons (with per-user owner
  enforcement), plan-review approve / request-changes modal,
  notification reply routing back into the workstream, session recovery
  after restart.  Install with `pip install 'turnstone[slack]'`. (#355)
- **Console admin UX support for Slack** — channel-link modal offers
  Slack alongside Discord; skill notify-on-complete forms expose a
  per-row channel-type dropdown (and no longer hardcode `discord`);
  per-platform `.scope-discord` / `.scope-slack` badge classes with
  theme-aware tokens (`--discord` / `--slack`) so light theme passes
  WCAG AA. (#365)
- **Per-call plan/task model selection** — `plan_model` and `task_model`
  are now distinct from the conversation model and from each other, with
  configurable reasoning effort per agent.  `ConfigStore` admin tab in
  the console UI lets operators set defaults; per-call overrides
  available via the `plan_agent` / `task_agent` tools. (#360, #361)
- **Provider capability passthrough** — resolved per-model capabilities
  (vision support, reasoning support, native web search, etc.) flow
  through to provider clients so feature gating no longer relies on
  string matching.  Server companion published in the same change. (#352)
- **Claude Opus 4.7 support** — provider capabilities, tokenizer
  awareness, and adaptive thinking semantics. (#357 — also in 1.3.1)
- **Dashboard composer refactor** — unified single-flow create from the
  per-node dashboard.  Multi-line textarea + collapsible Options panel
  (model / judge / skill) + paperclip + drag-drop / paste-image + chip
  strip.  Submit-button label dynamically toggles between `Create`
  (empty) and `Send` (text or attachments staged); Enter and click both
  go through the same `dashboardSubmit()`.  Replaces the inconsistent
  prior split where Enter created+sent raw and the button opened a
  separate modal.  Options panel state persists in `localStorage`;
  active non-default selections render as an inline summary chip beside
  the Options button; drag-over shows an explicit "Drop to attach"
  overlay. (#362, #366)
- **Workstream attachments — orphan reservation sweep** — periodic
  background sweep clears `reserved_for_msg_id` on rows whose
  `reserved_at` exceeds a 1-hour threshold, self-healing reservations
  leaked by process crashes between reserve and consume.  Backed by a
  partial index on `(reserved_at) WHERE reserved_at IS NOT NULL` so the
  scan stays cheap as the consumed-history grows.  Threshold tracks
  reservation age, not upload age, so a long-pending fresh send can't
  be racially unreserved. (#363)
- **`SendResponse` extended** — `attached_ids`,
  `dropped_attachment_ids`, `priority`, `msg_id` fields exposed in
  Pydantic + TypeScript SDKs so attachment-aware clients can detect
  partial reservations and dequeue queued messages. (#365)

### Changed

- **`plan_model` and `task_model` now split** from the conversation
  model and from each other — operators who rely on a single model for
  all three should set both `plan_model` and `task_model` explicitly in
  their config; otherwise both default to the conversation model so
  behaviour is unchanged. (#54dd557)
- **Channel notify-on-complete `channel_type` is no longer hardcoded
  in the admin UI** — operators creating notify targets through the
  skill admin form previously got `channel_type: "discord"` regardless
  of what they wanted.  Existing skill JSON values are unaffected; only
  newly created targets through the form differ. (#365)
- **Slack adapter approval previews** — capped at 600 chars per item
  with a 2700-char total budget so multi-tool approval batches never
  exceed Slack's 3000-char `section.text` limit.  Truncated batches
  show a `…and N more (preview truncated)` suffix. (#365)
- **PostgreSQL deployment image** swapped from `bitnami/pgbouncer` to
  `edoburu/pgbouncer` to track upstream releases and reduce image size.
  No config changes required for typical deployments; review your helm
  values if you depend on `bitnami`-specific environment variable
  conventions. (#353)

### Fixed

- **`plan_resolved` SSE broadcast** — when one client resolved a plan
  approval, other clients viewing the same workstream now have the
  approval card dismissed in sync. (#87a9af1)
- **Slack notification reply routing** — one notification reply
  previously pinned every later assistant response for that workstream
  to the notification thread until the bot restarted.  Reply-route
  override now clears on `StreamEndEvent`. (#365)
- **Slack plan-review mrkdwn fence** — plan content containing triple
  backticks (very common — plans often quote code) no longer breaks the
  surrounding fence and lets later content render as live markup.  The
  shared `_sanitize_slack_preview` helper splices a zero-width space
  inside any ``` ``` `` sequence while keeping single backticks
  readable. (#365)
- **Slack-routed workstreams now load the chat-specific system prompt**
  via `client_type="chat"`, matching Discord. (#365)
- **`/v1/api/workstreams/new` no longer emits a phantom
  `ws_created`/`ws_closed` SSE pair** when attachment validation
  rejects a multipart create.  Validation runs before the broadcast so
  failed creates are silent on dashboards. (#362)
- **Multipart Content-Type boundary preservation** in console routing
  proxy — `boundary=` parameter is case-sensitive and was being
  lowercased before forwarding to the upstream node, breaking parsing
  for clients that used mixed-case boundaries (most browsers). (#362)
- **Local-theme contrast for new badge colors** — `.scope-discord` and
  `.scope-slack` first shipped with raw hex that failed WCAG AA on
  light theme (1.8:1 / 2.4:1).  Theme-aware `--discord` / `--slack`
  tokens with proper light variants now pass. (#365)

### Security

- **Slack approval per-user authentication** — only the session owner
  can click Approve/Deny on a Slack tool-approval card.  Without this,
  any channel member with view access could approve dangerous tool
  calls initiated by someone else. (#355)
- **Attachment ownership masking** — cross-user/cross-workstream
  attachment ID lookups return 404 (not 403) so non-owners can't
  enumerate workstream existence by response code. (#356)
- Bumped Debian base image; remaining unfixable `jq` CVEs are
  documented and exception-listed. (#aaea4d3)

### Database migrations

- **`037_workstream_attachments`** — new `workstream_attachments` table
  with the lifecycle columns described above.  Indexes for ws_id,
  pending lookups, message linkage, and reservation scoping.
- **`038_workstream_attachments_reserved_at`** — adds `reserved_at`
  column for the orphan-sweep staleness signal, plus a partial index
  on `reserved_at IS NOT NULL` so the periodic scan is cheap.

Both migrations are additive and idempotent, and the server applies
them automatically on first startup against an existing 1.3.x database.
No manual `alembic upgrade` step is required — though running it
manually beforehand (e.g. as part of a phased deploy) remains safe.

### SDK

Python + TypeScript clients gained:

- `AttachmentUpload` type
- `upload_attachment(ws_id, filename, data, mime_type=None)`
- `list_attachments(ws_id)`
- `get_attachment_content(ws_id, attachment_id) → bytes / Blob`
- `delete_attachment(ws_id, attachment_id)`
- `send(message, ws_id, attachment_ids=...)` (extended)
- `create_workstream(..., attachments=[...])` — multipart variant with
  client-side `ws_id` generation for cluster-routed callers
- Console SDK: `route_create_workstream(attachments=...)`,
  `route_upload_attachment`, `route_list_attachments`,
  `route_get_attachment_content`, `route_delete_attachment`
- Refusal of `attachments + target_node` combination at the SDK
  boundary (the multipart routing layer doesn't honor `target_node`,
  so silently picking the wrong node is now an explicit error)

## [1.3.1]

### Added

- Backport: Claude Opus 4.7 support (provider capabilities, tokenizer,
  adaptive thinking). (#357)
