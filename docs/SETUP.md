# Staged Local Setup: Kubernetes and Temporal

Install only what the current milestone needs. This keeps failures attributable: first to application code, then Temporal, then Kubernetes, and finally the integrated platform.

## 1. Detected host baseline

The repository was initialized on an Apple Silicon Mac (`arm64`). At the time this guide was created:

| Tool | State |
| --- | --- |
| Homebrew | Installed |
| Docker CLI | Installed; daemon availability not confirmed |
| `kubectl` | Installed, client v1.36.1 |
| `kind` | Not installed |
| Helm | Not installed |
| Temporal CLI | Not installed |

Re-check rather than assuming this table is still current.

## 2. Choosing a local Kubernetes distribution

| Choice | Best fit | Trade-off |
| --- | --- | --- |
| `kind` | This project; reproducible local and CI clusters | Runs Kubernetes nodes as containers and requires Docker or Podman |
| `minikube` | Learning multiple drivers and add-ons | More distribution-specific conveniences can hide the base mechanics |
| Docker Desktop Kubernetes | Fastest if already enabled | Less reproducible in CI and coupled to a desktop setting |

Use `kind` for the main path. Kubernetes lists both `kind` and `minikube` as local options, and `kind` supports Docker or Podman. See the [Kubernetes tool guide](https://kubernetes.io/docs/tasks/tools/) and [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/).

## 3. Stage A: local application tools

Use this stage for Milestones 0-2. Do not create a cluster yet.

### Verify the container runtime

Start Docker Desktop or another Docker-compatible engine, then verify both the client and server respond:

```sh
docker version
docker info
```

If `docker version` shows only a client section or `docker info` fails, the daemon is not reachable. Fix that before using `kind`.

### Install the Temporal CLI

The official Temporal CLI includes a development server. On this Mac:

```sh
brew install temporal
temporal --version
```

For Milestone 2, run the local server in its own terminal:

```sh
temporal server start-dev
```

The default frontend is available to SDK clients on port `7233`, and the command reports the Temporal UI address when it starts. This server is for development, not a model for production persistence or availability. See the official [Temporal CLI repository](https://github.com/temporalio/cli) and [Temporal documentation](https://docs.temporal.io/).

## 4. Stage B: install the Kubernetes client tools

Use this at the beginning of the Kubernetes milestone.

```sh
brew install kind helm
kubectl version --client
kind version
helm version
```

The `kubectl` client should be within one minor version of the cluster control plane. Verify compatibility after cluster creation rather than assuming the newest clients and node images match. See [installing kubectl on macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) and [installing Helm](https://helm.sh/docs/intro/install/).

## 5. Stage C: create the learning cluster

First inspect the `kind` version and its supported Kubernetes node images. Then create a named cluster:

```sh
kind create cluster --name observable-processing --wait 5m
kubectl cluster-info --context kind-observable-processing
kubectl get nodes -o wide
```

For reproducibility, the learner must later write and commit a `kind` cluster configuration that pins an official node image, defines the required nodes, and documents host port mappings. Do not copy a finished configuration from the agent.

Suggested capacity for the complete learning stack is roughly 6 CPU cores and 10-12 GiB of container-engine memory. This is a planning estimate, not a Kubernetes requirement. If the machine cannot provide it, reduce replicas, omit optional components, and document how the reduced topology changes the failure model.

### Cluster acceptance check

You should be able to explain and demonstrate:

- the active kubeconfig context;
- a control-plane node and at least one schedulable worker node in the final topology;
- DNS resolution between two test workloads;
- how a locally built image becomes available inside `kind`;
- what data is lost when the cluster is deleted.

## 6. Stage D: install the monitoring stack

Use the community `kube-prometheus-stack` chart to install Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, and kube-state-metrics. The learner must inspect the chart, select a compatible pinned version, and write a values file before installing:

```sh
helm show chart oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
helm show values oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack
```

The eventual install should be an `upgrade --install` operation using:

- a pinned chart version;
- a dedicated `monitoring` namespace;
- the learner-authored values file committed to the repository;
- resource requests/limits appropriate for the local cluster;
- explicit retention and persistent-storage decisions.

Do not rely only on bundled dashboards. Build the project dashboards yourself. The chart and its components are documented in the [kube-prometheus-stack README](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/README.md).

## 7. Stage E: deploy the application and discover it

The learner containers and deploys the API, Temporal Workers, fleet simulator, and load generator. Prometheus discovers targets through learner-authored `ServiceMonitor` or `PodMonitor` resources.

Before installing, render and inspect every resource:

```sh
helm template <release> <chart-path> --namespace <namespace>
```

The acceptance check is not merely that `up == 1`. Explain how Service selectors, ServiceMonitor selectors, namespaces, ports, and target labels combine to create the final time series.

## 8. Stage F: self-host Temporal on Kubernetes

Do this only after the learner can run and observe Workflows against the CLI development server.

Use the official [Temporal Helm chart](https://github.com/temporalio/helm-charts). Current chart releases require the operator to provide persistence and a monitoring stack rather than silently bundling everything. For this lab:

1. Select and pin a chart version compatible with the selected Temporal Server version.
2. Provide a learner-operated PostgreSQL persistence service and document its durability limitations in `kind`.
3. Write a Temporal values file with explicit persistence, replica, resource, metrics, and visibility decisions.
4. Render the chart and inspect Deployments/StatefulSets, Services, ConfigMaps, Secrets, and schema jobs before installation.
5. Install into a dedicated `temporal` namespace.
6. Enable the chart's metrics endpoints and create or enable the appropriate ServiceMonitors for the existing Prometheus Operator.
7. Connect application Workers through the in-cluster Temporal frontend Service rather than host port-forwarding.
8. Use port-forwarding only for local human access to Temporal UI or debugging.

Do not install a second hidden Prometheus/Grafana stack with Temporal. One explicit monitoring platform should scrape both the application and Temporal.

### Temporal-on-Kubernetes acceptance check

Demonstrate:

- a Workflow survives an application Worker Pod restart;
- Activity retries are visible in Temporal history and aggregate metrics without using Workflow IDs as Prometheus labels;
- scaling Worker replicas changes task-queue behavior in an explainable way;
- Temporal service health and application workflow health appear as separate dashboard sections;
- the learner can identify which state is lost after deleting a Worker, a Temporal frontend Pod, a database Pod, a PVC, or the entire `kind` cluster.

## 9. Choosing where each concern belongs

| Concern | Primary home |
| --- | --- |
| Restarting a crashed stateless Worker process | Kubernetes Deployment |
| Remembering a multi-step job after Worker loss | Temporal Workflow history |
| Retrying a fallible external operation | Temporal Activity retry policy plus idempotency |
| Determining whether a Pod should receive traffic | Kubernetes readiness probe |
| Aggregate rate, latency, saturation, or error signal | Prometheus metric |
| Interactive fleet and workflow investigation | Grafana dashboard |
| Per-job causal history | Temporal history and structured logs/traces |

## 10. Teardown

Teardown is destructive. Inspect the active context and exact target before running it:

```sh
kubectl config current-context
kind get clusters
kind delete cluster --name observable-processing
```

Deleting the cluster removes cluster-local workloads and storage. Preserve only artifacts intentionally committed to the repository; do not treat the learning cluster as durable storage.

## 11. Setup troubleshooting framework

When a component fails:

1. Identify the layer: host runtime, cluster, Kubernetes resource, network discovery, application, Temporal, or metrics pipeline.
2. Predict which control-plane state and metrics should exist.
3. Inspect events and resource status before restarting anything.
4. Check endpoints from both sides of the network boundary.
5. Change one variable, record the result, and restore the intended state.

Do not use repeated reinstalling as the default debugging method; it destroys evidence.

