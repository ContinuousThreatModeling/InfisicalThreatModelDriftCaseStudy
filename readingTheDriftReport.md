# Reading the output

What one lifecycle run produced, and how to read it. Start at [README.md](README.md) for why the lifecycle exists; this file is about the artifacts on disk.

## What the run produced

Seven Markdown files sit in this repo, but they are not peers. Three are the **seed** — the threat model of record as it stood at `infisical/v0.42.0`. One describes the architecture at the change event. Two are the **output** of the run: what the lifecycle concluded when it tested the seed against that change. One is the harness that produced them.

| File | Pin | Role |
| ---- | --- | ---- |
| [serviceTokenDfd-v0.42.0.md](serviceTokenDfd-v0.42.0.md) | seed | Baseline DFD. Entities, processes, stores, boundaries, flows. No threats. |
| [securityAssumptions-0.42.0.md](securityAssumptions-0.42.0.md) | seed | Ten assumptions the baseline model rests on. Unnumbered bullets. |
| [serviceTokenThreatModel-v0.42.0.md](serviceTokenThreatModel-v0.42.0.md) | seed | Threats `T1`–`T13` and mitigations `M1`–`M10`, bound to that DFD and those assumptions. |
| [serviceTokenDfd-v0.47.0-postgres.md](serviceTokenDfd-v0.47.0-postgres.md) | change event | Current DFD, generated independently at the later pin. Also no threats. |
| **[serviceTokenThreatModelDrift-v0.42.0-to-v0.47.0-postgres.md](serviceTokenThreatModelDrift-v0.42.0-to-v0.47.0-postgres.md)** | **output** | **The drift report. Every recorded ID tested, every new one identified, all of them reconciled.** |
| **[securityAssumptions-v0.47.0-postgres.md](securityAssumptions-v0.47.0-postgres.md)** | **output** | **The matured assumption set: thirteen statements, restated to what is true now. Seed for the next event.** |
| [threatModelDriftAgentPrompt.md](threatModelDriftAgentPrompt.md) | harness | The tag-agnostic contract that produced both outputs. |

Two diagrams support the same story. [threat-model-drift-v4.jpg](threat-model-drift-v4.jpg) is the lifecycle itself; the side-by-side [serviceTokenDfd-consolidated-v0.42.0-v0.47.0-postgres.jpg](serviceTokenDfd-consolidated-v0.42.0-v0.47.0-postgres.jpg) is the architectural delta the run consumed. Both have `.drawio` sources beside them.

The distinction that matters: **the seed files were not edited by this run.** A lifecycle layer does not rewrite history when a threat goes obsolete. `securityAssumptions-0.42.0.md` still claims a two-tiered JWT lifetime, which is false at the later pin, and that is correct — it is the record as it stood. The drift report says so, and `securityAssumptions-v0.47.0-postgres.md` carries the restatement. Diffing the two assumption files is the shortest view of what the change event did to the model.

## What the outputs say, in one paragraph each

The **drift report** is a verdict on the record. It closed `T1` and `T10` because the code now checks what it did not check before, dropped seven threats whose surfaces no longer exist, flagged `T6` as regressed because per-token IP allowlisting is gone, and named six new threats — `T14`–`T19` — that the change introduced. Its Escalated findings section is the queue for a human.

The **matured assumption set** is what the next run will test against. It has no `A2` or `A3`: those described the JWT lifetime design and lost their subject, so they drop out of the current set rather than being marked false forever. `A1`, `A4`, `A6`, `A7`, `A8` and `A10` survive as IDs but with rewritten statements, because the mechanism beneath them changed. `A11`–`A15` are new.

## Reading the drift report

The report's ten sections are ordered so the record is tested before it is extended, and the later sections are only meaningful because the earlier ones constrained them. Read in order.

