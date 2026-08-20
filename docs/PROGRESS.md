# Learning Progress

## Current state

- Current milestone: Milestone 0 — first-principles design
- Current task: Write the problem statement and architecture decision record described in the learning path.
- Last demonstrated evidence: None yet
- Open question: Document and defend the Python trade-offs in the Milestone 0 architecture decision record.

## Milestone gates

- [ ] Milestone 0 — first-principles design
- [ ] Milestone 1 — one observable process
- [ ] Milestone 2 — durable processing with Temporal
- [ ] Milestone 3 — fleet telemetry and cardinality
- [ ] Milestone 4 — advanced PromQL
- [ ] Milestone 5 — operational Grafana dashboards
- [ ] Milestone 6 — SLIs, SLOs, and recording rules
- [ ] Milestone 7 — alerting, routing, and runbooks
- [ ] Milestone 8 — Kubernetes foundations and deployment
- [ ] Milestone 9 — Kubernetes-native Temporal and full-stack monitoring
- [ ] Milestone 10 — fleet decoration, correlation, and health checks
- [ ] Milestone 11 — Prometheus operations and scale-out design
- [ ] Milestone 12 — game days and capstone defense

## Evidence log

Add one entry only after a demonstration.

| Date | Milestone | Evidence demonstrated | Remaining risk or misconception |
| --- | --- | --- | --- |
| | | | |

## Decision log

| Date | Decision | Reason | Revisit when |
| --- | --- | --- | --- |
| 2026-08-20 | Use Python for the API and Temporal Worker. | The learner chose Python and demonstrated a Python 3.13 project managed by uv; the Milestone 0 ADR still needs to defend the trade-off. | If performance, ecosystem, or operational evidence challenges the choice. |
| 2026-08-20 | Introduce each automation gate when its corresponding project risk first appears. | Keeps CI proportional to the system while requiring a local command, controlled failure, recovery evidence, least privilege, and reproducible dependencies before enforcement. | At every milestone review and whenever a new artifact, dependency, execution model, or deployment boundary is introduced. |
