# grafana-kubernetes

It's a collection of a few solution for Grafana - Kubernetes integration.

## Presenting info from Kubernetes API

It's possible to access the Kubernetes API trough <https://grafana.com/grafana/plugins/marcusolsson-json-datasource/>, but access token to Kubernetes API must be set in Grafana, which is not too secure.
Instead, a kubectl proxy can run in same namespace with read-only access, limited to a set of resources.

Grafana can present Loki logs, but if Loki (Promtail) is not deployed or Loki does not configured to collect logs from all Pods, Grafana can show Pod logs trough JSON API datasource, too. There is one trick: an nginx container transforms lines to JSON array by a simple script.

Table panel can have external link, which is configured on Pod listing panels to access an external Kubernetes dashboard.

The example setup looks like:

```text
     ┌──────────────────────────────────────────┐     ┌──────────────────────┐
     │                                          │     │                      │
     │                 Grafana                  │     │     Adapter Pod      │
     │                                          │     │                      │
     │   ┌──────────────┐    ┌──────────────┐   │     │   ┌──────────────┐   │
     │   │              │    │              │   │     │   │              │   │
     │   │  Pod info:   │    │   JSON API   │   │     │   │   kubectl    │   │
     │   │ Table, Stat  │◄───┤   datasurce  │◄──┼─────┼───┤     ext      │   │
┌────┼───┤ panel plugin │    │    plugin    │   │     │   │              │   │
│    │   │              │    │              │   │     │   │              │   │
│    │   └──────────────┘    └──────────────┘   │     │   └──────▲───────┘   │
│    │                                          │     │          │           │
│    │   ┌──────────────┐    ┌──────────────┐   │     │   ┌──────┴───────┐   │     ┌──────────────┐
│    │   │              │    │              │   │     │   │              │   │     │              │
│    │   │  Pod info:   │    │   JSON API   │   │     │   │   kubectl    │   │     │  Kubernetes  │
│    │   │ Table, Stat  │◄───┤   datasurce  │◄──┼─────┼───┤    proxy     │◄──┼─────┤     API      │
├────┼───┤ panel plugin │    │    plugin    │   │     │   │              │   │     │              │
│    │   │              │  ┌─┤              │   │     │   │              │   │     │              │
│    │   └──────────────┘  │ └──────────────┘   │     │   └──────────────┘   │     └──────┬───────┘
│    │                     │                    │     │                      │            │
│    │   ┌──────────────┐  │ ┌──────────────┐   │     │   ┌──────────────┐   │            │
│    │   │              │◄─┘ │              │   │     │   │              │   │            │
│    │   │  Pod logs:   │    │   JSON API   │   │     │   │    nginx,    │   │            │
│    │   │    Table     │◄───┤   datasurce  │◄──┼─────┼───┤   lines to   │◄──┼────────────┘
│    │   │ panel plugin │    │    plugin    │   │     │   │     JSON     │   │
│    │   │              │    │              │   │     │   │              │   │
│    │   └──────────────┘    └──────────────┘   │     │   └──────────────┘   │
│    │           ▲                              │     │                      │
│    │           │                              │     └──────────────────────┘
│    │           │                              │
│    │           │                              │
│    │           │                              │
│    │           │                              │     ┌──────────────────────┐
│    │           │                              │     │                      │
│    │           │                              │     │      Loki Pod        │
│    │           │                              │     │                      │
│    │           │           ┌──────────────┐   │     │   ┌──────────────┐   │
│    │           │           │              │   │     │   │              │   │
│    │           │           │     Loki     │   │     │   │              │   │
│    │           └───────────┤   datasurce  │◄──┼─────┼───┤    Loki      │   │
│    │                       │    plugin    │   │     │   │              │   │
│    │                       │              │   │     │   │              │   │
│    │                       └──────────────┘   │     │   └──────────────┘   │
│    │                                          │     │                      │
│    └──────────────────────────────────────────┘     └──────────────────────┘
│
│
│                ┌──────────────┐
│                │              │
│     link to    │    Kiali     │
└───────────────►│   or other   │
                 │   dashboard  │
                 │              │
                 └──────────────┘
```

