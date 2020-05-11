# Prometheus-exporter Operator

[![build status](https://circleci.com/gh/3scale/prometheus-exporter-operator.svg?style=shield)](https://circleci.com/gh/3scale/prometheus-exporter-operator)
[![release](https://badgen.net/github/release/3scale/prometheus-exporter-operator)](https://github.com/3scale/prometheus-exporter-operator/releases)
[![license](https://badgen.net/github/license/3scale/prometheus-exporter-operator)](https://github.com/3scale/prometheus-exporter-operator/blob/master/LICENSE)

A Kubernetes Operator based on the Operator SDK (`ansible` version) to centralize the setup of 3rd party prometheus exporters on **Kubernetes/OpenShift**.

Current prometheus exporters `types` supported, managed by same prometheus-exporter operator:
* memcached
* redis
* mysql
* postgresql
* sphinx
* es (elasticsearch)
* cloudwatch

The operator manages, for each CR, the lifecycle of the following objects:
* Deployment
* Service
* ServiceMonitor (optional, enabled by default `serviceMonitor.enabled: true`)

In addition, the operator manages, for each CR, a GrafanaDashboard (optional, enabled by default `grafanaDashboard.enabled: true`), but in reality it manages a single dashboard type per Namespace (not per CR), so:
* If you deploy for example different redis CRs, and you want to have the redis dashboard created, you need to enabled it on every redis CR with the same grafana-operator label selector (but in reality it will just manage a single dashboard per Namespace shared accross all CRs from the same type)
* You can deploy the prometheus-exporter-operator with different operator versions on different Namespaces, so it will create separate dashboards per Namespace (they won't collision, that's why dashboard name includes the Namespace)
* All grafana dashboards are preconfigured to use `CR_NAME` as the filter of all possible dashboards of every type (for example `staging-system-memcached`)

> **NOTE**
><br /> Some exporters need some **extra objects to be previously manually created** in order to work (**manual objects names need to be specified on required CR fields**). This extra needed objects includes **Secrets (credentials) or Configmaps (configuration files) on specific formats**. Examples to help you create these extra objects are provided on [examples](examples/) directory for all exporter types.

## Requirements

* [prometheus-operator](https://github.com/coreos/prometheus-operator) v0.17.0+
* [grafana-operator](https://github.com/integr8ly/grafana-operator) v3.0.0+

## Getting Started

### Operator image

* Apply changes on Operator ([ansible role](roles/prometheusexporter/)), then create a new operator image and push it to the registry with:
```bash
$ make operator-image-update
```
* Operator images are available [here](https://quay.io/repository/3scale/prometheus-exporter-operator?tab=tags)

### Operator deploy

* Deploy operator (Namespace, CRD, operator objects):
```bash
$ make operator-create
```
* Create any `PrometheusExporter` resource type (you can find examples on [examples](examples/) directory).
* Once tested, delete created operator objects (except CRD/Namespace for caution):
```bash
$ make operator-delete
```

## PrometheusExporter CustomResource

### Full CR Example

Most of the fields do not need to be specified (can use default values), this is just an example of everything that can be overriden:

```yaml
apiVersion: ops.3scale.net/v1alpha1
kind: PrometheusExporter
metadata:
  name: staging-system-memcached
spec:
  type: memcached
  serviceMonitor:
    enabled: true
    interval: 45s
  grafanaDashboard:
    enabled: true
    label:
      key: monitoring-key
      value: middleware
  extraLabel:
    key: tier
    value: frontend
  dbHost: system-memcache
  dbPort: 11211
  image:
    name: prom/memcached-exporter
    version: v0.6.0
  port: 9150
  livenessProbe:
    timeoutSeconds: 10
    periodSeconds: 20
    successThreshold: 1
    failureThreshold: 7
  readinessProbe:
    timeoutSeconds: 10
    periodSeconds: 20
    successThreshold: 1
    failureThreshold: 7
  resources:
    requests:
      cpu: 75m
      memory: 64Mi
    limits:
      cpu: 150m
      memory: 128Mi
```

### CR Spec common

| **Field** | **Type** | **Required** | **Default value (depends on type)** | **Description** |
|:---:|:---:|:---:|:---:|:---:|
| `type` | `string` | Yes | `none` | Possible prometheus-exporter types: `memcached`, `redis`, `mysql`, `postgresql`, `sphinx`, `es`, `cloudwatch` |
| `serviceMonitor.enabled` | `bool` | No | `true` | Create (`true`) or not (`false`) ServiceMonitor object |
| `serviceMonitor.interval` | `string` | No | `30s` | Prometheus scrape interval |
| `grafanaDashboard.enabled` | `bool` | No | `true` | Create (`true`) or not (`false`) GrafanaDashboard object |
| `grafanaDashboard.label.key` | `string` | No | discovery | Label `key` used by grafana-operator for dashboard discovery |
| `grafanaDashboard.label.value` | `string` | No | enabled | Label `value` used by grafana-operator for dashboard discovery |
| `extraLabel.key` | `string` | No | - | Add extra label `key` to all created resources (example `tier`) |
| `extraLabel.value` | `string` | No | - | Add extra label `value` to all created resources (example `frontend`) |
| `image.name` | `string` | No | Depends on exporter | Prometheus exporter image name |
| `image.version` | `string` | No | Depends on exporter | Prometheus exporter image version |
| `port` | `int` | No | Depends on exporter | Prometheus exporter metrics port |
| `resources.requests.cpu` | `string` | No | `25m` | Override CPU requests |
| `resources.requests.memory` | `string` | No | `32Mi` | Override Memory requests |
| `resources.limits.cpu` | `string` | No | `50m` | Override CPU limits |
| `resources.limits.memory` | `string` | No | `64Mi` | Override Memory limits |
| `livenessProbe.timeoutSeconds` | `int` | No | `3` | Override liveness timeout (seconds) |
| `livenessProbe.periodSeconds` | `int` | No | `15` | Override liveness period (seconds) |
| `livenessProbe.successThreshold` | `int` | No | `1` | Override liveness success threshold |
| `livenessProbe.failureThreshold` | `int` | No | `5` | Override liveness failure threshold |
| `readinessProbe.timeoutSeconds` | `int` | No | `3` | Override readiness timeout (seconds) |
| `readinessProbe.periodSeconds` | `int` | No | `30` | Override readiness period (seconds) |
| `readinessProbe.successThreshold` | `int` | No | `1` | Override readiness success threshold |
| `readinessProbe.failureThreshold` | `int` | No | `5` | Override readiness failure threshold |

### CR Spec custom

Specific CR fields per exporter type, extra objects needed, database permissions... can be found on [examples](examples/) directory.

## Prometheus Rules

PrometheusRules management is not included inside the operator, because it depends on:
* What you need to monitor (maybe ones just need basic cpu/mem alerts, while others may be interested on specific alerts checking internals of a database)
* Why you want to be paged (severity warning/critical, minutes duration before firing an alert...)
* Customizable thresholds definition (it is something that depends on infrastructure dimensions...)

However, some examples of prometheus rules can be found on [prometheus-rules](prometheus-rules/) directory.
* Create all example Prometheus Rules (General, Memcached, Redis, MySQL, PostgreSQL, Sphinx, ElasticSearch, Cloudwatch):
```bash
$ make prometheus-rules-create:
```
* Once tested, delete created rules:
```bash
$ make prometheus-rules-delete
```

## License

Prometheus-exporter operator is under Apache 2.0 license. See the [LICENSE](LICENSE) file for details.
