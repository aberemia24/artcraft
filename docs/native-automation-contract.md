# Native automation contract (proposal)

Status: design only. The commands, tools, endpoints, and schemas below are not
implemented. Scope and prerequisites are in the
[Codex/native-control plan](codex-native-automation-plan.md).

## One command service, two external adapters

`artcraftctl` provides shell commands and an MCP stdio server. Both call the
same app command dispatcher through user-restricted local IPC. Desktop buttons
and the integrated assistant use those same domain handlers. CLI commands are
explicit operations; MCP tools expose the same arguments and results without
requiring an agent to construct shell syntax.

Rust handles assets, jobs, provider state, and persistence. The registered live
frontend handles scene/canvas/timeline edits through existing engine actions.
An editor operation succeeds only after the frontend returns an acknowledged
result and updated revision. Do not evaluate arbitrary JavaScript, expose a raw
Tauri-invoke passthrough, or write directly into Zustand/Three.js internals.

## Proposed CLI examples

```text
artcraftctl instances list --json
artcraftctl --instance <id> doctor --json
artcraftctl --instance <id> capabilities --json
artcraftctl --instance <id> assets import --path <file> --json
artcraftctl --instance <id> scene inspect --document <id> --json
artcraftctl --instance <id> scene apply --document <id> --file <commands.json> --json
artcraftctl --instance <id> scene snapshot --document <id> --output <png> --json
artcraftctl --instance <id> project save --document <id> --path <project> --json
artcraftctl --instance <id> jobs get --job <id> --json
artcraftctl --instance <id> jobs cancel --job <id> --json
artcraftctl --instance <id> mcp --stdio
```

`--json` prints exactly one result envelope for a normal command. A separate
`jobs watch --format jsonl` streams versioned events. Diagnostics go to stderr;
MCP stdout contains only protocol messages. No implicit instance selection if
multiple apps run. No implicit app launch, OAuth login, model change, generation,
cloud synchronization, or payment from a read-only command.

## Operation catalog

Names below describe the proposed contract, not an existing command inventory.

| Domain         | Operations                                                                         | Release scope                                                               |
| -------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| App            | `app.info`, `app.capabilities`, `app.editors.list`                                 | MVP: version, active documents, readiness, provider/capability status.      |
| Assets         | `assets.import`, `assets.list`, `assets.get`, `assets.export`                      | MVP: local asset IDs, metadata, previews, paths within allowed roots.       |
| Project        | `project.open`, `project.save`, `project.checkpoint`                               | MVP: explicit document targeting, atomic local saves, reloadable assets.    |
| Scene          | `scene.inspect`, `scene.apply`, `scene.snapshot`, `scene.undo`, `scene.redo`       | MVP: object import/transform, visibility, selection, camera and lights.     |
| Generation     | `generation.submit`, `generation.capabilities`                                     | Only when the provider/modality has passed its acceptance gate.             |
| Jobs           | `jobs.get`, `jobs.list`, `jobs.cancel`, `jobs.events`                              | Persistent IDs, artifacts, progress, terminal status and recovery detail.   |
| Canvas         | `canvas.inspect`, `canvas.apply`, `canvas.export`                                  | Later: layers, masks, local references, edits via the canvas store actions. |
| Timeline       | `timeline.inspect`, `timeline.apply`, `timeline.render`                            | Later: clips, tracks, duration, transforms, preview, native export jobs.    |

MCP uses names such as `artcraft_scene_inspect`, `artcraft_scene_apply`, and
`artcraft_jobs_get`. Tools include descriptions, JSON input/output schemas,
structured results, and accurate read-only/destructive/idempotent annotations.
Resource reads expose document descriptors, capability descriptions, previews,
and artifacts using stable app resource URIs. Return an MCP image content block
or an accessible image resource for a scene capture so the agent can inspect it;
a local path alone may not be readable to a remote agent. Remote access is not
included in the MVP.

## Versioned command envelope

```json
{
  "protocol_version": 1,
  "request_id": "<uuid>",
  "instance_id": "<instance-id>",
  "document_id": "<document-id>",
  "expected_revision": 12,
  "operation": "scene.apply",
  "arguments": {
    "commands": [
      {
        "type": "object.transform",
        "object_id": "<stable-object-id>",
        "position": [0, 1, 0],
        "rotation_radians": [0, 0.5, 0],
        "scale": [1, 1, 1]
      }
    ]
  }
}
```

