# Threat model drift — service token

## 1. Scope

Feature: service token.

Baseline pin: `infisical/v0.42.0` (commit `735cf093f0`). Recorded as `ServiceTokenDataV3` / `AuthMode.SERVICE_ACCESS_TOKEN`.

Current pin: `infisical/v0.47.0-postgres` (commit `041535bb47`). Recorded as `TableName.ServiceToken` / `AuthMode.SERVICE_TOKEN` / `ActorType.SERVICE`.

Inputs: `serviceTokenDfd-v0.42.0.md`, `serviceTokenDfd-v0.47.0-postgres.md`, `securityAssumptions.md`, `serviceTokenThreatModel.md`.

Excluded: machine identities and universal auth; user login except as stores read during JWT validation; secret-approval requests (the service throws when `actor === ActorType.SERVICE`); Mongo models in `pg-migrator`.

This report does not replace the threat model of record.

## 2. Discover

IDs below are the authors' IDs. Correspondence is by function.

### External entities

| Baseline | Current | Baseline name | Current name |
|---|---|---|---|
| E1 | E1 | User with JWT | User with JWT |
| E2 | E2 | Agent / workload | Workload |
| E3 | E4 | License server | License server |
| — | E3 | — | PostHog |

### Processes

| Baseline | Current | Baseline name | Current name |
|---|---|---|---|
| P1 | P3 | Validate user JWT | Validate user JWT |
| P2 | P4 | Validate access token | Validate service token |
| P3 | P6, P7 | Authorize project action | Authorize token admin; authorize ST secret access |
| P4 | P9 | Create token | Create service token |
| P5 | — | Update token | — |
| P6 | P11 | Delete token | Delete service token |
| P7 | P10 | List tokens | List service tokens |
| P8 | — | Refresh tokens | — |
| P9 | P12 | Return project key | Return token details |
| P10 | P16 (plan read) | Resolve license plan | Persist audit log (calls `getPlan`) |
| P11 | P16 | Write audit log | Persist audit log |
| P12 | — | Mint JWT | — |
| P13 | P14, P15 | Serve secrets | Serve encrypted secrets; serve raw secrets |
| P14 | — (FK cascade) | Cascade delete | `service_tokens.projectId ON DELETE CASCADE` |
| P15 | — | Agent refresh loop | — |
| P16 | P8 | Client key wrap | Wrap workspace key |
| P17 | P1 | Assign request IP | Assign request IP |
| P18 | — | List audit-log actors | — |
| — | P2 | — | Classify Authorization |
| — | P5 | — | Bind actor, permission, audit |
| — | P13 | — | Single-scope query override |
| — | P17 | — | Emit telemetry |
| — | P18 | — | Decrypt secrets |
| — | P19 | — | Write managed Secret |

### Data stores

| Baseline | Current | Baseline name | Current name |
|---|---|---|---|
| D1 | D4 | ServiceTokenDataV3 | service_tokens |
| D2 | D4 (`encryptedKey`/`iv`/`tag`) | ServiceTokenDataV3Key | columns on service_tokens |
| D3 | D7 (user path only) | Role | project_memberships / project_roles |
| D4 | — (project row via DAL) | Workspace | — |
| D5 | — (org via license) | Organization | — |
| D6 | D5 | User | users |
| D7 | D6 | TokenVersion | auth_token_sessions |
| D8 | D12 | AuditLog | audit_logs |
| D9 | D13 | License plan cache | Redis plan cache |
| D10 | D15 | HMAC signing secret | AUTH_SECRET, SALT_ROUNDS, ENCRYPTION_KEY |
| D11 | D2 | Browser PRIVATE_KEY | User private key |
| D12 | D3 | Credential file | Four-part token |
| D13 | — | Sink access token | — |
| D14 | — | Rendered templates | — |
| D15 | D10 | Secret | secrets / folders / imports |
| D16 | D11 | Bot | project_bots |
| D17 | D7 | Membership | project_memberships |
| — | D1 | — | User JWT |
| — | D8 | — | project_environments |
| — | D9 | — | project_keys |
| — | D13 | — | Redis queue / counters (also plan cache) |
| — | D14 | — | Managed Kubernetes Secret |
| — | D16 | — | Operator infisicalToken Secret |

