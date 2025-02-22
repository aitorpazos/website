---
title: Traces For Kubernetes System Components
reviewers:
- logicalhan
- lilic
content_type: concept
weight: 90
---

<!-- overview -->

{{< feature-state for_k8s_version="v1.22" state="alpha" >}}

System component traces record the latency of and relationships between operations in the cluster.

Kubernetes components emit traces using the
[OpenTelemetry Protocol](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/protocol/otlp.md#opentelemetry-protocol-specification)
with the gRPC exporter and can be collected and routed to tracing backends using an
[OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector#-opentelemetry-collector).

<!-- body -->

## Trace Collection

For a complete guide to collecting traces and using the collector, see
[Getting Started with the OpenTelemetry Collector](https://opentelemetry.io/docs/collector/getting-started/).
However, there are a few things to note that are specific to Kubernetes components.

By default, Kubernetes components export traces using the grpc exporter for OTLP on the
[IANA OpenTelemetry port](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=opentelemetry), 4317.
As an example, if the collector is running as a sidecar to a Kubernetes component,
the following receiver configuration will collect spans and log them to standard output:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
exporters:
  # Replace this exporter with the exporter for your backend
  logging:
    logLevel: debug
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [logging]
```

## Component traces

### kube-apiserver traces

The kube-apiserver generates spans for incoming HTTP requests, and for outgoing requests
to webhooks, etcd, and re-entrant requests. It propagates the
[W3C Trace Context](https://www.w3.org/TR/trace-context/) with outgoing requests
but does not make use of the trace context attached to incoming requests,
as the kube-apiserver is often a public endpoint.

#### Enabling tracing in the kube-apiserver

To enable tracing, enable the `APIServerTracing`
[feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
on the kube-apiserver. Also, provide the kube-apiserver with a tracing configuration file
with `--tracing-config-file=<path-to-config>`. This is an example config that records
spans for 1 in 10000 requests, and uses the default OpenTelemetry endpoint:

```yaml
apiVersion: apiserver.config.k8s.io/v1alpha1
kind: TracingConfiguration
# default value
#endpoint: localhost:4317
samplingRatePerMillion: 100
```

For more information about the `TracingConfiguration` struct, see
[API server config API (v1alpha1)](/docs/reference/config-api/apiserver-config.v1alpha1/#apiserver-k8s-io-v1alpha1-TracingConfiguration).

### kubelet traces

{{< feature-state for_k8s_version="v1.25" state="alpha" >}}

The kubelet CRI interface and authenticated http servers are instrumented to generate
trace spans. As with the apiserver, the endpoint and sampling rate are configurable.
Trace context propagation is also configured. A parent span's sampling decision is always respected.
A provided tracing configuration sampling rate will apply to spans without a parent.
Enabled without a configured endpoint, the default OpenTelemetry Collector receiver address of "localhost:4317" is set.

#### Enabling tracing in the kubelet

To enable tracing, enable the `KubeletTracing`
[feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
on the kubelet. Also, provide the kubelet with a
[tracing configuration](https://github.com/kubernetes/component-base/blob/release-1.25/tracing/api/v1/types.go).
This is an example snippet of a kubelet config that records spans for 1 in 10000 requests, and uses the default OpenTelemetry endpoint:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
featureGates:
  KubeletTracing: true
tracing:
  # default value
  #endpoint: localhost:4317
  samplingRatePerMillion: 100
```

## Stability

Tracing instrumentation is still under active development, and may change
in a variety of ways. This includes span names, attached attributes,
instrumented endpoints, etc. Until this feature graduates to stable,
there are no guarantees of backwards compatibility for tracing instrumentation.

## {{% heading "whatsnext" %}}

* Read about [Getting Started with the OpenTelemetry Collector](https://opentelemetry.io/docs/collector/getting-started/)

