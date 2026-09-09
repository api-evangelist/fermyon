---
name: fermyon-authenticate-to-fermyon-cloud
description: Obtain and manage a Fermyon Cloud bearer token — via the device-code login flow or a personal access token — and refresh or revoke it.
api: Fermyon Cloud REST API
base_url: https://cloud.fermyon.com
generated: '2026-09-09'
method: generated
source: openapi/_original/fermyon-openapi.yml + https://developer.fermyon.com/cloud/user-settings
operations:
  - 'POST /api/device-codes'
  - 'GET /api/device-codes/{userCode}'
  - 'POST /api/device-codes/activate'
  - 'POST /api/auth-tokens'
  - 'POST /api/auth-tokens/refresh'
  - 'POST /api/personal-access-tokens'
  - 'GET /api/personal-access-tokens'
  - 'DELETE /api/personal-access-tokens/{id}'
---

# Authenticate to Fermyon Cloud

Every one of the 61 operations carries the same top-level requirement: a
`Authorization: Bearer {token}` header. The contract declares this as an `apiKey` scheme
named `Bearer` carrying a JWT. There are no scopes and no OAuth 2.0 metadata.

## Option A — personal access token (best for an agent)

1. `POST /api/personal-access-tokens` with a `CreatePersonalAccessTokenCommand` (`name`).
   The response (`PersonalAccessTokenValue`) is the **only** time the token value is
   returned. Store it in a secret store, never in a config file.
2. `GET /api/personal-access-tokens` lists tokens by `id`, `name` and `createdAt` — values
   are never returned again.
3. `DELETE /api/personal-access-tokens/{id}` revokes one. This is immediate and irreversible.

## Option B — device-code login (what `spin cloud login` drives)

1. `POST /api/device-codes` with a `CreateDeviceCodeCommand` (`clientId`). The response
   (`DeviceCodeItem`) returns `deviceCode`, `userCode`, `verificationUrl`, `expiresIn` and
   `interval`.
2. Show the user `verificationUrl` and `userCode`. Poll no faster than `interval` seconds
   and give up after `expiresIn`.
3. The user completes activation; `POST /api/device-codes/activate` takes the `userCode`.
   `GET /api/device-codes/{userCode}` returns `DeviceCodeDetails` for the confirmation screen.
4. `POST /api/auth-tokens` returns a `TokenInfo` (`token`, `refreshToken`, `expiration`).
   Refresh with `POST /api/auth-tokens/refresh` before `expiration`.

## Rules an agent must follow here

- **This flow is RFC 8628-shaped but not RFC 8628.** The response fields are the device
  authorization grant's, camelCased, but the endpoints and the token exchange are not the
  RFC's. A generic OAuth device-flow client will not work — see
  `conformance/fermyon-conformance.yml`.
- **A token is all-or-nothing.** No scopes are declared, and the Akamai Functions side of the
  product states plainly that there is no RBAC and "any member can permanently delete any
  application in the account". A token you hold can destroy every app in the account.
- Never log or echo a token value, a `refreshToken`, or a `deviceCode`.
