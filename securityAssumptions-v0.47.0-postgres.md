# Service token security assumptions

Scope: the service token as implemented at tag `infisical/v0.47.0-postgres`.

- **A1 Environment reach** — A service token's secret reach is exactly its stored `scopes` (`environment` × `secretPath`) and `permissions` (`read` / `write`). `secretPath` is matched with picomatch `$glob`. The default UI scope is one environment at `/`. *Upheld by code. Threat-enabling when a scope is a broad glob or lists many environments.*

- **A4 Project binding** — A service token is bound to a single project for administration, token-detail delivery, and secret authorization. Secret authorization rejects a request whose `projectId` is not the token's `projectId`. *Upheld by code.*

- **A5 No org escalation** — A service token has no organization permission path. `getOrgPermission` has no `SERVICE` case. The token's CASL is secret actions on its scopes only. *Upheld by code.*

- **A6 Open network origin** — Network origin is unrestricted. There is no per-token IP allowlist, and project `trusted_ips` are not consulted on the service-token request path. The address written to audit is whatever P1 records from the first matching forwarded header. *Threat-enabling.*

- **A7 Hash-compared authentication** — Authentication is a prefix check (`st.`), a row load by the second segment, an `expiresAt` check that deletes the row when past, `bcrypt.compare` of the third segment to `secretHash`, and a `lastUsed` write. There is no `isActive` flag and no `tokenVersion`. *Upheld by code.*

- **A8 Operator-governed revocation** — Revocation is immediate delete or optional `expiresAt`. There is no generational revocation. `expiresIn` is nullable and is not applied by policy. The UI default is one day; `Never` is offered. *Environmental.*

- **A9 Per-token attribution** — Audit actor is the token (`serviceId`, `name`) plus IP and user agent. Token CRUD and secret read/write emit events. Distinct workloads sharing one token are indistinguishable by actor. Insert is skipped when `auditLogsRetentionDays` is 0. *Threat-enabling for shared tokens. Environmental for durability.*

- **A10 Adaptive-hash authentication** — Authentication is `bcrypt.compare` with `SALT_ROUNDS` default 10 on every service-token request that finds a row, plus a `lastUsed` write on success. Per-request denial-of-service amplification is high relative to an HMAC verify. *Upheld by code. Threat-enabling for cost.*

- **A11 Durable presented credential** — The credential presented on each request is `st.{id}.{secret}`. It is durable until delete or optional `expiresAt`. The fourth segment is a client-only AES-256-GCM key for the workspace-key ciphertext and is not sent to the API. The three presented segments are sufficient for raw secret delivery via the project bot. *Threat-enabling.*

- **A12 Import scope exemption** — When `include_imports` is set, a `SERVICE` actor is given every import on an authorized folder. Per-import CASL is skipped. *Threat-enabling.*

- **A13 Cluster secret rest** — The Kubernetes operator stores the four-part token in a cluster Secret (`infisicalToken`) and writes plaintext secret maps to a managed Secret. *Environmental.*

- **A14 Membership-gated administration** — Create, list, and delete of service tokens require a project membership whose role includes the matching `service-tokens` action. Create also requires `Create` on `Secrets` for each requested scope. There is no update route. *Upheld by code.*

- **A15 Third-party telemetry** — On cloud with `TELEMETRY_ENABLED`, secret operations are sent to PostHog with actor metadata. Self-host increments Redis counters. *Environmental.*
