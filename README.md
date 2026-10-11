# RabbitMQ Observability Lab

A Kubernetes observability lab for investigating RabbitMQ queue demand, consumer behavior, and KEDA-driven autoscaling with Prometheus and Grafana.

> **Status: planning and initial documentation.** This README defines the intended scope and implementation sequence. Monitoring manifests, dashboards, alerts, and experiment results have not yet been validated. Repository paths below are a proposed structure, not existing deliverables.

## Why this project?

Autoscaling changes the number of workers, but replica count alone does not explain whether messages are being processed successfully or whether a backlog is growing faster than consumers can handle it.

This lab aims to connect queue metrics, consumer activity, and Kubernetes workload state so that operational questions can be answered with evidence:

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

The proposed monitoring namespace is `observability`. The existing application namespace is `keda-rabbitmq-autoscaling`. Cross-namespace discovery and access must be configured explicitly.

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

Exact metric names, labels, and queue-level exposure will be verified against the selected RabbitMQ version before dashboard queries are committed. Broker-wide aggregates are not substitutes for queue-specific measurements.

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

The existing lab was validated using context `kind-freekubelab`. Monitoring installation on that cluster has not yet been validated. An AWS account is not required for the initial scope.

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

Return to this repository to configure monitoring. Installation commands will be added after the Helm chart version, values, and metrics integration have been implemented and tested. No monitoring setup script is available yet.

## Validation plan

1. Verify RabbitMQ exposes the required metrics, including queue-level labels.
2. Verify Prometheus discovers the intended targets and successfully scrapes them.
3. Compare dashboard values with RabbitMQ queue counts and Kubernetes replica state.
4. Generate a controlled workload using the application repository's load script.
5. Observe activation, scale-out, queue drain, and return to zero on the dashboard.
6. Introduce a controlled failure and confirm alert firing and resolution.
7. Repeat the scenario and record versions, configuration, timestamps, and evidence.

Record both ready and unacknowledged message counts at completion. Keep workloads separate and verify the queue is empty before comparable runs. Avoid rebuilding or changing worker configuration during a measurement.

## Proposed repository structure

```text
rabbitmq-observability-lab/
├── helm/
│   └── monitoring-values.yaml
├── monitoring/
│   ├── services/
│   ├── service-monitors/
│   ├── recording-rules/
│   └── alert-rules/
├── grafana/
│   └── dashboards/
├── scripts/
│   ├── setup.sh
│   └── cleanup.sh
├── tests/
├── docs/
│   ├── architecture.md
│   ├── operations.md
│   ├── experiments.md
│   ├── results.md
│   └── runbooks/
└── README.md
```

Directories and scripts will be added as their implementation is completed. Application source code and deployment manifests remain in the upstream KEDA lab.

## Implementation roadmap

- [ ] Record the application commit and validate its required metrics configuration.
- [ ] Select and pin a monitoring Helm chart version.
- [ ] Define lab resource requests, limits, and retention settings.
- [ ] Deploy the monitoring stack and validate target discovery.
- [ ] Implement RabbitMQ and Kubernetes scrape integration.
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

This is a local lab, not a production monitoring platform. High availability, remote metric storage, production retention sizing, external alert delivery, and multi-cluster operation are outside the initial scope.

A processing-time SLO requires application instrumentation and a clear definition of successful processing and its denominator. Queue depth alone cannot establish latency compliance. Later iterations can add message latency metrics, logs, traces, and SLO/error-budget analysis after these signals are implemented and validated.

No observability results are claimed yet. Completed experiments will document both successful behavior and limitations.
