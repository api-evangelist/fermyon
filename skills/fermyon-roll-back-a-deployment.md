---
name: fermyon-roll-back-a-deployment
description: Undo a bad Fermyon Cloud deploy by re-pointing the channel at a previous revision, and stop a channel entirely if the rollback target is not yet known.
api: Fermyon Cloud REST API
base_url: https://cloud.fermyon.com
generated: '2026-09-09'
method: generated
source: openapi/_original/fermyon-openapi.yml + https://developer.fermyon.com/cloud/deployment-concepts
operations:
  - 'GET /api/revisions'
  - 'GET /api/channels/{id}'
  - 'PATCH /api/channels/{id}'
  - 'PUT /api/channels/{channelId}/desired-status'
  - 'GET /api/channels/{id}/healthz'
  - 'GET /api/apps/{id}/events'
---

# Roll back a Fermyon Cloud deployment

This is the one genuine reversal path in the Fermyon Cloud API. It works because revisions
are immutable and a channel merely *selects* one — so undoing a deploy is a pointer change,
not a restore.

## Steps

1. **Find the current state** — `GET /api/channels/{id}`. Read `activeRevision`
   (a `RevisionItem` with `revisionNumber`) and `revisionSelectionStrategy`.
2. **List candidate revisions** — `GET /api/revisions` for the app. Pick the
   `revisionNumber`/`id` you want to return to.
3. **Re-point the channel** — `PATCH /api/channels/{id}` with a `PatchChannelCommand`
   setting `revisionSelectionStrategy` to `UseSpecifiedRevision` and `activeRevisionId` to
   the target revision's id. Setting the strategy explicitly matters: if the channel is on
   `UseRangeRule`, a later matching build will silently move it forward again.
4. **Verify** — `GET /api/channels/{id}/healthz`, then `GET /api/apps/{id}/events` for the
   `Update` event.

## If you need to stop serving immediately

`PUT /api/channels/{channelId}/desired-status` with `desiredStatus: Dead` takes the channel
out of service. It is fully reversible — set it back to `Running`. Prefer this to any
destructive action while you work out the rollback target.

## Rules an agent must follow here

- **No retention window is published.** Fermyon does not state how long revisions are kept,
  so there is no documented floor on how far back you can roll. Always confirm the target
  revision still exists with `GET /api/revisions` before you `PATCH`; never promise a user a
  rollback horizon.
- **`DELETE` is not a rollback.** `DELETE /api/apps/{id}` is documented as permanent:
  "this is a permanent action and cannot be undone. The default key/value store and all
  application variables will also be deleted"
  (<https://developer.fermyon.com/cloud/delete>). There is no soft-delete and no restore.
- The same no-idempotency and no-documented-errors rules apply as everywhere else on this
  API; see `conventions/fermyon-conventions.yml`.