### Trust boundaries

| Baseline | Current | Baseline name | Current name |
|---|---|---|---|
| B1 | B1 | User client | User client |
| B2 | B2 | Workload host | Workload host |
| B3 | B4 (P1) | API edge | folded into API process |
| B4 | B4 | Authenticated API | API process (classify + auth) |
| B5 | — | Unauthenticated refresh | — |
| B6 | B5 | MongoDB | Postgres |
| B7 | B7 | License server | Third party (license + PostHog) |
| B8 | B4 (D15) | HMAC signing secret | process secrets inside API process |
| — | B3 | — | Kubernetes API |
| — | B6 | — | Redis |

### Flows

Baseline F1–F3, F4, F17–F25 map to current F2–F5, F19–F26 (provision create). Baseline F26–F32 (update) have no current counterpart. Baseline F33–F39 map to F32–F36 (delete). Baseline F40–F43 map to F27–F31 (list). Baseline F44–F53, F46–F52 (agent refresh) have no current counterpart. Baseline F54–F64 map to F50–F68 (secret use). Baseline F65–F68 map to F39–F48 (token details). Baseline F69–F72 map to the Postgres FK cascade (no flow). Baseline F73–F76 (audit-actor list) have no current counterpart. Current F37, F76, F77 (operator) and F69–F74 (telemetry / queue / license) have no baseline counterpart. Current F13 (single-scope override) has no baseline counterpart.

## 3. Compare

### External entities

- **unchanged:** E1. E3→E4 license server.
- **changed:** E2. The Infisical agent refresh loop is gone. Callers are CLI, generic HTTP, and the Kubernetes operator.
- **added:** E3 PostHog.

### Processes

- **unchanged:** none with the same data, privilege, and binding.
- **changed:** P1→P3 (session table, not Mongo TokenVersion). P2→P4 (bcrypt of `st.{id}.{secret}`, not JWT). P3 split into P6 (user CASL on `service-tokens`) and P7 (token `projectId` equality + scope CASL). P4→P9 (scopes and permissions, not role / trustedIps / refresh JWT). P6→P11. P7→P10 (now requires `Read` on `service-tokens`). P9→P12 (returns `encryptedKey`/`iv`/`tag` and `secretHash` on the GET schema). P11→P16 (BullMQ, skip insert when retention is 0). P13→P14/P15 (encrypted and raw; raw uses the project bot). P16→P8 (AES-256-GCM wrap; client appends a fourth segment). P17→P1 (header list is longer: `cf-connecting-ip` first, then `x-forwarded-for` and others).
- **added:** P2 prefix classify. P5 actor/audit bind. P13 single-scope query override of `workspaceId`/`environment`/`secretPath`. P17 telemetry. P18 client decrypt. P19 operator write.
- **removed:** P5 update. P8 unauthenticated refresh. P12 JWT mint. P15 agent loop. P18 audit-actor list. P14 as a process (cascade is a foreign key).

### Data stores

- **changed:** D1→D4 Mongo document to Postgres row with `secretHash`, jsonb `scopes`, `permissions[]`, optional `expiresAt`, `projectId` FK. D2 merged into D4. D8→D12. D9→D13. D10→D15 (HMAC secret plus bcrypt cost plus bot unwrap key). D12→D3 (`st.{id}.{secret}.{aes-key}`). D15→D10. D16→D11. D17→D7.
- **added:** D1 user JWT as a named store. D8 environments. D9 project keys. D13 Redis. D14 managed Secret. D16 operator token Secret.
- **removed:** D13 access-token sink. D14 rendered templates.

### Trust boundaries

- **changed:** B3 (API edge) is no longer a separate subgraph; IP assignment is P1 inside B4. B4 now includes unauthenticated classification. B6→B5 engine change. B8 is not a separate boundary; process secrets sit in B4. B7 now also carries PostHog.
- **added:** B3 Kubernetes API. B6 Redis.
- **removed:** B5 unauthenticated refresh.

Material differences: the presented credential is a durable three-part secret, not a short-lived access JWT. Environment reach is stored scopes, not a project role. Secret authorization is bound to `projectId`. There is no per-token IP allowlist. Authentication cost is bcrypt. The operator path stores the four-part token and plaintext secrets in the cluster.

