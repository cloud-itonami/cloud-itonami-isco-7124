# cloud-itonami-isco-7124

Open Occupation Blueprint for **ISCO-08 7124**: Insulation Workers.

This repository designs a forkable OSS business for an insulation-crew job-site scheduling and logistics coordination practice: a job-site scheduling and supply-coordination robot manages crew/task records under a governor-gated actor, so an insulation crew keeps its own operating records instead of renting a closed workforce-management SaaS.

**Maturity: `:implemented`.** `src/insulationcrew/` implements the
`InsulationCrewActor` as a `langgraph.graph/state-graph`
(`insulationcrew.actor`) wired to an `Insulation Crew Advisor`
(`insulationcrew.advisor`) and an independent `InsulationCrewGovernor`
(`insulationcrew.governor`), following the itonami actor pattern
(ADR-2607121000): `:intake -> :advise -> :govern -> :decide -+-> :commit
(:ok?) +-> :request-approval (:escalate?, human-in-the-loop interrupt)
+-> :hold (:hard?)`. 23 tests / 50 assertions green (`clojure -M:test`).
HARD invariants (always hold, never overridable): worker provenance,
site provenance, no-actuation (`:effect` must be `:propose`), a closed
op-allowlist (`:log-work-record`, `:schedule-crew-operation`,
`:flag-safety-concern`, `:coordinate-supply-order` — nothing else may
ever be proposed), and a permanent, unconditional block on any
proposal that would directly finalize an insulation-installation-execution
decision or override a site safety officer's judgment. Always-escalate
paths (human sign-off regardless of confidence, mapping this repo's
Trust Controls in [`docs/business-model.md`](docs/business-model.md)):
`:flag-safety-concern` (always) and `:coordinate-supply-order` above
the registered cost threshold.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a job-site scheduling/logistics coordination robot performs crew scheduling, task/materials-usage/progress-record logging and insulation-materials supply-order coordination for an insulation crew, under an actor that proposes actions and an independent **Insulation Crew Governor** that gates them. The governor never
dispatches hardware itself, never performs insulation-installation work on the job site, and never finalizes an insulation-installation-execution decision or overrides a site safety officer's judgment; `:high`/`:safety-critical` actions (such as a flagged particulate-exposure/confined-space/ventilation concern, or an above-threshold supply order) require human sign-off. **This actor coordinates job-site scheduling/logistics only — it never performs insulation-installation work itself.**

## Core Contract

```text
crew roster + job-site registration + safety-reporting policy
        |
        v
Insulation Crew Advisor -> Insulation Crew Governor -> log/schedule/coordinate, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, finalize
an insulation-installation-execution decision, override a site safety
officer's judgment, suppress an operating record, or disclose sensitive data
without governor approval and audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `7124`). Required capabilities:

- :robotics
- :identity
- :audit-ledger

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## License

AGPL-3.0-or-later.
