---
title: "Sending OpenTelemetry from GKE to TrueWatch"
description: "GKE's new Managed OpenTelemetry only sends to Google. What we measured on GKE 1.37 Standard and Autopilot clusters, and the settings that get your OTLP data into TrueWatch through DataKit."
date: 2026-11-09 14:00:00 +0800
tags: [opentelemetry, gke, kubernetes, observability]
asset_path: /assets/img/posts/2026-11-09-opentelemetry-from-gke-to-truewatch
image:
  path: /assets/img/posts/2026-11-09-opentelemetry-from-gke-to-truewatch/two-roads.png
  alt: "Two roads from your app: GKE's managed collector delivers only to Google Cloud Observability; DataKit delivers to TrueWatch"
---

*Tested on GKE 1.37.1-gke.2097000 (Standard and Autopilot) with DataKit 2.18.0.*

Since 29 October 2026, GKE offers **Managed OpenTelemetry**: Google runs an OTLP collector inside
your cluster and can inject the OpenTelemetry settings into your Pods. If TrueWatch is where your
telemetry should go, two things matter.

## 1. GKE's managed OpenTelemetry does not send to TrueWatch

- The managed collector exports only to Google Cloud Observability (Cloud Trace, Cloud Monitoring,
  Cloud Logging). It has no setting for another destination and no option to send a copy elsewhere.
- Its injection resource (`Instrumentation`) can only point Pods at that managed collector. We tried
  to point it at DataKit; the cluster rejects it: *"endpoint must point to the in-cluster OTLP gRPC
  endpoint"*.
- If an `Instrumentation` resource matches a Pod, it **overwrites the Pod's own OTLP endpoint and
  protocol without any warning**. A Pod we configured for DataKit was rewritten to send to Google.

![A Pod configured to send gRPC to DataKit on port 4317 comes out of admission sending http/protobuf to the GKE managed collector on port 4318]({{ page.asset_path }}/silent-hijack.png)
_What a matching Instrumentation resource did to our test Pod._

**So: send to DataKit directly, and keep Google's automatic injection away from those workloads.**

Check whether any `Instrumentation` resource selects your workloads:

```bash
kubectl get instrumentations.telemetry.googleapis.com -A
```

If one matches a namespace or label you want in TrueWatch, remove that workload from its selector
(or delete the resource) and restart the Pods; the settings are injected only when a Pod is created.

## 2. How to send to TrueWatch

### Step 1: DataKit with the OpenTelemetry input open to the cluster

Install DataKit as a DaemonSet (the standard Kubernetes install), then make two changes in its
container environment:

```yaml
- name: ENV_DEFAULT_ENABLED_INPUTS
  value: dk,container,opentelemetry          # keep your existing list and add opentelemetry
- name: ENV_INPUT_OTEL_GRPC
  value: '{"addr":"0.0.0.0:4317","max_payload":16777216}'
```

The second line matters: by default the OpenTelemetry gRPC receiver listens on `127.0.0.1:4317`,
so other Pods get *connection refused* even though the DataKit Service exposes port 4317.
(HTTP on port 9529 works without this change.)

On GKE's Container-Optimized OS nodes the root file system is read-only. If the DataKit Pods stay in
`CreateContainerError: mkdir /root/datakit_cache: read-only file system`, change that hostPath in
the manifest to a path under `/var/lib`, for example `/var/lib/datakit_cache`.

### On GKE Autopilot

Autopilot rejects the standard DataKit DaemonSet: it uses host namespaces, privileged mode and
writable hostPath volumes, which Autopilot only allows for vendors on its partner allowlist. In our
test the admission was refused under three policies (`autogke-disallow-hostnamespaces`,
`autogke-disallow-privilege`, `autogke-no-write-mode-hostpath`).

Use the Autopilot manifest from the DataKit documentation instead (`datakit-gke-autopilot.yaml`, a
Deployment), with the inputs reduced to what receiving OTLP needs:

```yaml
- name: ENV_DEFAULT_ENABLED_INPUTS
  value: dk,opentelemetry
- name: ENV_INPUT_OTEL_GRPC
  value: '{"addr":"0.0.0.0:4317","max_payload":16777216}'
```

