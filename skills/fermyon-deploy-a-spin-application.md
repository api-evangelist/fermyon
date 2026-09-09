---
name: fermyon-deploy-a-spin-application
description: Deploy a Spin application to Fermyon Cloud end to end — push the build to the Cloud OCI registry, register it as a revision, and point a channel at it so it goes live.
api: Fermyon Cloud REST API
base_url: https://cloud.fermyon.com
generated: '2026-09-09'
method: generated
source: openapi/_original/fermyon-openapi.yml + https://developer.fermyon.com/cloud/deployment-concepts
operations:
  - 'POST /api/apps'
  - 'POST /api/oci/{name}/blobs/uploads'
  - 'PATCH /api/oci/{name}/blobs/uploads/{digest}'
  - 'PUT /api/oci/{name}/blobs/uploads/{digest}'
  - 'PUT /api/oci/{name}/manifests/{reference}'
  - 'POST /api/revisions'
  - 'POST /api/channels'
  - 'PATCH /api/channels/{id}'
  - 'GET /api/apps/{id}'
---

# Deploy a Spin application to Fermyon Cloud

> **Operation identifiers.** Fermyon's published contract declares no `operationId` on any
> of its 61 operations, so every step below is anchored to the HTTP method and path exactly
> as the contract states them. Do not expect named operation ids from the provider.
> `overlays/fermyon-*-overlay.yaml` carries API Evangelist's computed ids if you need them.

## Before you start

- Authenticate first. Every operation requires `Authorization: Bearer {token}`; see
  `fermyon-authenticate-to-fermyon-cloud.md`.
- Send `Api-Version: 1.0` on each request. It is a declared header with that default on
  every operation.
- **The supported path is the CLI.** `spin cloud deploy` performs this whole sequence.
  Drive the REST API directly only when you need to script something the plugin does not do.

## Steps

1. **Create the application** — `POST /api/apps` with a `CreateAppCommand`
   (`name`, `storageId`). The response carries the app `id` (a bare UUID with no type prefix)
   and its `subdomain`. If the app already exists, list first with `GET /api/apps`
   (`searchText`, `exactMatch=true`) rather than creating a duplicate.
2. **Start an OCI blob upload** — `POST /api/oci/{name}/blobs/uploads`. Fermyon Cloud uses
   the OCI Image and OCI Distribution Specifications as its transport; the upload session
   semantics are the spec's, mounted at `/api/oci` instead of `/v2`.
3. **Stream the layers** — `PATCH /api/oci/{name}/blobs/uploads/{digest}` for chunks, then
   `PUT /api/oci/{name}/blobs/uploads/{digest}` to complete by digest.
4. **Push the manifest** — `PUT /api/oci/{name}/manifests/{reference}`. Check first with
   `HEAD /api/oci/{name}/manifests/{reference}` to avoid re-pushing an existing build.
5. **Register the revision** — `POST /api/revisions` with a `RegisterRevisionCommand`
   (`appStorageId`, `revisionNumber`). A revision is immutable; this is the build you can
   later roll back to.
6. **Point a channel at it** — `POST /api/channels` (`CreateChannelCommand`: `appId`,
   `name`, `revisionSelectionStrategy`, `activeRevisionId`) for a new deployment, or
   `PATCH /api/channels/{id}` to move an existing channel to the new revision. The channel,
   not the revision, is what is publicly addressable.
7. **Confirm** — `GET /api/apps/{id}` and read `healthStatus`, `lastDeployed` and the
   channel's `activeRevision`.

## Rules an agent must follow here

- **No idempotency.** There is no `Idempotency-Key` header and no replay protection on any
  of the 61 operations. If step 1 or 5 times out, do **not** blindly retry — read back with
  `GET /api/apps` or `GET /api/revisions` and only then decide.
- **No documented errors.** The contract declares only `200` responses. There is no error
  schema, no `application/problem+json`, and no documented status code for a failure. Treat
  any non-2xx as opaque, log the raw body, and stop rather than branching on a code you
  assumed.
- **Deployment quotas are real and low.** 10 deployments per minute and 100 per hour on both
  Starter and Growth; 5 applications on Starter, 100 on Growth. See
  `rate-limits/fermyon-rate-limits.yml`.
- **Rolling forward is reversible; deleting is not.** To undo a bad deploy, re-point the
  channel (see `fermyon-roll-back-a-deployment.md`). Never reach for `DELETE /api/apps/{id}`.