## 4. Validate

### Threats

#### T1 — Cross-project secret access

- **Target then / now:** P3, P13, F54, F59 / P7, P14, P15, F51, F50
- **Narrative still valid:** no. `getServiceTokenProjectPermission` throws when `serviceToken.projectId !== projectId` (`permission-service.ts:182-185`). P13 overwrites `workspaceId` to the token's project on single non-glob scope (`secret-router.ts:63-70`). Multi-scope and glob requests still take the query `workspaceId`, then fail the same check.
- **Recorded mitigations:** none
- **Strength then / now:** empty / M11 full
- **Evidence:** `backend/src/ee/services/permission/permission-service.ts:178-189`; `backend/src/services/secret/secret-service.ts:406-410`

#### T2 — Built-in role covers every environment

- **Target then / now:** P3, P13 / no counterpart. Tokens do not carry `viewer` / `member` / `admin`.
- **Narrative still valid:** no. Create requires `scopes` min 1 (`service-token-router.ts:77-83`). P7 builds CASL from those scopes (`project-permission.ts:233-268`).
- **Recorded mitigations:** M6 conditional
- **Strength then / now:** M6 conditional / — (surface gone)
- **Evidence:** `backend/src/server/routes/v2/service-token-router.ts:77-83`; `backend/src/ee/services/permission/project-permission.ts:233-268`

#### T3 — Stolen refresh JWT at the unauthenticated grant

- **Target then / now:** P8, D12, F46, B5 / none
- **Narrative still valid:** no. No `/me/token` route. No refresh JWT. `AuthTokenType.SERVICE_ACCESS_TOKEN` / `SERVICE_REFRESH_TOKEN` have no `inject-identity` branch (`inject-identity.ts:66-88`).
- **Recorded mitigations:** M1, M2, M8, M10
- **Strength then / now:** partial / conditional / — (surface gone)

#### T4 — World-readable access-token sink

- **Target then / now:** P15, D13, F53 / none
- **Narrative still valid:** no. No agent, no 0644 sink.
- **Recorded mitigations:** M3
- **Strength then / now:** M3 partial / — (surface gone)

#### T5 — Source-IP spoof through P17

- **Target then / now:** P17, P3, F5, B3 / P1
- **Narrative still valid:** the header-trust half is still true. P1 still takes the first present header in `headersOrder`, starting with `cf-connecting-ip` (`ip.ts:5-29`). The allowlist-bypass half is not. There is no `trustedIps` on the token and `checkIPAgainstBlocklist` is not called from `fnValidateServiceToken` or `getServiceTokenProjectPermission`.
- **Recorded mitigations:** none
- **Strength then / now:** empty / — (allowlist-bypass surface gone; spoof now only poisons audit IP, which is A6)

#### T6 — Possession equals access on the default origin

- **Target then / now:** D1, P4, P10 / D4, P9
- **Narrative still valid:** yes, stronger. Create stores no CIDR (`service-token-service.ts:76-87`; schema `service-tokens.ts:10-25`). Project `trusted_ips` are not read on this path. A stolen three-part token works from any address.
- **Recorded mitigations:** M7
- **Strength then / now:** M7 conditional / none
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:76-87`, `130-145`; `backend/src/db/schemas/service-tokens.ts:10-25`

#### T7 — Shared-token actions are one actor

- **Target then / now:** P2, P11, D8 / P4, P16, D12
- **Narrative still valid:** yes. Actor metadata is `{ name, serviceId }` (`audit-log.ts:51-58`). Secret routes now call `createAuditLog` (`secret-router.ts:85-96`). Two workloads still share one actor.
- **Recorded mitigations:** none
- **Strength then / now:** empty / empty
- **Evidence:** `backend/src/server/plugins/audit-log.ts:51-58`

#### T8 — Audit rows expire at write time

- **Target then / now:** P11, D8, P10 / P16, D12
- **Narrative still valid:** yes. Worker computes `ttl = plan.auditLogsRetentionDays * MS_IN_DAY` and returns without insert when `ttl === 0` (`audit-log-queue.ts:45-48`). On-prem default `auditLogsRetentionDays` is 0 (`licence-fns.ts:25`).
- **Recorded mitigations:** M9
- **Strength then / now:** M9 conditional / M9 conditional
- **Evidence:** `backend/src/ee/services/audit-log/audit-log-queue.ts:45-61`; `backend/src/ee/services/license/licence-fns.ts:25`

#### T9 — Unauthenticated refresh as a cheap flood

- **Target then / now:** P8, B5 / none
- **Narrative still valid:** no. The grant is gone.
- **Recorded mitigations:** none
- **Strength then / now:** empty / — (surface gone)

#### T10 — Token inventory without a project permission check

- **Target then / now:** P7, F40 / P10, F27
- **Narrative still valid:** no. `getProjectServiceTokens` requires `Read` on `service-tokens` (`service-token-service.ts:123-124`). Viewer includes that action (`project-permission.ts:216`).
- **Recorded mitigations:** none
- **Strength then / now:** empty / M13 full
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:122-127`; `backend/src/server/routes/v1/project-router.ts:344-366`

