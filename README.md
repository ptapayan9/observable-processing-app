# Observable Processing App

This repository is a guided, code-it-yourself observability capstone. You will build a small accelerator-job control plane that produces realistic traffic, latency, failures, retries, queue pressure, fleet-health changes, and infrastructure pressure. Prometheus, Grafana, Kubernetes, and Temporal are not decorative add-ons: each one must answer a concrete operating question.

The target is advanced working knowledge suitable for discussing a production-engineering observability platform. The project is tailored to Fluidstack's [Production Engineer, IaaS role](https://fluidstack.io/jobs/bff2c500-d321-4bde-b251-9f7a2ef2d894), especially its emphasis on fleet visibility from site to device, health checks, durable control-plane APIs, SLOs, Kubernetes, time-series systems, and Temporal/Cadence.

## The learner owns the code

You write all application code, tests, PromQL, rules, dashboards, containers, Kubernetes resources, Temporal workflows, and deployment configuration. The agent teaches concepts, asks design questions, reviews your work, helps you interpret evidence, and gives progressively stronger hints when you are stuck. The complete contract is in [AGENTS.md](AGENTS.md).

## The system you will build

```text
client/load generator
        |
        v
job API / control plane -----> Temporal frontend + durable workflow history
        |                                  |
        |                                  v
        +--------------------------> task queues ---> worker pool
                                                        |
                                                        v
                                             simulated accelerator fleet
                                             site/rack/node/device/link
                                                        |
                         +------------------------------+
                         v
             health checks and application metrics
                         |
                         v
                 Prometheus / rules
                    |           |
                    v           v
                 Grafana    Alertmanager
```

The final system must let an operator move from a fleet-wide symptom to a site, node, device, link, workflow stage, or metrics-pipeline failure without using a job ID as a Prometheus label.

## Learning path

The curriculum is in [docs/LEARNING_PATH.md](docs/LEARNING_PATH.md). It progresses through:

1. First-principles architecture and metric contracts.
2. Local application instrumentation and Prometheus scraping.
3. Temporal workflows, activities, retries, heartbeats, and task queues.
4. Advanced PromQL and cardinality analysis.
5. Grafana drill-down dashboards and provisioning.
6. SLIs, SLOs, recording rules, alerts, and runbooks.
7. A local Kubernetes cluster and Kubernetes-native discovery.
8. Self-hosted Temporal and the monitoring stack on Kubernetes.
9. Fleet metadata decoration, correlation, and health-check design.
10. Prometheus operations, HA, remote write, and long-term storage design.
11. Game days, postmortems, and an interview-ready capstone defense.

Installation is deliberately staged in [docs/SETUP.md](docs/SETUP.md). Do not install the entire stack on day one.

## Start here

1. Read [AGENTS.md](AGENTS.md).
2. Read the project outcome and Milestone 0 in [docs/LEARNING_PATH.md](docs/LEARNING_PATH.md).
3. Ask the agent: `Teach me Milestone 0. Follow AGENTS.md and do not write my implementation.`
4. Create the Milestone 0 design artifacts yourself.
5. Demonstrate the milestone evidence before moving on.

Progress is recorded in [docs/PROGRESS.md](docs/PROGRESS.md). A checked box means you demonstrated the acceptance evidence; it does not merely mean that a file exists.

## Local development

Install [`uv`](https://docs.astral.sh/uv/) before working with the application. The project requires Python 3.13 or newer; `uv` manages the project environment, so manual virtual-environment activation is unnecessary.

Create or synchronize the environment from the committed lockfile:

```sh
uv sync --locked
```

Run the application entry point:

```sh
uv run --locked oba
```

The current bootstrap application should print:

```text
Hello from oba!
```

### Quality checks

Synchronize the locked development environment:

```sh
uv sync --locked
```

Check Python source for lint violations:

```sh
uv run --locked ruff check src
```

Verify that Python source matches Ruff's formatter:

```sh
uv run --locked ruff format --check src
```

These commands are check-only and do not modify source files. GitHub Actions runs the same checks on pushes and pull requests.

### Package build and clean-artifact smoke test

Verify that the committed lockfile agrees with the project configuration, then build both the source distribution and wheel through the configured build backend:

```sh
uv lock --check
uv build
```

The build writes both artifact formats under `dist/`:

```text
dist/oba-0.1.0.tar.gz
dist/oba-0.1.0-py3-none-any.whl
```

Create a fresh Python 3.13 environment outside the repository and install only the built wheel into it:

```sh
oba_smoke_dir="$(mktemp -d /tmp/oba-smoke.XXXXXX)"
uv venv --python 3.13 "$oba_smoke_dir/venv"
uv pip install --python "$oba_smoke_dir/venv/bin/python" dist/*.whl
```

Invoke the installed command from outside the repository checkout:

```sh
(
  cd /tmp
  "$oba_smoke_dir/venv/bin/oba"
)

# Expected output:
# Hello from oba!
```

This smoke test proves that the tested wheel can be installed and started without borrowing OBA source or dependencies from the repository `.venv`. It does not prove complete application behavior, portability across every supported host, security, or production readiness. GitHub Actions reproduces the same build and clean-install boundary without publishing the generated artifacts.

## Durable principles

- Metrics are aggregate operating signals, not an event log or business database.
- Every unique label set is a time series with storage and query cost.
- Dashboards support investigation; alerts demand a specific human action.
- Temporal provides durable execution, while Kubernetes provides workload orchestration. Neither replaces the other.
- Prometheus should monitor the application, Temporal, Kubernetes, and its own metrics pipeline.
- Configuration and dashboards are production artifacts and belong in version control.
