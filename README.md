# RabbitMQ Observability

RabbitMQ observability on Kubernetes with Prometheus and Grafana, queue and consumer monitoring, operational alerts, and KEDA autoscaling analysis.

> **Status: monitoring stack deployed and RabbitMQ queue collection validated in the existing local cluster.** Prometheus returned queue-specific metrics for `tasks` during a message submission test. Grafana dashboards, custom alerts, and complete autoscaling experiments are still pending. See [experiment evidence](docs/experiments.md).

## Why this project?

Autoscaling changes the number of workers, but replica count alone does not explain whether messages are being processed successfully or whether a backlog is growing faster than consumers can handle it.

This project aims to connect queue metrics, consumer activity, and Kubernetes workload state so that operational questions can be answered with evidence:

- Is the queue accumulating work or draining?
- Are consumers connected and acknowledging messages?
- Does KEDA scale workers as demand changes?
- Are workers at their replica limit while backlog continues to grow?
- Has processing finished, or are messages still unacknowledged?

The goal is to build versioned monitoring configuration and validate it through controlled scenarios, including alerts with actionable investigation steps.

## Relationship with the autoscaling project

The monitored application comes from [keda-rabbitmq-autoscaling](https://github.com/aurelio-miguel/keda-rabbitmq-autoscaling). Both projects run in the same local Kubernetes cluster.

| Repository | Responsibility |
| --- | --- |
| `keda-rabbitmq-autoscaling` | Python producer and worker, Docker images, RabbitMQ, application manifests, and KEDA scaling configuration |
| `rabbitmq-observability-lab` | Monitoring stack configuration, scrape integration, dashboards, alert rules, runbooks, and observability experiments |

The producer, worker, broker, and KEDA manifests remain in the application repository. This repository will reference a recorded application commit or release rather than maintaining duplicate copies. Monitoring-specific Services and ServiceMonitors belong here when needed.

Enabling RabbitMQ metrics may require a change in the application repository, such as enabling its Prometheus plugin and exposing a metrics endpoint. That change should be documented and versioned there; scrape configuration belongs here.

## Intended architecture

| Component | Responsibility |
| --- | --- |
| RabbitMQ metrics endpoint | Expose broker and queue metrics for collection |
| Prometheus | Scrape metrics and evaluate recording and alert rules |
| Grafana | Display versioned dashboards for queue demand and workload behavior |
| Kubernetes metrics sources | Expose Deployment state and, where configured, pod/container resource metrics |
| Alertmanager | Group and display alerts; external notification integrations are optional |
| Existing KEDA lab | Supply the application, load generation, and autoscaling scenarios |

Prometheus observes the existing application; it does not publish or consume business messages. Grafana visualizes collected data. KEDA continues to control worker scaling independently of the monitoring dashboards.

The monitoring namespace is `monitoring`. The existing application namespace is `keda-rabbitmq-autoscaling`. Cross-namespace discovery and access must be configured explicitly.

## Initial scope

- Deploy Prometheus, Grafana, and Alertmanager using versioned Helm configuration.
- Enable and verify RabbitMQ metric collection.
- Collect Kubernetes workload-state metrics for the worker Deployment.
- Create a dashboard combining queue demand, consumers, and worker replicas.
- Define alerts for controlled lab failures and document investigation steps.
- Record experiments with timestamps, screenshots, and measurement limits.
- Provide setup and cleanup instructions with explicit cluster context selection.

Logs, distributed traces, application-level latency instrumentation, and SLO-based alerting are later extensions. The first version focuses on metrics and operational diagnosis.

## Metrics and dashboard design

| Signal | Operational question | Required source |
| --- | --- | --- |
| Ready messages | How much work is waiting for delivery? | RabbitMQ queue metrics |
| Unacknowledged messages | How much work is in flight? | RabbitMQ queue metrics |
| Connected consumers | Are workers attached to the queue? | RabbitMQ queue metrics |
| Publication and acknowledgement rates | Is processing keeping up with incoming work? | RabbitMQ counters converted to rates |
| Redelivery rate | Is work being delivered again? | RabbitMQ delivery counters |
| Desired and available worker replicas | Is the workload following scaling decisions? | Kubernetes workload-state metrics |
| Restarts and resource utilization | Are worker failures or resource constraints affecting consumption? | Kubernetes state and container metrics |
| Scrape health | Is monitoring itself collecting data? | Prometheus target health |

The endpoint `/metrics/per-object` exposed `rabbitmq_queue_messages_ready`, `rabbitmq_queue_messages_unacked`, and `rabbitmq_queue_consumers` with `queue="tasks"` and `vhost="/"`. Prometheus collection of ready messages was confirmed during a message submission test. Publication, acknowledgement, and redelivery counters still need verification before dashboard queries are committed. Broker-wide aggregates are not substitutes for queue-specific measurements.

A Kubernetes Ready pod does not necessarily mean a RabbitMQ consumer is connected. An empty ready queue does not prove completion while unacknowledged messages remain.

## Planned alert scenarios

| Alert condition | Investigation focus |
| --- | --- |
| Backlog persists with no consumers | Worker state, connection failures, credentials, and queue name |
| Backlog remains high at the replica ceiling | Consumer capacity, processing delay, available resources, and scaling limit |
| Redelivery rate remains elevated | Worker interruptions, connection failures, and acknowledgement behavior |
| RabbitMQ metrics target is unavailable | Broker health, plugin configuration, Service discovery, and network access |

Thresholds and evaluation windows will be derived from controlled runs rather than copied as production recommendations. No-consumer alerts must account for normal scale-to-zero behavior and startup delay: zero consumers with an empty queue is expected.

Each rule should include its purpose, severity, duration, relevant labels, and a runbook. Missing telemetry must be distinguishable from a healthy zero value.

## Prerequisites

- A local Kubernetes cluster; Kind is the initial target.
- Docker, Kind, kubectl, Helm, Bash, and Python 3 for the existing application setup.
- The KEDA/RabbitMQ application deployed and working.
- A working storage provisioner where persistence is requested.
- Capacity for the application and monitoring stack; resource sizing remains to be measured.

The application and monitoring stack were verified using context `kind-freekubelab`. RabbitMQ queue collection was confirmed through a Prometheus query during a message submission test. An AWS account is not required for the initial scope.

## Prepare the monitored application

Clone the existing application repository alongside this project:

```bash
git clone https://github.com/aurelio-miguel/keda-rabbitmq-autoscaling.git
cd keda-rabbitmq-autoscaling
git rev-parse HEAD
```

Record the commit used by the experiment. Follow that repository's README to prepare credentials, storage, images, RabbitMQ, workers, and KEDA. Setup requires an existing cluster; it does not create one.

Before submitting load, explicitly select the intended lab context:

```bash
kubectl config use-context kind-freekubelab
kubectl --context kind-freekubelab get pods -n keda-rabbitmq-autoscaling
kubectl --context kind-freekubelab get scaledobject,hpa -n keda-rabbitmq-autoscaling
```

Return to this repository to configure monitoring. Run the following commands from its root. The application must already expose metrics on port `15692`; plugin enablement belongs in the application repository and must survive pod recreation.

## Install monitoring

The selected chart version is `kube-prometheus-stack` **92.3.0**. The values configure one Prometheus replica, three-day retention, 30-second scrape and evaluation intervals, resource requests and memory limits, and ClusterIP access. Grafana persistence is disabled and no Prometheus persistent volume is configured; data can be lost when pods are recreated. Resource sizing is an initial configuration, not a measured capacity recommendation.

Download the chart through the official OCI registry, then install the local archive:

```bash
helm pull oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack \
  --version 92.3.0 --destination /tmp

helm upgrade --install monitoring /tmp/kube-prometheus-stack-92.3.0.tgz \
  --kube-context kind-freekubelab \
  --namespace monitoring --create-namespace \
  --values helm/monitoring-values.yaml \
  --wait --timeout 10m

kubectl --context kind-freekubelab apply -f monitoring/services/rabbitmq-metrics.yaml
kubectl --context kind-freekubelab apply -f monitoring/service-monitors/rabbitmq.yaml
kubectl --context kind-freekubelab -n monitoring get pods
```

The traditional Helm repository is also supported, using `prometheus-community/kube-prometheus-stack` and `--version 92.3.0`. Downloads from GitHub release assets timed out during setup; OCI offers another distribution path. The exact installation route used for the successful deployment was not recorded. The Helm wait timeout does not resolve download or DNS failures.

The metrics Service lives in `keda-rabbitmq-autoscaling` and selects pods with `app: rabbitmq`. Its named port `metrics` maps `15692` to `15692`. The ServiceMonitor lives in `monitoring`, has label `release: monitoring`, and selects Services with `app: rabbitmq` and `component: metrics` in the application namespace. It scrapes `/metrics/per-object` every 30 seconds with a 10-second scrape timeout.

## Verify collection

Access RabbitMQ metrics locally in one terminal:

```bash
kubectl --context kind-freekubelab -n keda-rabbitmq-autoscaling \
  port-forward service/rabbitmq-metrics 15692:15692
```

In another terminal:

```bash
curl -fsS http://localhost:15692/metrics/per-object |
  rg '^rabbitmq_queue_(messages_ready|messages_unacked|consumers)\{'
```

To access Prometheus, start another port-forward:

```bash
kubectl --context kind-freekubelab -n monitoring \
  port-forward service/monitoring-kube-prometheus-prometheus 9090:9090
```

Open `http://localhost:9090/targets`, check the RabbitMQ target, and run:

```promql
rabbitmq_queue_messages_ready{queue="tasks",vhost="/"}
```

A returned zero is a collected measurement; an empty result is missing telemetry. Port-forward is only for local inspection: Prometheus collects through the internal Service. If forwarding reports `15692 -> 15962`, correct the Service target port to `15692` and reapply it.

Setup and scoped cleanup scripts have not been implemented yet.

## Validation plan

1. Verify RabbitMQ exposes the required metrics, including queue-level labels.
2. Verify Prometheus discovers the intended targets and successfully scrapes them.
3. Compare dashboard values with RabbitMQ queue counts and Kubernetes replica state.
4. Generate a controlled workload using the application repository's load script.
5. Observe activation, scale-out, queue drain, and return to zero on the dashboard.
6. Introduce a controlled failure and confirm alert firing and resolution.
7. Repeat the scenario and record versions, configuration, timestamps, and evidence.

Record both ready and unacknowledged message counts at completion. Keep workloads separate and verify the queue is empty before comparable runs. Avoid rebuilding or changing worker configuration during a measurement.

## Repository structure

Implemented files:

```text
rabbitmq-observability-lab/
├── helm/
│   └── monitoring-values.yaml
├── monitoring/
│   ├── services/
│   │   └── rabbitmq-metrics.yaml
│   └── service-monitors/
│       └── rabbitmq.yaml
├── docs/
│   └── experiments.md
└── README.md
```

Recording rules, custom alerts, Grafana dashboards, runbooks, setup/cleanup scripts, and CI checks will be added as they are implemented. Application source code and deployment manifests remain in the upstream KEDA project.

## Implementation roadmap

- [ ] Record the application commit and validate its required metrics configuration.
- [x] Select and pin a monitoring Helm chart version: `92.3.0`.
- [x] Define initial resource requests, memory limits, and retention settings (capacity measurement pending).
- [x] Deploy the monitoring stack and confirm RabbitMQ discovery through a collected queue series.
- [x] Implement RabbitMQ scrape integration.
- [ ] Validate Kubernetes worker-state and resource metric queries.
- [ ] Version the initial Grafana dashboard.
- [ ] Implement alert rules and runbooks.
- [ ] Validate healthy, slow-consumer, no-consumer, and replica-ceiling scenarios.
- [ ] Add setup and scoped cleanup scripts with context validation.
- [ ] Add CI checks for configuration, alert rules, and dashboard JSON.
- [ ] Document results and validate setup from a fresh environment.

## Engineering principles

Monitoring configuration should be reproducible and versioned. Record chart versions, application commits, metric definitions, and experiment conditions. Keep dashboards and alert rules in Git instead of relying only on changes made through the Grafana interface.

Separate observed results from expected behavior. A dashboard screenshot is supporting evidence, not proof of delivery guarantees or sustained performance. Unit and configuration checks complement live collection and failure experiments.

Keep credentials out of Git, dashboards, shared logs, and screenshots. Access to Grafana, Prometheus, and the RabbitMQ management interface should remain local for the initial lab. Cleanup must be scoped to monitoring resources and preserve the application by default.

## Limitations and future work

The current deployment targets a local Kubernetes environment. Production readiness has not been assessed. High availability, remote metric storage, production retention sizing, external alert delivery, and multi-cluster operation are outside the initial scope.

A processing-time SLO requires application instrumentation and a clear definition of successful processing and its denominator. Queue depth alone cannot establish latency compliance. Later iterations can add message latency metrics, logs, traces, and SLO/error-budget analysis after these signals are implemented and validated.

Initial RabbitMQ collection evidence is recorded in [docs/experiments.md](docs/experiments.md). Complete scaling behavior, alert firing and resolution, performance, and fresh-environment reproducibility have not yet been validated.
