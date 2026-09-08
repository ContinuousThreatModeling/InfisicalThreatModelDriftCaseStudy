# Threat model drift — case study

Threat models are point-in-time. Draw a DFD, run a workshop, enumerate threats, and the model starts rotting on the next PR. That is truer with rapid AI-powered development and shrinking SSDLCs, which is why many organisations have quietly given up on “shift left.”

This case study is one run of a **lifecycle layer** that keeps a threat model alive from change events instead of regenerating it from scratch. A security engineer plus a seed agent produces the initial DFD, threats, assumptions, and mitigations. On a later change, the same record is tested: the DFD is updated from code, assumptions are revalidated, and only then are new threats identified. Each update is classified so a human can triage. Low-confidence or high-risk items are meant for the security team; the rest can flow through.

The point is the **harness**, not a clever one-shot agent. Company context here is the threat model of record plus the feature’s assumption set. Past dispositions are the stable `T#` / `M#` / `A#` IDs: confirmed items stay, obsolete classes drop out of the current set, dismissed surfaces are not re-invented as a green-field STRIDE pass. Flags (confirmed, regressed, new, obsolete) are how the organisation sees posture change against its own record.

The operational vision is a watcher on PRs. This repository demonstrates the same lifecycle on a real Infisical change event: two public tags of **service tokens**. The prompts are tag-agnostic and reusable. The pins exist so a reader can reproduce the artifacts from code.


| Pin          | Tag                                         | Role in the lifecycle  |
| ------------ | ------------------------------------------- | ---------------------- |
| Seed         | `infisical/v0.42.0` (`735cf093f0`)          | Threat model of record |
| Change event | `infisical/v0.47.0-postgres` (`041535bb47`) | Drift under test       |


Between those pins the credential is not the same object. The seed is a JWT access/refresh pair, authorized by a project role, stored in MongoDB, with an agent refresh loop. The later pin is a durable `st.{id}.{secret}`, authorized by stored scopes, stored in Postgres, with a Kubernetes operator path. Regenerating a threat model at the later pin would hide that. The lifecycle has to ask what happened to the record.

## Lifecycle

![Continuous threat modelling](threat-model-drift-v4.jpg)

Editable source: [threat-model-drift-v4.drawio](threat-model-drift-v4.drawio).

A change event triggers analysis. **Discover** derives the current architecture. **Compare** measures it against the approved baseline. **Validate** tests the existing model (are recorded threats still mitigated, controls still present, assumptions still true). **Identify** names what the change introduces. **Reconcile** classifies every threat, control, and assumption as **confirmed · regressed · new · obsolete**. Escalated findings go to security review. Remediation returns to the author. An approved change updates the threat model of record. Blue is delivery; white is automated analysis; amber is human decision; green is approved state.

1. **Seed.** Engineer + agent. DFD from code. Assumptions tied to the feature. Threats and mitigations bound to both. A mitigation is only admissible if it names a threat it reduces and an assumption it upholds.
2. **Change event.** Here, a tag-to-tag evolution. In production, a PR that moves design, code, or infrastructure.
3. **Discover / Compare.** Current architecture from the later DFD; architectural delta against the seed DFD. Independently generated DFDs do not share IDs; mapping is by function.
4. **Validate.** Test the *existing* model. Are recorded threats still mitigated? Are controls still in code? Do documented assumptions still hold?
5. **Identify.** What the change *introduces*: new threats, controls, assumptions. STRIDE is a coverage checklist, not a reason to rebuild the list.
6. **Reconcile.** Every ID labeled **confirmed · regressed · new · obsolete**. Escalated findings go to a human. The report does not silently replace the threat model of record.
7. **Mature the assumptions.** Confirmed IDs are kept. Failed statements are restated to what is true now. Obsolete JWT-lifetime claims drop from the current set. The revised file is the seed for the next event.



## Architecture drift (this change event)

![Service token architecture drift](serviceTokenDfd-consolidated-v0.42.0-v0.47.0-postgres.jpg)

Editable source: [serviceTokenDfd-consolidated-v0.42.0-v0.47.0-postgres.drawio](serviceTokenDfd-consolidated-v0.42.0-v0.47.0-postgres.drawio).

Left is the seed (refresh JWT + access JWT, MongoDB, agent). Right is the change event (`st.{id}.{secret}`, Postgres, operator). Red is gone, amber is changed, green is new. The wrap, credential at rest, auth path, secret authorization, stores, and observability all moved. That is the Compare input the lifecycle tested; it is not itself a threat list.

## How to read this run

1. Lifecycle diagram above, then the side-by-side DFD.
2. Seed: [serviceTokenDfd-v0.42.0.md](serviceTokenDfd-v0.42.0.md), [securityAssumptions-0.42.0.md](securityAssumptions-0.42.0.md), [serviceTokenThreatModel-v0.42.0.md](serviceTokenThreatModel-v0.42.0.md).
3. Change-event DFD: [serviceTokenDfd-v0.47.0-postgres.md](serviceTokenDfd-v0.47.0-postgres.md).
4. Lifecycle output: [serviceTokenThreatModelDrift-v0.42.0-to-v0.47.0-postgres.md](serviceTokenThreatModelDrift-v0.42.0-to-v0.47.0-postgres.md) and [securityAssumptions-v0.47.0-postgres.md](securityAssumptions-v0.47.0-postgres.md).

**[readingTheDriftReport.md](readingTheDriftReport.md)** explains what those outputs are, which files are inputs versus results, and how to read the drift report section by section. Read it before the report itself.



## Prompts (the harness)

Placeholders only. No tag or feature is baked in. Run them on another feature or another change event.


| Prompt                                                           | Layer                           | Produces                                                                             |
| ---------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------ |
| [dfdAgentPrompt.md](dfdAgentPrompt.md)                           | Seed (and DFD update on change) | One descriptive Markdown DFD at a pin. No threats.                                   |
| [threatModelAgentPrompt.md](threatModelAgentPrompt.md)           | Seed                            | Threats and mitigations bound to that DFD and the assumption set.                    |
| [threatModelDriftAgentPrompt.md](threatModelDriftAgentPrompt.md) | Lifecycle                       | Drift report + revised assumptions. Validate first, identify second, reconcile last. |


Every claim about current behavior needs a `path:line` at the change-event pin. Absence of a formerly recorded control is a regression, not a missing bullet in a new model.

## What this change event did to the record

Closed: cross-project secret access (T1), inventory without a permission check (T10), and threats whose surfaces are gone (refresh grant, agent sink, built-in roles on the token).

Regressed: per-token IP allowlisting is gone, so possession of the three-part token is access from any address (T6). Seed assumptions about origin, JWT auth, generational revoke, and HMAC cost no longer hold as written.

New residual, for triage: durable presented credential (T17), bcrypt on every `st.` request (T15), SERVICE actors skipping per-import CASL (T16), four-part token as a workspace-key package including in cluster Secrets (T14, T18).

The drift report’s **Escalated findings** section is the security-team queue for this event. This case study does not auto-merge the record.

## Reproduce

```bash
git checkout infisical/v0.42.0            # seed DFD + baseline TM
git checkout infisical/v0.47.0-postgres   # change-event DFD + drift
```

JPGs render in this README. Open the matching `.drawio` files in [diagrams.net](https://app.diagrams.net/) or a draw.io editor extension to edit. Checkout only to read code at a pin.

This run is not a PR bot, a confidence scorer, or an org-wide disposition store. Those belong to a deployment of the same lifecycle. What is in this repo is the seed, one change event, the harness prompts, and a matured assumption set ready for the next event.