There are several options to run a demo Kubernetes on laptop, see more details here: [Environment for comparing several on-premise Kubernetes distributions (K3s, MicroK8s, KinD, kubeadm)](https://faun.pub/environment-for-comparing-several-on-premise-kubernetes-distributions-k3s-kind-kubeadm-a53675a80a00). The `make all` command deploys JSON API datasource plugin with configuration. The [pgillich-tree-panel](https://github.com/pgillich/grafana-tree-panel) plugin is also included.

## Adapter Pod

Deployment files of Adapter Pod can be found in [kubernetes/monitoring](kubernetes/monitoring) directory. These files can be deployed by `kubectl apply -k kubernetes/monitoring` command.

The `kubect proxy` is configured to deny any modification requests, see at `kubectl-proxy-deployment.yaml`. Other access limitations are configured in `kubectl-proxy-clusterRole.yaml`.

A simple LUA code transforms the raw log lines to JSON array, see `kubectl-proxy-openresty-config.yaml`.

Example requests to the adapter pod from the Grafana container:

```sh
export GRAFANA_POD=$(kubectl get pod -n monitoring -l 'app.kubernetes.io/name=grafana' -o name | sed 's#^pod/##g');

kubectl exec -n monitoring ${GRAFANA_POD} -c grafana -- /bin/sh -c 'wget -q -O - http://kubectl-proxy/api/v1/namespaces/monitoring/pods'

kubectl exec -n monitoring ${GRAFANA_POD} -c grafana -- /bin/sh -c 'wget -q -O - http://kubectl-proxy:8002/api/v1/namespaces/monitoring/pods/alertmanager-prometheus-st
ack-kube-prom-alertmanager-0/log?container=alertmanager'

kubectl exec -n monitoring ${GRAFANA_POD} -c grafana -- /bin/sh -c 'wget -q -O - http://kubectl-proxy:8003/api/v1/namespaces/monitoring/pods'
```

## Datasource configuration

Example configurations:

![K8s API](images/datasource_k8s.jpg)

![K8s log API](images/datasource_k8s_log.jpg)

## Sample Grafana Dashboards

Raw Kubernetes responses should be transformed to a flat JSON array, substituting the missing values. Fortunately, JSON API datasource supports JSONata. Example JSONata expression for transformation and substitution:

```jsonata
$map(items, function($v) {{"namespace": $v.metadata.namespace, "name": $v.metadata.name, "appName": $v.metadata.labels."app.kubernetes.io/name" ? $v.metadata.labels."app.kubernetes.io/name" : "-", "statusPhase": $v.status.phase, "containerCount": $count($v.spec.containers), "containerImage": $join($v.spec.containers[*].image, " "), "containerState": $v.status.containerStatuses ? $string($v.status.containerStatuses[*].state) : "-"}})
```

JSONAta expressions can be evaluated by `jfq`. It can be installed by below command:

```sh
npm install --global jfq
```

Example usage:

```sh
jfq '$filter($each(items{status.phase: $count(status.phase)}, function($v, $k) {{"status": $k, "count": $v}}), function($v) {$v.status="Running" or $v.status="Succeeded"})' /tmp/pods.json
```

Importable Grafana Dashboard files can be found in [grafana/dashboards](grafana/dashboards) directory. The `*(ext).json` dashboards use `kubectl-ext` container as a data source.

### Namespace and All Pods

Pod table panels are configured with external links, which uses the external server URL from a Dashboard variable. Example screenshot with link to Kiali Dashboard:

![Namespace Pods](images/namespace_pod.jpg)

### Flux

Flux does not have maintained UI, but Grafana can be used to show the current state, for example:

![Flux Helm status](images/flux.jpg)

### Logs

Example sceenshot:

![Logs](images/logs.jpg)