#### T11 — Project-admin token plus T1

- **Target then / now:** P3 / none
- **Narrative still valid:** no. Tokens have no `role: admin`. P7 CASL is only Secrets on scopes (`project-permission.ts:233-268`). T1's cross-project step is closed.
- **Recorded mitigations:** none
- **Strength then / now:** empty / — (surface gone)

#### T12 — Credential JSON is a workspace-key package

- **Target then / now:** P16, D12, F3 / none as recorded
- **Narrative still valid:** no. The download is no longer `{ public_key, private_key, refresh_token }`. The same property exists on D3 as a four-part string; that is T14.
- **Recorded mitigations:** none
- **Strength then / now:** empty / — (packaging gone)

#### T13 — User issues or revokes a token without project standing

- **Target then / now:** P4, P5, P6 / P9, P11
- **Narrative still valid:** yes on create and delete. Update is gone. Create requires `Create` on `service-tokens` and `Create` on `Secrets` per scope (`service-token-service.ts:50-57`). Delete requires `Delete` on `service-tokens` against the row's `projectId` (`service-token-service.ts:98-104`). Both go through `getUserProjectPermission`, which throws when the user is not in the project (`permission-service.ts:148-150`).
- **Recorded mitigations:** M4, M5
- **Strength then / now:** M4 full, M5 full / M4 full, M5 full
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:50-57`, `94-107`; `backend/src/ee/services/permission/permission-service.ts:148-162`

### Mitigations

#### M1 — HMAC verify and token-type split

- **Status:** gone
- **Upholds:** A7 (recorded). Current auth is bcrypt (`service-token-service.ts:140-141`).
- **Evidence:** no `serviceTokenV3` validator. `inject-identity.ts:56-61` selects `st.` then `fnValidateServiceToken`.

#### M2 — Server-side `isActive`, `expiresAt`, `tokenVersion`

- **Status:** gone as recorded. `expiresAt` remains inside P4 and deletes the row (`service-token-service.ts:135-138`). No `isActive`. No `tokenVersion`.
- **Upholds:** no longer A7/A8 as stated.

#### M3 — Access JWT `expiresIn: accessTokenTTL`

- **Status:** gone. No access JWT.

#### M4 — CASL `service-tokens` on create, update, delete

- **Status:** present, moved, narrower (no update), stronger on list (see M13)
- **Upholds:** A5, A14
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:50-51`, `104`

#### M5 — User membership at P3

- **Status:** present, moved to `getUserProjectPermission`
- **Upholds:** A5, A14
- **Evidence:** `backend/src/ee/services/permission/permission-service.ts:148-150`

#### M6 — Custom-role environment conditions

- **Status:** gone for this feature. Token CASL does not use project roles.

#### M7 — `trustedIps` checked at P3

- **Status:** gone. Schema has no `trustedIps`. `checkIPAgainstBlocklist` is used by identity-UA, not service tokens.

#### M8 — Operator revocation paths