Declare units, axes, quaternion/Euler conventions, bounds, and required fields
in each operation schema. Resolve IDs through the registered document adapter.
`scene.inspect` returns stable IDs, types, transforms, parent relationships,
asset references, camera/light properties, supported actions, and revision.

For mutations, reject a stale `expected_revision` before changing state.
Prevalidate a batch, capture undo state, apply it as one history entry, and
restore that state on failure. If an engine cannot restore atomically, keep its
batch operation unavailable until it can, rather than claiming atomic success.
User edits and agent edits use the same revision/history accounting.

## Results, errors, and jobs

```json
{
  "protocol_version": 1,
  "request_id": "<uuid>",
  "ok": true,
  "result": {
    "document_id": "<document-id>",
    "revision": 13,
    "changed_object_ids": ["<stable-object-id>"]
  }
}
```

Errors use `ok: false` and an `error` object with `code`, `message`, `retryable`,
and safe `details`. Define stable codes including `app_not_running`,
`ambiguous_instance`, `protocol_mismatch`, `editor_not_ready`, `document_closed`,
`revision_conflict`, `invalid_argument`, `asset_not_found`,
`unsupported_capability`, `authentication_required`, `rate_limited`,
`permission_denied`, `timeout`, `cancelled`, and `provider_error`.
Transport/protocol failures remain separate from a successfully delivered domain
error. CLI exit codes distinguish usage errors, domain errors, and unavailable
transport. MCP reports domain failures as tool errors with the same safe code.

Generation/rendering returns an app-owned `job_id` after durable local enqueue.
Suggested states: `queued`, `running`, `succeeded`, `failed`, `cancelled`, and
`needs_attention`. Results list local artifact IDs, MIME type, dimensions or
duration, content hash, provider/model, provenance, and output paths/resources.
Enqueue success does not mean generation completed. Validate artifacts before
marking a media job successful; never manufacture a cloud token to satisfy the UI.

Persist request IDs and normalized input hashes before submission. The same ID
with the same input returns the existing result/job; the same ID with changed
input is rejected. Use provider idempotency only where actually supported.
After a crash or lost provider response, reconcile using a known provider job ID.
If acceptance is ambiguous and cannot be reconciled, mark `needs_attention` and
do not automatically submit again. Cancellation reports whether the provider
acknowledged it; local interruption alone does not prove a charge was avoided.

Persist monotonic job-event sequence IDs so reconnecting clients can resume from
an event cursor. Keep one isolated SQLite-backed task/persistence owner in the
desktop app. The CLI and MCP must not write into the live task database directly.

## Local access and lifecycle

- Named pipe/Unix socket restricted to the current user. Discovery is per app
  profile and instance. App control authentication is separate from ChatGPT
  authentication; OpenAI access tokens never serve as IPC credentials.
- Restrict file imports/exports to configured roots, resolve canonical paths,
  account for Windows junctions/reparse points, and require explicit overwrite
  intent. Large artifacts use local streams/resources, not enormous JSON blobs.
- Register frontend handlers when the relevant editor is ready and unregister
  on close/reload. Correlate requests/replies; stale handlers and late callbacks
  cannot mutate another document. Renderer-dependent operations need an open
  native window in the MVP; do not advertise a headless renderer.
- Built-in Codex access stays scoped to this app profile/project; do not inherit
  unrelated MCP servers, credentials, or shell permissions. Tool authorization
  is operation-specific. Existing service choices and usage are visible before
  a chargeable job is submitted; no surprise paid-provider fallback.
- Read-only inspection stays available without a ChatGPT login. Mutation scopes
  cover local editor work; cloud sync and chargeable generation have separate
  policy. Log safe request/job IDs and operation names, never secrets.

## Acceptance scenario

Against the real Windows Tauri app, an external Codex agent connects by MCP,
lists instances/editors/capabilities, imports a local GLB, inspects its object ID,
applies a transform and camera change at the inspected revision, receives and
views a PNG capture, undoes the edit, reapplies it, saves locally, then reopens
the project and verifies state. Repeat the same operations through the CLI.

Run this scenario without ArtCraft login and with ArtCraft endpoints unavailable.
Then prove a stale revision fails without mutation, closing the editor produces
a structured error, a second instance stays untouched, and repeat requests do
not duplicate edits/jobs. Media generation is an additional acceptance scenario
only after its selected provider's capability has been confirmed.
