# Observability experiments

## RabbitMQ queue collection — reported on 2026-10-14

### Objective

Verify that RabbitMQ exposes queue-specific metrics and that Prometheus collects the `tasks` queue during message submission.

### Environment and configuration

| Item | Value |
| --- | --- |
| Cluster context | `kind-freekubelab` |
| Kubernetes version | `1.35.1`, queried during chart selection |
| Selected monitoring chart | `kube-prometheus-stack` `92.3.0` |
| Monitoring release / namespace | `monitoring` / `monitoring` |
| Application namespace | `keda-rabbitmq-autoscaling` |
| RabbitMQ image | `rabbitmq:4.3.6-management`, from the supplied application manifest; running image not independently checked |
| Queue / virtual host | `tasks` / `/` |
| Metrics Service | `rabbitmq-metrics`, port and target port `15692` |
| Metrics endpoint | `/metrics/per-object` |
| ServiceMonitor | `monitoring/rabbitmq`, label `release: monitoring` |
| Scrape interval / timeout | `30s` / `10s` |
| Prometheus retention | `3d`; no persistent volume configured |
| Application commit | Not recorded |

Configuration sources: [Helm values](../helm/monitoring-values.yaml), [metrics Service](../monitoring/services/rabbitmq-metrics.yaml), and [ServiceMonitor](../monitoring/service-monitors/rabbitmq.yaml).

### Procedure and evidence

1. The operator confirmed the metrics endpoint responded through port-forward.
2. `/metrics` returned aggregated queue values without queue labels. `/metrics/per-object` returned the following queue-specific series before load:

```text
rabbitmq_queue_messages_ready{vhost="/",queue="tasks"} 0
rabbitmq_queue_messages_unacked{vhost="/",queue="tasks"} 0
rabbitmq_queue_consumers{vhost="/",queue="tasks"} 0
```

3. The reported pod listing showed Prometheus, Grafana, Alertmanager, Prometheus Operator, kube-state-metrics, and node-exporter containers ready and running.
4. A Service target-port typo (`15962`) caused connection refusal. The operator corrected the configuration and confirmed access to metrics.
5. The operator submitted messages using the application project and queried Prometheus:

```promql
rabbitmq_queue_messages_ready{queue="tasks",vhost="/"}
```

The screenshot supplied in the conversation showed one returned series with value **478**, including these labels:

```text
endpoint="metrics"
instance="10.244.1.6:15692"
job="rabbitmq-metrics"
namespace="keda-rabbitmq-autoscaling"
pod="rabbitmq-0"
queue="tasks"
service="rabbitmq-metrics"
vhost="/"
```

The screenshot is evidence supplied in the conversation; an image file has not been saved in this repository. The report date is not an exact scrape or workload timestamp.

### Result and limits

The returned series confirms the RabbitMQ → Service → ServiceMonitor → Prometheus collection path for ready messages on `tasks`. The value 478 is an instantaneous backlog measurement, not the total number of messages published or processed.

Unacknowledged messages and consumers were verified directly at the broker endpoint before load; their Prometheus time series during load were not captured. The evidence does not establish processing completion, delivery guarantees, sustained throughput, full KEDA scaling behavior, or alert behavior.

### Next measurements

- Record the application commit, actual deployed versions, load command, message count, and start/end timestamps with timezone.
- Preserve a screenshot or exported query results in the repository.
- Build and version a Grafana dashboard for ready messages, unacknowledged messages, connected consumers, and desired/available worker replicas.
- Record the full activation, scale-out, queue drain, and return-to-zero sequence; check both ready and unacknowledged messages at completion.
- Confirm plugin enablement survives RabbitMQ pod recreation.
- Validate failure scenarios and alert firing/resolution after rules and runbooks are implemented.
