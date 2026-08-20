# Advanced Prometheus, Grafana, Kubernetes, and Temporal Capstone

## 1. Project outcome

Build and operate a small but realistic **accelerator-job control plane**. Clients submit processing jobs; Temporal durably coordinates validation, simulated device allocation, execution, health checks, cleanup, and status; Kubernetes runs the services; Prometheus gathers operational signals; Grafana supports investigation; and Alertmanager routes actionable alerts.

At completion, you should be able to design, implement, query, scale, and defend the observability system—not merely install it. Expect roughly 70-100 focused hours. Depth and demonstrated reasoning matter more than the calendar.

## 2. Why this matches the target role

The Fluidstack [Production Engineer, IaaS role](https://fluidstack.io/jobs/bff2c500-d321-4bde-b251-9f7a2ef2d894) describes a platform that makes a fleet legible from site to device and link, decorates and correlates telemetry, runs health checks, maintains fleet state, exposes stable APIs, operates on Kubernetes, and may use Prometheus, Thanos, VictoriaMetrics, Temporal/Cadence, Go, Python, and Postgres.

| Role-shaped capability | Capstone evidence |
| --- | --- |
| Site-to-device fleet visibility | Hierarchical simulated inventory, reusable dashboards, and drill-down links |
| Telemetry pipelines and correlation | Discovery metadata, info metrics, vector matching, recording rules, and stale-data handling |
| Health-check framework | Versioned checks with freshness, duration, result, and failure-reason signals |
| Production control plane | Versioned job/status API backed by durable Temporal Workflows |
| Source of truth | Explicit ownership among inventory, Temporal history, application state, and Prometheus aggregates |
| Kubernetes operations | Reproducible local cluster, probes, resources, discovery, rollouts, and failure exercises |
| Time-series scale | Cardinality budgets, TSDB self-monitoring, HA/remote-write design, and a Thanos/VictoriaMetrics comparison |
| Pager ownership | SLO alerts, routing/inhibition, runbooks, game days, and postmortems |

## 3. System boundaries and data model

The first implementation can be one API process and one Worker process. It evolves into these logical components:

| Component | Responsibility | Important failure modes |
| --- | --- | --- |
| Job API | Validate requests, start/query/cancel Workflows, expose versioned status | rejection, timeout, Temporal unavailable, incompatible API change |
| Temporal Workflow | Coordinate the durable job lifecycle | nondeterminism, history growth, bad timeout/retry policy |
| Activities/Workers | Allocate, process, health-check, and release simulated devices | crash, retry storm, heartbeat loss, task-queue saturation |
| Fleet inventory | Define site/rack/node/device/link identity and intended lifecycle state | stale metadata, duplicate identity, actual/desired mismatch |
| Fleet simulator/exporter | Emit bounded hardware and link telemetry | missing target, stale gauge, counter reset, label explosion |
| Load/fault controller | Produce traffic and controlled failures | unrealistic load, unbounded IDs, hidden test conditions |
| Prometheus/rules | Scrape, store, query, and precompute signals | scrape failure, slow query, disk pressure, rule failure, remote-write lag |
| Grafana | Support overview and drill-down investigation | misleading units, hidden aggregation, expensive query, dashboard drift |
| Alertmanager | Group, route, inhibit, silence, and deliver alerts | notification loss, alert storm, bad grouping, overbroad silence |

Use at least two sites, two racks per site, several nodes per rack, multiple devices per node, and links between devices or nodes. The inventory is intentionally bounded so that device identity can be analyzed as a legitimate fleet dimension. Request, user, Workflow, run, and job IDs remain unbounded and must not become Prometheus labels.

## 4. Learning method and milestone gate

Each milestone follows the same loop:

1. **Predict:** Write what you expect before changing code or configuration.
2. **Build:** Implement the smallest learner-owned change that tests the idea.
3. **Observe:** Capture target state, metric output, query result, dashboard behavior, or Workflow history.
4. **Break:** Inject one controlled failure.
5. **Explain:** Teach the mechanism back without reading the configuration.
6. **Record:** Add evidence and one remaining risk to `docs/PROGRESS.md`.

A milestone is complete only when its acceptance evidence is demonstrated. A screenshot alone is weak evidence; pair it with the query, target state, test, or event history that explains it.

## 5. Milestone map

| Milestone | Focus | Approximate effort | Demonstration |
| --- | --- | ---: | --- |
| 0 | First-principles design | 2-4 h | Defend system boundaries and signal ownership |
| 1 | One observable process | 5-7 h | Scrape useful application metrics and explain every series |
| 2 | Temporal durable processing | 7-10 h | Survive Worker loss without corrupting a job |
| 3 | Fleet telemetry/cardinality | 5-8 h | Quantify and remove a deliberate cardinality failure |
| 4 | Advanced PromQL | 8-12 h | Solve an investigation set and predict result labels/types |
| 5 | Grafana operations views | 6-9 h | Drill from fleet symptom to device/workflow cause |
| 6 | SLIs/SLOs/rules | 6-9 h | Defend error budgets and tested recording rules |
| 7 | Alerting/runbooks | 5-8 h | Route one actionable incident without an alert storm |
| 8 | Kubernetes foundations | 7-10 h | Deploy and discover the app in `kind` |
| 9 | Temporal on Kubernetes | 8-12 h | Observe app, Temporal, and cluster through one stack |
| 10 | Decoration/correlation | 6-9 h | Explain desired/actual/observed state mismatch |
| 11 | Prometheus operations/scale | 6-10 h | Defend an HA and long-term-storage architecture |
| 12 | Game days/capstone | 8-12 h | Run incident, postmortem, and architecture interview |

## 6. Milestone 0 — First-principles design

A **signal contract** states what an operator needs to know, which component owns the truth, how fresh it must be, and how failure appears. Begin with the operating questions, not metric names.

### Learner work

Create a short design document containing:

- the user journey from job submission to durable completion;
- an architecture diagram and trust boundaries;
- desired state, actual state, and observed state for a simulated device;
- three user-visible failure symptoms and five internal causes;
- what belongs in an API/database, Temporal history, logs/traces, and Prometheus;
- an initial label taxonomy for service, site, rack, node, device, link, route, status class, operation, and failure class;
- an architecture decision record for implementation language. Go is especially relevant to the role and ecosystem; Python is also role-relevant and faster for some learners. Choose based on what you want to practice and defend the trade-off.

### Acceptance gate

- Explain why Prometheus's pull model changes failure detection.
- Explain why a job ID is useful operational context but a dangerous metric label.
- Explain why Kubernetes restart behavior does not provide durable job orchestration.
- Identify the source of truth for a Workflow, fleet inventory, current telemetry, and an SLO.
- Estimate the total duration and weekly time budget for the project.

### Interview drill

**Question:** A dashboard says a device is healthy, inventory says it is draining, and its health check has not run for ten minutes. Which fact wins?

A strong answer separates desired lifecycle state, last observed health, signal freshness, and the action allowed by the control plane. There is no universal winning row without a freshness and ownership contract.

## 7. Milestone 1 — One observable process

An **instrumented application** exposes signals about user-visible work and internal saturation at the event where each value changes. Instrumentation is application behavior, not a wrapper added after the fact.

### Learner work

Build a minimal job API with bounded in-process work, variable duration, controlled failures, and a metrics endpoint. Run one Prometheus instance locally and make it scrape both itself and the application.

Before writing metrics, create a metric design table with:

| Field | Required reasoning |
| --- | --- |
| Operator question | What decision does this metric support? |
| Metric type | Why counter, gauge, histogram, or info? |
| Name/unit | Why is the suffix and base unit correct? |
| Labels | Is every value bounded and useful for aggregation? |
| Series estimate | Product of possible label values plus histogram buckets |
| Update site | Which exact event changes the value? |
| Failure behavior | What happens on restart, scrape failure, or missing observation? |

Cover request rate, error rate, duration, in-flight work, queue depth, job outcomes, and process/build information. Do not use raw path parameters, exception messages, user IDs, or job IDs as labels.

### Required experiments

- Restart the process and observe counters, process start time, and query behavior.
- Make the metrics endpoint slower than the scrape timeout.
- Compare application success with scrape success; show a state where one is healthy and the other is not.
- Calculate expected series count and compare it with the exposed/ingested result.

### Acceptance gate

- Prometheus target state and the metrics endpoint agree with your explanation.
- Every custom metric has a documented operator question and cardinality bound.
- You can explain counter reset, gauge lifecycle, histogram sum/count/buckets, scrape interval, timeout, and staleness.
- You can identify at least one valuable fact that should remain a log field rather than a metric label.

Read the official guidance on [instrumentation](https://prometheus.io/docs/practices/instrumentation/), [metric naming](https://prometheus.io/docs/practices/naming/), and [histograms](https://prometheus.io/docs/practices/histograms/).

## 8. Milestone 2 — Durable processing with Temporal

A **Temporal Workflow** is durable orchestration logic whose event history lets execution resume after process failure. An **Activity** performs fallible or side-effecting work and must be safe under its configured retry behavior.

### Learner work

Replace the in-process job lifecycle with a Workflow that coordinates activities such as validation, allocation, processing, verification, and release. Keep the API responsible for starting, querying, signaling/cancelling, and reporting the Workflow through a versioned contract.

Design and defend:

- Workflow ID and duplicate-start behavior;
- Task Queue boundaries and Worker ownership;
- Activity start-to-close, schedule-to-start, and heartbeat timeouts;
- retryable versus non-retryable failure classes;
- idempotency keys and compensation/cleanup;
- cancellation and at least one Signal or Update;
- queryable status without exposing raw Temporal internals as the public API;
- Workflow history growth and a continue-as-new threshold;
- deterministic code, replay tests, and a future versioning strategy.

### Observability requirements

Separate three views:

1. user journey: accepted, completed, failed, and end-to-end latency;
2. Worker/task queue: pollers, available capacity, schedule-to-start delay, Activity attempts, and heartbeat failures;
3. Temporal service: frontend/history/matching/persistence health.

Do not add Workflow or job IDs as Prometheus labels. Explain whether each custom metric is emitted by API code, Workflow code, an Activity, a Worker interceptor, or derived from Temporal metrics, and why Workflow replay cannot double-count it.

### Failure experiments

- Terminate the Worker during a long Activity and predict retry/resumption behavior.
- Make an Activity fail before and after its external side effect.
- Remove all pollers from one Task Queue.
- Send cancellation during cleanup.

### Acceptance gate

- A job finishes correctly after Worker loss without duplicating a simulated external side effect.
- Event history, application metrics, and logs tell consistent but intentionally different stories.
- You can explain why Temporal and Kubernetes solve different failure layers.
- A replay or Workflow test guards determinism.

Start with the official [Temporal documentation](https://docs.temporal.io/) and [Temporal CLI](https://github.com/temporalio/cli).

## 9. Milestone 3 — Fleet telemetry and cardinality engineering

**Cardinality** is the number of distinct time series produced by metric names and label-value combinations. It is a capacity decision that affects memory, disk, network, query cost, and remote-write behavior.

### Learner work

Add the bounded site/rack/node/device/link inventory and simulated telemetry. Include utilization, temperature, memory, corrected/uncorrected error counters, link throughput/errors, health-check result/duration/freshness, and intended lifecycle state where semantically justified.

Create a cardinality budget that estimates:

- series per metric family;
- multiplier from classic histogram buckets;
- total series per node, rack, site, and full lab fleet;
- the projected cost at 10,000 nodes and multiple devices per node;
- churn caused by ephemeral Workers, Pods, or changing label values.

### Deliberate failure lab

Behind an explicit lab-only switch, temporarily attach an unbounded job or run identifier to one frequently updated metric. Generate traffic, measure series growth and churn, inspect Prometheus TSDB/head metrics, then remove the label and explain why old series do not disappear from storage immediately.

### Acceptance gate

- Every identity dimension is classified as bounded inventory, bounded category, or unbounded event identity.
- Series estimates are within an order of magnitude of measurement and discrepancies are explained.
- Missing target, stale telemetry, unhealthy device, and administratively drained device are distinguishable.
- You can defend when per-device metrics are necessary despite their cost.

## 10. Milestone 4 — Advanced PromQL

**PromQL** evaluates instant vectors, range vectors, scalars, and strings. Correct queries require reasoning about metric semantics, evaluation time, label sets, and aggregation—not only syntax.

For each exercise, write the expected expression type and output label set before executing it.

### Query lab

Write and explain queries for:

1. request throughput by route and status class;
2. fleet-wide and per-site error ratios, aggregating numerator and denominator correctly;
3. job queue wait and end-to-end latency at p50/p95/p99;
4. percentage of observations meeting an explicit latency threshold;
5. hottest devices without accidentally comparing unrelated units;
6. site/rack saturation and remaining worker capacity;
7. counter resets after a rollout;
8. absent targets versus present-but-stale health observations;
9. error contribution by failure class;
10. enrichment of device telemetry with bounded inventory metadata using vector matching;
11. desired-versus-observed state mismatch;
12. current versus offset historical behavior;
13. a subquery and a recording rule alternative, with cost comparison;
14. a query that is syntactically valid but semantically wrong, plus proof of the error.

### Advanced constraints

- Compare classic and native histogram query shapes using the client/server versions you actually run.
- Explain `rate` versus `irate`, `increase`, `delta`, and `deriv` by metric semantics.
- Demonstrate one-to-one and many-to-one vector matching and identify the risk of many-to-many matches.
- Inspect query execution cost and the number of input/output series before turning a broad query into a dashboard panel.

### Acceptance gate

Given an unfamiliar query, you can predict its type, output labels, counter-reset behavior, aggregation boundary, and likely cardinality. Complete the lab without receiving finished expressions from the agent.

Use the official [querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/), [operators](https://prometheus.io/docs/prometheus/latest/querying/operators/), and [functions](https://prometheus.io/docs/prometheus/latest/querying/functions/).

## 11. Milestone 5 — Operational Grafana dashboards

An **operational dashboard** supports a specific audience and decision. It should preserve units and aggregation meaning, expose missing data, and provide a deliberate drill-down path.

### Learner work

Build and provision at least three dashboards:

| Dashboard | Primary question |
| --- | --- |
| Service/SLO overview | Are users receiving successful, timely processing? |
| Fleet overview | Which site/rack/node/device or link is unhealthy or saturated? |
| Temporal operations | Is delay in API, task queue, Worker, Temporal service, or persistence? |

Use variables for site, rack, node, device, service, and Task Queue where bounded. Use dashboard links or data links to preserve investigation context. Choose panels intentionally: time series for trend, stat for current headline, table for ranked inventory, and heatmap for distributions when the metric supports it.

### Dashboard review checklist

- units, decimals, null behavior, time zone, and legends are explicit;
- rate windows adapt correctly to dashboard interval and scrape interval;
- p99 panels do not average already-computed quantiles;
- variables do not create unsafe regexes or enormous query expansions;
- every red/green threshold has a documented reason;
- a deployment or failure event is visible as an annotation;
- repeated panels are bounded and useful;
- dashboards and data sources are provisioned from version-controlled artifacts;
- query inspector evidence is recorded for the most expensive panel.

### Acceptance gate

Run a blind drill: another person or the teaching agent chooses a fault, and you navigate from symptom to the most specific useful component without opening Prometheus first.

See Grafana's official guidance on [dashboard variables](https://grafana.com/docs/grafana/latest/visualizations/dashboards/variables/), [Prometheus variables](https://grafana.com/docs/grafana/latest/datasources/prometheus/template-variables/), [provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/), and [dashboard practices](https://grafana.com/docs/grafana/latest/visualizations/dashboards/build-dashboards/best-practices/).

## 12. Milestone 6 — SLIs, SLOs, error budgets, and recording rules

A **service-level indicator (SLI)** is a measured user outcome. A **service-level objective (SLO)** is the target over a window. The **error budget** is the allowed amount of bad service within that objective.

### Learner work

Define and defend at least three SLOs:

- API acceptance availability;
- end-to-end job completion or latency;
- fleet telemetry/health-check freshness.

For each, document the user, good events, valid events, exclusions, window, target, low-traffic behavior, data gaps, and ownership. Prefer threshold-based latency SLIs when the objective is “under X seconds”; do not substitute a p99 chart unless it measures the same contract.

Create tested recording rules for common rates, ratios, histogram thresholds/quantiles, and SLO burn inputs. Use a consistent naming scheme and preserve the labels required by downstream consumers. Test syntax and behavior, including no traffic, total failure, counter reset, missing series, and partial-site failure.

### Acceptance gate

- Recalculate an error budget manually from controlled traffic and match the rule result.
- Explain why averaging per-instance ratios or quantiles can be wrong.
- Explain when a rule reduces query cost and when it only hides a bad metric model.
- Rule tests fail when an intentionally wrong aggregation is introduced.

Read Prometheus's [recording-rule practices](https://prometheus.io/docs/practices/rules/) and [rule configuration/testing](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/).

## 13. Milestone 7 — Alerting, Alertmanager, and runbooks

An **alert** should represent a condition requiring a timely human action. Prometheus evaluates alert rules; Alertmanager groups, routes, inhibits, silences, and delivers the resulting alerts.

### Learner work

Implement:

- fast-burn and slow-burn SLO alerts using multiple windows;
- one telemetry-staleness alert;
- one Worker/task-queue saturation alert;
- one monitoring-pipeline/meta-monitoring alert;
- severity/team/service/site labels and useful summary, description, dashboard, and runbook annotations;
- an Alertmanager routing tree, grouping policy, inhibition rule, and maintenance silence exercise;
- a local webhook receiver or equivalent observable notification target.

Write a runbook for each page. A runbook must include impact, confirmation queries, first safe actions, escalation, rollback/mitigation, and false-positive notes.

### Failure lab

Cause a site-wide condition that also produces device-level symptoms. Configure grouping/inhibition so the operator receives the smallest useful notification set without losing affected-device context.

### Acceptance gate

- Alert rules have executable tests for pending, firing, and recovery behavior.
- A notification arrives with enough context to act without editing the rule query.
- You can explain the difference between an alert's `for` duration and a multi-window SLO condition.
- You can demonstrate a silence that is narrowly scoped and expires.

Use Prometheus's [alerting practices](https://prometheus.io/docs/practices/alerting/), [Alertmanager overview](https://prometheus.io/docs/alerting/latest/overview/), and [Alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/). Google's SRE Workbook explains [multi-window, multi-burn-rate alerting](https://sre.google/workbook/alerting-on-slos/).

## 14. Milestone 8 — Kubernetes foundations and application deployment

Kubernetes provides declarative workload orchestration. Learn each resource's ownership and failure semantics before packaging the application with Helm.

Follow Stages B-E of [SETUP.md](SETUP.md).

### Learner work

- create a reproducible multi-node `kind` cluster;
- containerize and deploy API, Workers, fleet simulator, and load generator;
- use Deployments for stateless services and explain any StatefulSet choice;
- define Services, ConfigMaps, Secrets, ServiceAccounts/RBAC, resource requests/limits, and persistent volumes where needed;
- implement distinct startup, readiness, and liveness semantics;
- install the monitoring stack with a pinned chart and learner-authored values;
- discover application targets through ServiceMonitor/PodMonitor resources;
- create recording/alert rules as Kubernetes resources;
- perform a rolling update and rollback while observing both user and cluster signals.

### Failure experiments

- bad readiness probe, then bad liveness probe;
- CPU throttling or memory pressure;
- Worker replica loss and node drain;
- Service selector mismatch and ServiceMonitor selector mismatch;
- pending Pod due to resource requests;
- a Pod that is Running but not useful.

### Acceptance gate

- Explain desired versus current state through controllers, Pods, and EndpointSlices.
- Predict which metrics come from the app, kubelet/cAdvisor, kube-state-metrics, and Kubernetes components.
- Prometheus discovers the app without hard-coded Pod IPs.
- A rollout's effect is visible in application metrics, Temporal behavior, and Kubernetes state.

Read Kubernetes on [workloads](https://kubernetes.io/docs/concepts/workloads/), [probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/), [observability](https://kubernetes.io/docs/concepts/cluster-administration/observability/), and [object-state metrics](https://kubernetes.io/docs/concepts/cluster-administration/kube-state-metrics/).

## 15. Milestone 9 — Temporal and full-stack monitoring on Kubernetes

Self-hosting Temporal introduces stateful persistence, multiple service roles, schema management, operational metrics, and independent scaling of Workers and the Temporal service.

Follow Stage F of [SETUP.md](SETUP.md).

### Learner work

- deploy a pinned Temporal Helm chart backed by a learner-operated PostgreSQL service;
- connect in-cluster API and Worker clients through the Temporal frontend Service;
- scrape Temporal frontend, history, matching, Worker, and persistence-related metrics through the existing Prometheus stack;
- build dashboard rows that separate user workflow health, Worker/task-queue health, Temporal service health, and Kubernetes resource health;
- define resource and replica strategy for API, Workers, Temporal roles, and persistence;
- document schema upgrade, backup/restore, and version-skew risks even if the local lab does not automate them;
- test Worker deployment version compatibility and a safe rollout strategy.

### Failure experiments

- delete a Worker Pod during an Activity;
- scale Workers to zero and back;
- restart a Temporal frontend Pod;
- inject persistence latency or unavailability in a controlled lab method;
- exhaust Worker capacity without exhausting API capacity;
- create a task-queue name or label that would cause unsafe metric cardinality and redesign it.

### Acceptance gate

- Workflow durability is shown across Worker replacement and explained from event history.
- An operator can distinguish schedule-to-start delay from execution latency.
- Temporal service unavailability, Worker unavailability, and application failure produce distinguishable signals.
- The monitoring stack is singular, explicit, and reproducible; Temporal does not hide a second Prometheus/Grafana deployment.

## 16. Milestone 10 — Fleet decoration, correlation, and health-check framework

**Decoration** adds bounded, authoritative context to raw telemetry. **Correlation** combines signals using stable identity and time semantics. Neither should silently turn Prometheus into the fleet database.

### Learner work

Create a versioned inventory/control-plane API and a health-check framework that supports:

- site/rack/node/device/link identity and lifecycle state;
- check type/version, start/completion time, duration, result class, and last-success freshness;
- bounded reason classes with detailed errors kept outside metric labels;
- desired-versus-actual state reconciliation;
- explicit identity rules and orphan/conflict handling;
- a supported API view for common operational questions so other components do not need unrestricted raw production scraping.

Compare three decoration locations:

| Location | Strength | Risk |
| --- | --- | --- |
| Target/service-discovery labels | Consistent context at ingestion | Metadata change creates new series; scrape ownership required |
| Exported info metric plus PromQL join | Preserves raw measurement labels | Vector matching complexity and stale identity conflicts |
| Recording/correlation service | Stable downstream contract and cheaper queries | Additional pipeline, freshness, and ownership concerns |

Choose one primary design and implement enough of another to compare failure behavior.

### Acceptance gate

- A new site and device model can be added without changing metric names or cloning dashboards.
- Conflicting inventory and telemetry identity fails visibly.
- An operator can distinguish unhealthy, unknown, stale, draining, and decommissioned states.
- You can explain the consistency model and maximum time until a metadata change appears in each consumer.

## 17. Milestone 11 — Prometheus operations, HA, and long-term storage

Prometheus local TSDB is a single-node storage engine. Advanced operation requires monitoring ingestion, query, rule, disk, scrape, and remote-write health, then choosing explicit HA and retention semantics.

### Learner work

Operate and explain:

- WAL, head series/chunks, block compaction, retention, and restart recovery;
- sample/label/body limits and a controlled scrape-limit failure;
- scrape duration/sample count, rule duration/failures, query load, and disk forecasts;
- cardinality/churn investigation using Prometheus status APIs and self-metrics;
- remote-write queue flow, pending/failed/retried samples, sharding, backpressure, and data-loss window;
- two Prometheus replicas with external labels, deduplication expectations, and alert delivery behavior.

Write an architecture decision record comparing:

| Option | Evaluate |
| --- | --- |
| Prometheus federation | Hierarchical selected aggregation and global-query limitations |
| Thanos | Sidecar/Receive, Querier, Store Gateway, Compactor, object storage, deduplication, partial responses |
| VictoriaMetrics | Single/cluster topology, Prometheus compatibility, remote write, operational complexity |

Implement one bounded scale-out proof, preferably Thanos with local object storage or a VictoriaMetrics remote-write target. Do not attempt production-scale load on a laptop; model and measure enough to defend the design.

### Acceptance gate

- Diagnose a remote-write backlog without immediately increasing every queue setting.
- Explain what an HA pair does and does not protect against.
- Explain query freshness and failure semantics during loss of one Prometheus replica or object-store access.
- Provide capacity estimates and a migration path for 10,000 nodes and tens of thousands of devices.

Read Prometheus on [storage](https://prometheus.io/docs/prometheus/latest/storage/), [remote-write tuning](https://prometheus.io/docs/practices/remote_write/), and [federation](https://prometheus.io/docs/prometheus/latest/federation/), plus the official [Thanos component guide](https://thanos.io/tip/components/).

## 18. Milestone 12 — Game days and capstone defense

A **game day** is a controlled failure exercise with a hypothesis, safety boundary, evidence timeline, mitigation, and follow-up. It tests the operating system and the operator.

### Required incidents

Run at least five without announcing the fault to the investigator:

1. a single device becomes hot and begins corrected-error growth;
2. a link degrades and increases only one workflow stage's latency;
3. all Workers for one Task Queue disappear;
4. a cardinality regression creates head-series growth and query slowdown;
5. remote write or long-term storage becomes unavailable;
6. optional: Temporal persistence latency causes control-plane degradation;
7. optional: a bad Kubernetes probe creates a restart cascade.

For each incident, record:

- detection source and time;
- user impact and SLO burn;
- first useful dashboard/query and misleading signals;
- timeline from symptom to cause;
- mitigation and verification;
- systemic fix, owner, and prevention test;
- what Prometheus/Grafana could not tell you and which other signal supplied it.

### Capstone package

- architecture and data-flow diagram;
- metric/label contract and cardinality budget;
- provisioned dashboards, rules, alerts, and runbooks;
- Temporal and Kubernetes design decisions;
- tested failure scenarios and one blameless postmortem;
- HA/long-term-storage architecture decision record;
- a 10-minute demo and a 30-minute architecture defense.

### Final interview questions

1. Why does every label create a capacity concern, and when is per-device cardinality justified?
2. How do you calculate a histogram quantile correctly across replicas?
3. What happens to a counter through process restart, missed scrapes, and HA deduplication?
4. How would you make tens of thousands of devices legible without one dashboard per device?
5. How do Temporal retries, Kubernetes restarts, and application idempotency interact?
6. What distinguishes schedule-to-start latency from Activity execution latency?
7. How do you alert on an SLO while avoiding noise during low traffic?
8. Where should fleet metadata decoration happen, and how does it become stale?
9. How do you know the monitoring system itself is failing?
10. When would you choose federation, Thanos, or VictoriaMetrics?

### Final acceptance gate

You can receive an unfamiliar failure, form and revise a hypothesis using evidence, mitigate it safely, and explain the architecture's scaling and consistency trade-offs without the teaching agent supplying the answer.

## 19. Choosing what to learn locally versus conceptually

| Goal | Required depth |
| --- | --- |
| Instrumentation, PromQL, dashboards, rules, alerts | Implement and test fully |
| Temporal Workflows/Activities and Worker failure | Implement and test fully |
| Kubernetes deployment and discovery | Implement and operate locally |
| Fleet decoration/health checks | Implement a representative bounded version |
| Prometheus TSDB and remote-write failure | Operate a controlled local experiment |
| Thanos or VictoriaMetrics | Implement one proof; deeply design both |
| 10,000-node capacity | Model from measured per-node data; do not emulate blindly |
| Production multi-region Temporal/Postgres | Design and failure-analyze; local lab cannot prove it |

## 20. Problem-solving framework

For any observability failure:

1. State the user-visible symptom and decision being made.
2. Identify the source of truth and its freshness boundary.
3. Trace the signal path: event → instrumentation → exposition → discovery → scrape → storage → rule/query → dashboard/alert.
4. Predict the metric values and labels at each boundary.
5. Inspect the earliest boundary that contradicts the prediction.
6. Change one variable and preserve evidence.
7. Verify recovery from the user's perspective and the monitoring system's perspective.
8. Turn the systemic cause into a test, limit, runbook, or design change.

### Applied question

**Question:** Job latency rises, CPU is low, Temporal is healthy, and Workers are Running.

Do not jump to scaling. Split end-to-end latency into API, Temporal schedule-to-start, Activity execution, and retry/queue components. “Running” is Kubernetes lifecycle state, not evidence that Workers poll the correct Task Queue or have available execution slots.

### Applied question

**Question:** A site disappears from the fleet dashboard.

Distinguish an empty-but-valid result, dashboard-variable filtering, query/vector-match loss, recording-rule failure, scrape/discovery failure, and an actual site outage. Inspect the signal path in that order instead of treating “no data” as zero.

## 21. Practice prompts

- [ ] Sketch the series multiplier for a classic histogram with site, route, status class, and eight finite buckets.
- [ ] Explain why `average(p99_per_pod)` is not a fleet p99.
- [ ] Design an alert label set that groups a site outage without losing site identity.
- [ ] Classify ten candidate labels as bounded category, bounded inventory, or unbounded event identity.
- [ ] Predict Workflow behavior when an Activity completes its side effect but the completion response is lost.
- [ ] Explain the difference between a failed readiness probe and a failed liveness probe during high load.
- [ ] Identify which component should own job state, desired fleet state, device temperature, and SLO compliance.
- [ ] Explain how a remote-write outage can increase Prometheus memory pressure.
- [ ] Defend one reason to use a recording rule and one reason not to.
- [ ] Design a blind game day that tests both the telemetry system and the operator.

### Answer-key standard

There is intentionally no copyable implementation answer key. A correct response must name the invariant, show its label/type or failure semantics, state a trade-off, and cite measured evidence from this project. Use the teaching agent for review after attempting each prompt.

## 22. Summary

1. Build from operating questions and ownership contracts, then choose metrics and tools.
2. Keep unbounded event identity out of Prometheus labels; quantify bounded fleet cardinality instead of avoiding it blindly.
3. Temporal durably coordinates job state, Kubernetes runs processes, and Prometheus observes aggregates.
4. Correct PromQL depends on types, labels, time, counter behavior, aggregation order, and missing data.
5. Grafana should provide reusable overview-to-detail investigation paths with version-controlled provisioning.
6. SLOs describe user outcomes; alerts should be actionable threats to those outcomes and route without storms.
7. Monitor the application, Temporal, Kubernetes, and the monitoring pipeline itself.
8. Scale-out designs require explicit durability, deduplication, consistency, query, and operational trade-offs.
9. Failure injection, rule tests, replay tests, game days, and postmortems turn tool familiarity into production skill.