- **Status:** present, weaker. Delete and optional `expiresAt` remain. `isActive`, `tokenVersion`, and refresh rotation are gone.
- **Upholds:** A8
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:94-107`, `135-138`; `backend/src/server/routes/v2/service-token-router.ts:87`

#### M9 — Audit document on token CRUD

- **Status:** present, stronger (secret GET/create/update/delete also emit). Still skipped when retention is 0. Enforced write path is default; durability is licensed.
- **Upholds:** A9
- **Evidence:** `backend/src/server/routes/v2/service-token-router.ts:106-116`, `143-153`; `backend/src/server/routes/v3/secret-router.ts:85-96`; `backend/src/ee/services/audit-log/audit-log-queue.ts:45-48`

#### M10 — Refresh-token rotation

- **Status:** gone

### Assumptions

#### A1 — Environment reach

- **Verdict:** fails as stated. Tokens no longer inherit built-in role coverage of every environment. Reach is `scopes` × `permissions` (`permission-service.ts:186-188`; `project-permission.ts:240-266`). Default UI scope is one environment at `/` (`AddServiceTokenModal.tsx:92-97`).

#### A2 — Two-tiered lifetime

- **Verdict:** inapplicable. No access JWT, no refresh JWT, no `accessTokenTTL`.

#### A3 — Ephemeral presented credential

- **Verdict:** inapplicable. The presented credential is the durable three-part secret. The agent sink is gone.

#### A4 — Partial project binding

- **Verdict:** fails. Secret authorization is bound to the token's `projectId` (`permission-service.ts:182-185`).

#### A5 — No org escalation

- **Verdict:** holds. `getOrgPermission` has no `SERVICE` case (`permission-service.ts:118-129`). Token CASL is Secrets on scopes only.

#### A6 — Open network origin

- **Verdict:** fails in the enforceable-allowlist clause. Origin is unrestricted and not enforceable per token. The IP-determination clause holds (`ip.ts:20-29`).

#### A7 — Stateful JWT authentication

- **Verdict:** fails as stated. Mechanism is bcrypt and `expiresAt`, then `lastUsed` (`service-token-service.ts:130-145`).

#### A8 — Three-form revocation

- **Verdict:** fails in the generational clause. Immediate delete and optional `expiresAt` remain. No `tokenVersion`. None is applied by policy (`expiresIn` nullable, `service-token-router.ts:87`).

#### A9 — Per-token attribution

- **Verdict:** holds. Actor is `{ name, serviceId }` plus IP and user agent (`audit-log.ts:51-58`). Retention still follows the plan (`audit-log-queue.ts:45-48`). Secret use is now also written.

#### A10 — Low-cost authentication

- **Verdict:** fails. `bcrypt.compare` with `SALT_ROUNDS` default 10 (`service-token-service.ts:140`; `env.ts:21`).

## 5. Identify

### T14 — Four-part token is a workspace-key package

- **Target:** P8, D3, F25, P18, D16
- **STRIDE:** Information disclosure
- **Exploits:** A11, A13
- **Narrative:** On create the client AES-256-GCM-encrypts the workspace key with 16 random bytes and appends the hex key to `st.{id}.{secret}` (`AddServiceTokenModal.tsx:136-157`; `tokens.go:124-159`; `queries.tsx:42`). Anyone who reads D3 or D16 has the API secret and the unwrap key. P18 decrypts `encryptedKey` then each secret (`cli/packages/util/secrets.go:20-66`; `k8-operator/packages/util/secrets.go:53-92`).
- **Impact:** Token use plus client-side decryption of every secret the token can fetch on the encrypted path.
- **Mitigations:** none
- **Strength:** —

### T15 — bcrypt compare as a request-path flood

- **Target:** P4, B4
- **STRIDE:** Denial of service
- **Exploits:** A10, A6
- **Narrative:** `inject-identity` runs on every request. A Bearer value starting `st.` calls `fnValidateServiceToken` (`inject-identity.ts:56-61`, `115-116`). A known id (from list, create, or logs) loads D4 and runs `bcrypt.compare` at cost 10 (`service-token-service.ts:131-141`). Success also writes `lastUsed`. The global limiter is 600/min keyed on `req.realIp` (`rateLimiter.ts:12-18`; `app.ts:71`), and P1's address is caller-influenced.
- **Impact:** CPU and write amplification on any route that accepts `SERVICE_TOKEN`, keyed on a spoofable address.
- **Mitigations:** M15
- **Strength:** M15 partial — bounds one address; does not bound cost per attempt; fails if T5-style headers are accepted

### T16 — Import list ignores token scopes

- **Target:** P14, P15, F56
- **STRIDE:** Information disclosure
- **Exploits:** A12, A1
- **Narrative:** A token scoped to folder F calls GET secrets with `include_imports`. P7 allows F. `getSecrets` keeps every import when `actor === ActorType.SERVICE` (`secret-service.ts:418-430`, `499-510`). P15 decrypts those rows with the project bot (`secret-service.ts:737-754`).
- **Impact:** Secrets from environments or paths outside the token's scopes, if F imports them.
- **Mitigations:** M12
- **Strength:** M12 partial — the caller must still be allowed on F; imports from F are not filtered

### T17 — Stolen durable three-part token

- **Target:** P4, D3, F38, F39
- **STRIDE:** Spoofing
- **Exploits:** A11, A8, A6
- **Narrative:** An attacker reads `INFISICAL_TOKEN`, a file, or D16 and presents `st.{id}.{secret}`. P4 accepts until delete or `expiresAt`. P15 returns plaintext via the project bot without the fourth segment (`secret-service.ts:725-754`; `secret-router.ts:59`).
- **Impact:** Secret read or write for the token's scopes from any address, for the remaining lifetime, which may be unbounded.
- **Mitigations:** M8, M14
- **Strength:** M14 partial — proves possession of the secret and stops a past `expiresAt`; M8 conditional — operator must delete or have set `expiresIn`

### T18 — Cluster Secrets hold the token and plaintext

- **Target:** P19, D14, D16, B3
- **STRIDE:** Information disclosure
- **Exploits:** A13, A11
- **Narrative:** The operator reads `infisicalToken` from a kube Secret (`infisicalsecret_helper.go:72-100`) and writes the plaintext map to the managed Secret (`infisicalsecret_helper.go:180-195`). Anyone with get on those objects has T14 and T17 material.
- **Impact:** Token theft and secret disclosure to any principal the cluster RBAC allows.
- **Mitigations:** none
- **Strength:** —

### T19 — Operator-chosen glob covers a tree

- **Target:** P7, P9
- **STRIDE:** Elevation of privilege
- **Exploits:** A1
- **Narrative:** Create accepts any `secretPath` string (`service-token-router.ts:77-83`). P7 matches with `$glob` (`project-permission.ts:244-264`; `lib/casl/index.ts:15-18`). A path of `/**` or a scope per environment grants that tree. The UI defaults to `/` in one environment (`AddServiceTokenModal.tsx:92-97`), which picomatch treats as that folder, not the tree.
- **Impact:** One leaked token exposes every path the glob matches.
- **Mitigations:** M12
- **Strength:** M12 conditional — only if the stored scopes name a narrow path

### New mitigations

#### M11 — `projectId` equality on service-token permission

- **Targets:** T1 full
- **Upholds:** A4
- **Evidence:** `backend/src/ee/services/permission/permission-service.ts:182-185`
- **Notes:** Enforced by default on every `getProjectPermission(SERVICE, …)` call, including `getAProject` (`project-service.ts:314-316`).

#### M12 — Scope CASL with `$glob`

- **Targets:** T16 partial; T19 conditional
- **Upholds:** A1
- **Evidence:** `backend/src/ee/services/permission/project-permission.ts:233-268`; `backend/src/services/secret/secret-service.ts:406-410`; `backend/src/lib/casl/index.ts:15-18`
- **Notes:** Enforced by default. Breadth is the stored glob.

#### M13 — `Read` on `service-tokens` for list

- **Targets:** T10 full
- **Upholds:** A14
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:123-124`
- **Notes:** Enforced by default. Viewer includes Read.

#### M14 — bcrypt compare and `expiresAt` delete

- **Targets:** T17 partial
- **Upholds:** A7, A8
- **Evidence:** `backend/src/services/service-token/service-token-service.ts:130-145`
- **Notes:** Enforced by default on every `st.` request that finds a row. Does not bound a token with `expiresAt` null.

#### M15 — Global rate limit on `realIp`

- **Targets:** T15 partial
- **Upholds:** A6
- **Evidence:** `backend/src/server/app.ts:71`; `backend/src/server/config/rateLimiter.ts:12-18`
- **Notes:** 600 requests / 60s. Key is P1's address. Not specific to `st.` routes.

### New assumptions

A11, A12, A13, A14, A15. Statements are in `securityAssumptions-v0.47.0-postgres.md`. A11 is the current credential form. A12 is the import exemption in `secret-service.ts:421-422`. A13 is the operator stores. A14 is the membership gate that T10 lacked. A15 is PostHog / Redis telemetry (`telemetry-service.ts:62-89`).

## 6. Reconcile

### Threats

| ID | Label | Reason | Current strength |
|---|---|---|---|
| T1 | confirmed | Attack fails; M11 full | M11 full |
| T2 | obsolete | Tokens have scopes, not built-in roles | — |
| T3 | obsolete | Refresh grant gone | — |
| T4 | obsolete | Agent sink gone | — |
| T5 | obsolete | No ST allowlist to bypass | — |
| T6 | regressed | Still true; M7 gone | none |
| T7 | confirmed | Actor is still one token | none |
| T8 | confirmed | Insert still skipped at retention 0 | M9 conditional |
| T9 | obsolete | Unauthenticated refresh gone | — |
| T10 | confirmed | Now M13 full | M13 full |
| T11 | obsolete | No admin role on the token; T1 closed | — |
| T12 | obsolete | JSON key package gone; see T14 | — |
| T13 | confirmed | Create and delete still gated | M4 full, M5 full |
| T14 | new | Four-part token wraps the workspace key | none |
| T15 | new | bcrypt on the request path | M15 partial |
| T16 | new | SERVICE skips per-import CASL | M12 partial |
| T17 | new | Durable three-part presented credential | M14 partial; M8 conditional |
| T18 | new | K8s Secrets hold token and plaintext | none |
| T19 | new | Stored glob can cover a tree | M12 conditional |

### Mitigations

| ID | Label | Reason | Current strength |
|---|---|---|---|
| M1 | obsolete | JWT service-token path gone | — |
| M2 | obsolete | `isActive` / `tokenVersion` gone | — |
| M3 | obsolete | Access JWT gone | — |
| M4 | confirmed | Create and delete still throwUnlessCan | T13 full |
| M5 | confirmed | Membership check moved, same effect | T13 full |
| M6 | obsolete | Token does not use project roles | — |
| M7 | obsolete | No per-token `trustedIps` | — |
| M8 | confirmed | Delete and optional `expiresAt` remain; weaker set | T17 conditional |
| M9 | confirmed | CRUD plus secret events; still retention-gated | T8 conditional |
| M10 | obsolete | Rotation gone | — |
| M11 | new | `projectId` equality | T1 full |
| M12 | new | Scope `$glob` CASL | T16 partial; T19 conditional |
| M13 | new | List requires Read | T10 full |
| M14 | new | bcrypt + `expiresAt` | T17 partial |
| M15 | new | Global 600/min on `realIp` | T15 partial |

### Assumptions

| ID | Label | Reason | Current class |
|---|---|---|---|
| A1 | regressed | Statement false; restated as scope binding | Upheld; threat-enabling if glob is broad |
| A2 | obsolete | Access/refresh lifetime gone | — |
| A3 | obsolete | Ephemeral access JWT gone | — |
| A4 | regressed | Statement false; secret auth is now project-bound | Upheld by code |
| A5 | confirmed | Still no org path | Upheld by code |
| A6 | regressed | Allowlist clause false | Threat-enabling |
| A7 | regressed | JWT/`isActive`/`tokenVersion` false | Upheld by code (new mechanism) |
| A8 | regressed | Generational revocation gone | Environmental |
| A9 | confirmed | Actor and retention unchanged; secret use now logged | Threat-enabling; environmental |
| A10 | regressed | Adaptive hash is on the path | Upheld; threat-enabling for cost |
| A11 | new | Durable three-part + client fourth segment | Threat-enabling |
| A12 | new | Import CASL skipped for SERVICE | Threat-enabling |
| A13 | new | Operator cluster rest | Environmental |
| A14 | new | Admin actions require membership | Upheld by code |
| A15 | new | Cloud telemetry leaves the deployment | Environmental |

## 7. Traceability

| T# | M# | A# | Strength |
|---|---|---|---|
| T1 | M11 | A4 | full |
| T6 | — | — | — |
| T7 | — | — | — |
| T8 | M9 | A9 | conditional |
| T10 | M13 | A14 | full |
| T13 | M4 | A5 | full |
| T13 | M4 | A14 | full |
| T13 | M5 | A5 | full |
| T13 | M5 | A14 | full |
| T14 | — | — | — |
| T15 | M15 | A6 | partial |
| T16 | M12 | A1 | partial |
| T16 | M12 | A12 | partial |
| T17 | M14 | A7 | partial |
| T17 | M14 | A8 | partial |
| T17 | M8 | A8 | conditional |
| T18 | — | — | — |
| T19 | M12 | A1 | conditional |

T6, T7, T14, T18 have no row. Obsolete IDs do not appear.

## 8. Residual risk

| Threat | Coverage | What closes it |
|---|---|---|
| T6 | none | A check of client IP against a token or project allowlist on P4. Configuration of identity-UA trusted IPs does not substitute. |
| T7 | none | One token per workload. Events name `serviceId`, not a workload. |
| T8 | M9 conditional | A plan with `auditLogsRetentionDays > 0`. |
| T14 | none | Treat D3 and D16 as secrets. No server-side control after F25. |
| T15 | M15 partial | Rate-limit `st.` authentication on an address the proxy assigns. Lower `SALT_ROUNDS` does not close theft; it only changes cost. |
| T16 | M12 partial | Apply the same per-import CASL to `ActorType.SERVICE` as to other actors (`secret-service.ts:421-422`). |
| T17 | M14 partial; M8 conditional | Set `expiresIn`. Delete on theft. Restrict D3. |
| T18 | none | Restrict get/list on the token Secret and the managed Secret. The application will not. |
| T19 | M12 conditional | Store narrow paths. Do not issue `/**` or one scope per environment unless intended. |

T1 and T10 and T13 are closed on the current pin.

## 9. Escalated findings

1. **A1, A4, A6, A7, A8, A10 fail** as written. A1 and A4 fail because the gaps closed. A6, A7, A8, A10 fail because the mechanism changed in a security-relevant way. Restatements are in `securityAssumptions-v0.47.0-postgres.md`.
2. **T6 regressed.** M7 is gone. Possession of the three-part token is access from any address.
3. **T16 new, unmitigated beyond sitting on an allowed folder.** SERVICE skips per-import CASL.
4. **T15 new.** bcrypt cost 10 on every successful row load. M15 is a generic 600/min cap on a spoofable IP.
5. **T17 new.** The presented credential is durable. M3's TTL bound is gone.
6. **T14 and T18 new, unmitigated.** The four-part string and the operator's cluster Secrets are the workspace-key package.
7. **T19 new.** Scope glob is an operator choice. Default UI is one environment at `/`.
8. **Strength drop:** T6 conditional → none. M8's recorded target T3 is obsolete; remaining revocation is delete and optional `expiresAt` only.

## 10. Input defects

- Baseline assumptions A2 and A3 described a two-tiered JWT design that the current DFD does not contain. They were feature-tied to v3 JWT tokens, not to "service token" as a durable `st.` secret.
- `securityAssumptions.md` has no row for list authorization. T10 had nothing to bind to. A14 is added for the current pin.
- `securityAssumptions.md` has no row for D10 / AUTH_SECRET compromise. Current D15 also holds `SALT_ROUNDS` and `ENCRYPTION_KEY`. Still unmodeled.
- Current DFD F69–F75 sit under "Lifecycle end" while they are observability. Correspondence is by function, not by the author's section title.
- Current DFD open question: `GET /api/v2/workspace/:id/encrypted-key` registers `verifyAuth` as `onResponse` (`v2/project-router.ts:44`). This report does not treat that as a service-token threat; the caller is a user JWT.
- Current DFD notes `split(".", 3)` would put `secret.fourth` into `TOKEN_SECRET` if a client sent the four-part string. First-party CLI and operator strip to three parts (`secrets.go:21-27`). Not mapped as a threat.
- Baseline DFD P7 missing P3 is closed at the current pin (M13). The defect does not carry forward.
- A5's "ceiling is project admin" is stronger now (scopes only). The no-org-escalation claim is what was confirmed.