| §  | Section           | What it answers                                                                                              |
| -- | ----------------- | ------------------------------------------------------------------------------------------------------------ |
| 1  | Scope             | Which pins, which inputs, what was excluded. Closes by stating the report does not replace the record.       |
| 2  | Discover          | Architecture at the change-event pin, plus tables mapping baseline element IDs to current ones.               |
| 3  | Compare           | Architectural delta as **unchanged / changed / added / removed** per element class, then material differences. |
| 4  | Validate          | The existing record, one recorded `T#` / `M#` / `A#` at a time. Nothing here is new.                          |
| 5  | Identify          | Only what the change introduced: `T14`–`T19`, `M11`–`M15`, `A11`–`A15`.                                       |
| 6  | Reconcile         | Every ID labeled **confirmed · regressed · new · obsolete**, with a reason and current strength.              |
| 7  | Traceability      | Surviving `T#` → `M#` → `A#` triples. A threat with no row has no mitigation.                                 |
| 8  | Residual risk     | What is still open and what would close it.                                                                   |
| 9  | Escalated findings | The security-team queue for this event.                                                                      |
| 10 | Input defects     | Faults in the inputs themselves, not in the code.                                                             |

Section 4 is what makes this a lifecycle rather than a re-run. Each threat entry carries *Target then / now*, *Narrative still valid*, *Recorded mitigations*, *Strength then / now*, and `path:line` evidence; mitigations and assumptions end in a *Verdict*. A "no" on *Narrative still valid* is not a pass. `T1` is closed by a new check and `T2` is closed because its surface is gone — only §6 separates those as `confirmed` and `obsolete`. Likewise `A10` "fails" because bcrypt replaced an HMAC verify, which is a mechanism change, not a defect.

### Three vocabularies, read together

**Labels** (§6) are dispositions against the record: `confirmed` survived the test, `regressed` was true and is no longer covered, `new` arrived with the change, `obsolete` lost its surface. **Strength** grades a mitigation against a threat: `full`, `partial`, `conditional` (holds only under a configuration or operator choice), or `—`. **Assumption class** says how a statement behaves: *upheld by code*, *threat-enabling*, or *environmental*.

None of them reads alone. `T17` is `new` with `M14 partial; M8 conditional`, so possession of a durable token is bounded only by a bcrypt check and a nullable `expiresAt` — which is why §8 gives "set `expiresIn`" as what closes it. A `confirmed` threat is not a safe one: `T6` and `T7` are both still live with no mitigation at all.

### Resolving an ID across the repo

- **A recorded `T#` or `M#` in §4** — the original narrative is in [serviceTokenThreatModel-v0.42.0.md](serviceTokenThreatModel-v0.42.0.md) under the same ID and title. §4 gives the verdict, not the original text.
- **An element ID (`E#`, `P#`, `D#`, `F#`, `B#`)** — resolve it in the DFD *for that pin*: [serviceTokenDfd-v0.42.0.md](serviceTokenDfd-v0.42.0.md) or [serviceTokenDfd-v0.47.0-postgres.md](serviceTokenDfd-v0.47.0-postgres.md). **These do not correspond across pins.** The DFDs were generated independently, so baseline `P3` and current `P3` are unrelated; §2's tables are the mapping, and it is by function. This is the easiest way to misread the report.
- **An assumption** — the baseline file [securityAssumptions-0.42.0.md](securityAssumptions-0.42.0.md) has unnumbered bullets; the report numbers them `A1`–`A10` in file order, matching by name. Restated and new statements are in [securityAssumptions-v0.47.0-postgres.md](securityAssumptions-v0.47.0-postgres.md), which carries explicit IDs and has no `A2` or `A3` — obsolete assumptions drop out of the current set rather than being deleted from history.
- **Why a section exists at all** — [threatModelDriftAgentPrompt.md](threatModelDriftAgentPrompt.md) is the contract that produced the shape.

### What carries forward

`T#` / `M#` / `A#` are the stable identifiers across events; element IDs are local to one DFD. After this run, [securityAssumptions-v0.47.0-postgres.md](securityAssumptions-v0.47.0-postgres.md) plus the §6 tables are the seed for the next change event. §9 is the only section that asks a human for a decision.
