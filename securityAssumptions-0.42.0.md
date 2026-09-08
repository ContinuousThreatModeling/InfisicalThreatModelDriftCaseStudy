# Service token security assumptions

Scope: the service token as implemented at tag `infisical/v0.42.0`.

- **Environment reach** — A service token has no intrinsic environment binding. Its environment reach is exactly what its assigned project role grants, which for all three built-in roles is every environment in the project.

- **Two-tiered lifetime** — Credential lifetime is two-tiered: a short-lived access token bounded by `accessTokenTTL`, minted from a durable refresh credential optionally bounded by `expiresAt`. The credential presented on each request is always time-boxed.

- **Ephemeral presented credential** — Only the refresh token requires durable storage. The access token a workload presents is ephemeral and re-minted in memory, so the long-lived credential touches disk once, at bootstrap.

- **Partial project binding** — A service token is bound to a single project for token administration and project-key delivery, but **not** for secret access. Secret authorization is evaluated against the project named in the request rather than the project the token was issued for.

- **No org escalation** — Vertical privilege escalation to organization-level API control is impossible. The ceiling is project admin, which carries authority over project membership and roles.

- **Open network origin** — Network origin is unrestricted by default (`0.0.0.0/0`) and is enforceable per token, but enforcement is only as trustworthy as the deployment's determination of the client IP address.

- **Stateful authentication** — Authentication is stateful, not a hash comparison: signature verification followed by server-side checks of `isActive`, `expiresAt`, and `tokenVersion`, with a database write on every authenticated request.

- **Operator-governed revocation** — Revocation is available in three forms — immediate, generational, and time-based — and none of them is applied by policy, so credential lifetime governance rests entirely with the operator.

- **Per-token attribution** — Audit attribution is per token, enriched with source IP and user agent. Distinct workloads sharing one token are indistinguishable by actor, and the durability of the trail depends on the retention entitlement.

- **Low-cost authentication** — Authentication is low-cost: HMAC signature verification and indexed lookups, with no adaptive password hashing on the request path, so per-request denial-of-service amplification is minimal.