It was admitted without warnings and received 100/100 spans and 50/50 logs over both gRPC and HTTP.
The trade-off: this DataKit cannot see the nodes, so there is no host metrics or container file-log
collection, and there is one DataKit for the whole cluster. For receiving OTLP that your
applications send, that is enough.

### Step 2: point your applications at DataKit

gRPC (recommended):

```yaml
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: http://datakit-service.datakit.svc.cluster.local:4317
- name: OTEL_EXPORTER_OTLP_PROTOCOL
  value: grpc
```

HTTP alternative: DataKit uses its own paths on port 9529 and accepts protobuf only:

```yaml
- name: OTEL_EXPORTER_OTLP_PROTOCOL
  value: http/protobuf
- name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
  value: http://datakit-service.datakit.svc.cluster.local:9529/otel/v1/traces
- name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
  value: http://datakit-service.datakit.svc.cluster.local:9529/otel/v1/metrics
- name: OTEL_EXPORTER_OTLP_LOGS_ENDPOINT
  value: http://datakit-service.datakit.svc.cluster.local:9529/otel/v1/logs
```

Do not copy Google's port 4318, the standard `/v1/traces` paths, or `http/json`: in our test they
timed out, returned 404 and returned 400 respectively.

Also set `service.name` and the Kubernetes resource attributes (for example through
`OTEL_RESOURCE_ATTRIBUTES` and the downward API), so you can find the data by service and Pod.

### Step 3: verify

Send a known number of spans and logs, then count them in TrueWatch. In our test every route
delivered N of N (100/100 spans, 50/50 logs). When you query, note how the resource attributes are
named in TrueWatch:

| OpenTelemetry attribute | TrueWatch field |
|---|---|
| `service.name` | `service` |
| `k8s.pod.name` | `k8s_pod_name` (`pod_name` stays empty) |
| `k8s.namespace.name` | `k8s_namespace_name` |
| `deployment.environment.name` | `env` on spans |

## Note: if you also want the data in Google

Google's managed collector cannot send a copy to TrueWatch, but you can send a copy to Google.
Run one OpenTelemetry Collector of your own in front of both, point your applications at it
instead of at DataKit, and give it one exporter per destination. This is the configuration we
tested (Google-Built OpenTelemetry Collector `otelcol-google:0.163.0`):

```yaml
receivers:
  otlp:
    protocols:
      grpc: {endpoint: "0.0.0.0:4317"}
      http: {endpoint: "0.0.0.0:4318"}
processors:
  memory_limiter: {check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 20}
  batch: {}
exporters:
  otlp_grpc/datakit:
    endpoint: datakit-service.datakit.svc.cluster.local:4317
    tls: {insecure: true}
    sending_queue: {enabled: true}
  otlp_grpc/google:
    endpoint: opentelemetry-collector.gke-managed-otel.svc.cluster.local:4317
    tls: {insecure: true}
    sending_queue: {enabled: true}
service:
  pipelines:
    traces:  {receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp_grpc/datakit, otlp_grpc/google]}
    metrics: {receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp_grpc/datakit, otlp_grpc/google]}
    logs:    {receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp_grpc/datakit, otlp_grpc/google]}
```

What we measured with it: both sides received 100/100 spans and 50/50 logs, and 24,002 spans plus
6,001 logs in a short volume run. Each exporter has its own queue, so when DataKit was down for
55 seconds Google kept receiving, and TrueWatch caught up once DataKit was back.

Two things to watch on the Google side:

- **Several points of one metric series in one export are dropped.** Google's managed collector kept
  1 of 30 such points and logged *"Duplicate TimeSeries"* only in its own log; the sender saw
  success. Sent one point at a time, it kept 11 of 12. SDKs that export each series once per
  interval should not hit this (not tested with a real SDK).
- **Google attributes the data to the collector's Pod**, not the application's: Pod UID,
  Deployment and ReplicaSet are the collector's. Adding the `k8sattributes` processor to this
  collector should fix it; we did not test that.

Neither point affects what TrueWatch receives about the Pod. (TrueWatch stored 29 of the 30 batched points; the cause is not determined, and one point at a time arrived 12 of 12.)

## Not covered

- Real application SDKs were represented by the OpenTelemetry `telemetrygen` tool.
- The DataKit Operator's own OpenTelemetry injection was not tested together with Google's.
- Sampling was not tested.
