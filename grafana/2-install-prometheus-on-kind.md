# Introduction

Following the [guide](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/configure-infrastructure-manually/prometheus/prometheus-operator/) provided on grafana.com

> Prometheus Operator implements the Kubernetes Operator pattern for managing a Prometheus-based Kubernetes monitoring stack. A Kubernetes Operator consists of Kubernetes custom resources and Kubernetes controller code. Together, these abstract away the management and implementation details of running a given service on Kubernetes.

This looks up my alley. Let's try it out.

## What I will do differently

The guide states I need a Grafana Cloud account. I wouldn't want to use one, I'd like to just use prometheus with my local grafana installation. So I'll probably need to change some steps along the way, applying common sense.

## Installation

### Creating CRDs and operator

The first installation steps tells me to 
```
kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
```

Of course, before doing that, I'm going to have a peek inside.
```
$ wget https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml -O prometheus_operator_bundle.yaml
--2024-07-21 15:21:50--  https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.110.133, 185.199.109.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3417298 (3,3M) [text/plain]
Saving to: ‘prometheus_operator_bundle.yaml’

prometheus_operator_bundle.yaml               100%[=================================================================================================>]   3,26M   616KB/s    in 5,0s    

2024-07-21 15:21:56 (670 KB/s) - ‘prometheus_operator_bundle.yaml’ saved [3417298/3417298]
```

3,3M? What is in that thing??
```
$ cat prometheus_operator_bundle.yaml | grep '^kind:' | sort | uniq -c
      1 kind: ClusterRole
      1 kind: ClusterRoleBinding
     10 kind: CustomResourceDefinition
      1 kind: Deployment
      1 kind: Service
      1 kind: ServiceAccount

```

Apparently, a bunch of pretty huge 'CRD's and little else. Of course, there are the expected items like a ClusterRole and a binding for it, a Deployment (probably will run the related controllers for the CRDs), and a Service and ServiceAccount.

I'll want to have my prometheus stuff in a dedicated namespace, so I'll create one and apply the bundle there:
```
$ kubectl create ns sven-prometheus
namespace/sven-prometheus created

$ kubectl create -f prometheus_operator_bundle.yaml -n sven-prometheus
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusagents.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/scrapeconfigs.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
the namespace from the provided object "default" does not match the namespace "sven-prometheus". You must pass '--namespace=default' to perform this operation.
the namespace from the provided object "default" does not match the namespace "sven-prometheus". You must pass '--namespace=default' to perform this operation.
the namespace from the provided object "default" does not match the namespace "sven-prometheus". You must pass '--namespace=default' to perform this operation.
```

Note - if I use `kubectl apply` instead of `kubectl create`, kubernetes complains that some of the CRD annotations are too big. `kubectl create` works fine though. Also, I can use the `--server-side=true` flag to get `kubectl apply` to work.

What about the error though?
```
the namespace from the provided object "default" does not match the namespace "sven-prometheus". You must pass '--namespace=default' to perform this operation.
```

A few of the resources specified in the yaml manifest explicitly declare the `default` namespaces:
```
$ cat prometheus_operator_bundle.yaml | grep "  namespace: default" | wc -l
4
```

I still want to have my prometheus related resources in their own namespace, so I'll try changing these entries:

```
$ kubectl delete -n sven-prometheus -f prometheus_operator_bundle.yaml
<...>
$ sed -i 's/  namespace: default/  namespace: sven-prometheus/g' ./prometheus_operator_bundle.yaml
$ kubectl apply -n sven-prometheus -f prometheus_operator_bundle.yaml --server-side=true
customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheusagents.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/scrapeconfigs.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com serverside-applied
customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com serverside-applied
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator serverside-applied
clusterrole.rbac.authorization.k8s.io/prometheus-operator serverside-applied
deployment.apps/prometheus-operator serverside-applied
serviceaccount/prometheus-operator serverside-applied
service/prometheus-operator serverside-applied

$ kubectl get deploy -n sven-prometheus
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
prometheus-operator   1/1     1            1           5m37s
```

I also have a quick look at the deployment's logs:
```
$ kubectl logs deploy/prometheus-operator -n sven-prometheus
level=info ts=2024-07-21T12:53:50.480425757Z caller=main.go:205 msg="Starting Prometheus Operator" version="(version=0.75.1, branch=refs/tags/v0.75.1, revision=9b72f8549d166fd40c9dde5f051d6324c5ef4b60)"
level=info ts=2024-07-21T12:53:50.48060218Z caller=main.go:206 build_context="(go=go1.22.4, platform=linux/amd64, user=Action-Run-ID-9759554202, date=20240702-10:46:25, tags=unknown)"
level=info ts=2024-07-21T12:53:50.480795207Z caller=main.go:207 feature_gates="PrometheusAgentDaemonSet=false"
level=warn ts=2024-07-21T12:53:50.481634491Z caller=cpu.go:32 msg="Failed to set GOMAXPROCS automatically" err="path \"/docker/6b11e1c0d807c020975ab8987ef444566c9111bdc863bddfaf5ce8c8c1b3d415\" is not a descendant of mount point root \"/docker/6b11e1c0d807c020975ab8987ef444566c9111bdc863bddfaf5ce8c8c1b3d415/kubelet\" and cannot be exposed from \"/sys/fs/cgroup/misc/kubelet\""
level=info ts=2024-07-21T12:53:50.481699692Z caller=main.go:220 msg="namespaces filtering configuration " config="{allow_list=\"\",deny_list=\"\",prometheus_allow_list=\"\",alertmanager_allow_list=\"\",alertmanagerconfig_allow_list=\"\",thanosruler_allow_list=\"\"}"
level=info ts=2024-07-21T12:53:50.591666471Z caller=main.go:254 msg="connection established" cluster-version=v1.25.2
level=warn ts=2024-07-21T12:53:50.601934451Z caller=main.go:90 msg="missing permission on resource \"storageclasses\" (group: \"storage.k8s.io/v1\")" reason="missing \"get\" permission on resource \"storageclasses\" (group: \"storage.k8s.io\") for all namespaces"
level=warn ts=2024-07-21T12:53:50.680053485Z caller=main.go:296 msg="missing permission to emit events" reason="missing \"create\" permission on resource \"events\" (group: \"\") for all namespaces"
level=warn ts=2024-07-21T12:53:50.680149009Z caller=main.go:296 msg="missing permission to emit events" reason="missing \"patch\" permission on resource \"events\" (group: \"\") for all namespaces"
level=warn ts=2024-07-21T12:53:50.682563955Z caller=main.go:79 msg="resource \"scrapeconfigs\" (group: \"monitoring.coreos.com/v1alpha1\") not installed in the cluster"
level=warn ts=2024-07-21T12:53:50.684917544Z caller=main.go:79 msg="resource \"prometheuses\" (group: \"monitoring.coreos.com/v1\") not installed in the cluster"
level=warn ts=2024-07-21T12:53:50.687197065Z caller=main.go:79 msg="resource \"prometheusagents\" (group: \"monitoring.coreos.com/v1alpha1\") not installed in the cluster"
level=warn ts=2024-07-21T12:53:50.689477612Z caller=main.go:79 msg="resource \"alertmanagers\" (group: \"monitoring.coreos.com/v1\") not installed in the cluster"
level=warn ts=2024-07-21T12:53:50.691601389Z caller=main.go:79 msg="resource \"thanosrulers\" (group: \"monitoring.coreos.com/v1\") not installed in the cluster"
level=info ts=2024-07-21T12:53:50.773302303Z caller=server.go:298 msg="starting insecure server" address=[::]:8080
level=error ts=2024-07-21T12:53:50.778410515Z caller=controller.go:189 component=kubelet_endpoints kubelet_object=kube-system/kubelet msg="Failed to synchronize nodes" err="listing nodes failed: nodes is forbidden: User \"system:serviceaccount:sven-prometheus:prometheus-operator\" cannot list resource \"nodes\" in API group \"\" at the cluster scope"

```

This looks slightly worrying - it may be due to my initial failure to create all obects, though. What if I delete the pod
```
$ kubectl get pods -n sven-prometheus
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-6b88448777-nr5w4   1/1     Running   0          16m

$ kubectl delete pod prometheus-operator-6b88448777-nr5w4 -n sven-prometheus
pod "prometheus-operator-6b88448777-nr5w4" deleted

$ kubectl get pods -n sven-prometheus
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-6b88448777-bgh5m   1/1     Running   0          32s

$ kubectl logs deploy/prometheus-operator -n sven-prometheus
level=info ts=2024-07-21T13:09:52.179927037Z caller=main.go:205 msg="Starting Prometheus Operator" version="(version=0.75.1, branch=refs/tags/v0.75.1, revision=9b72f8549d166fd40c9dde5f051d6324c5ef4b60)"
level=info ts=2024-07-21T13:09:52.180130947Z caller=main.go:206 build_context="(go=go1.22.4, platform=linux/amd64, user=Action-Run-ID-9759554202, date=20240702-10:46:25, tags=unknown)"
level=info ts=2024-07-21T13:09:52.180203488Z caller=main.go:207 feature_gates="PrometheusAgentDaemonSet=false"
level=warn ts=2024-07-21T13:09:52.18399878Z caller=cpu.go:32 msg="Failed to set GOMAXPROCS automatically" err="path \"/docker/6b11e1c0d807c020975ab8987ef444566c9111bdc863bddfaf5ce8c8c1b3d415\" is not a descendant of mount point root \"/docker/6b11e1c0d807c020975ab8987ef444566c9111bdc863bddfaf5ce8c8c1b3d415/kubelet\" and cannot be exposed from \"/sys/fs/cgroup/misc/kubelet\""
level=info ts=2024-07-21T13:09:52.184246935Z caller=main.go:220 msg="namespaces filtering configuration " config="{allow_list=\"\",deny_list=\"\",prometheus_allow_list=\"\",alertmanager_allow_list=\"\",alertmanagerconfig_allow_list=\"\",thanosruler_allow_list=\"\"}"
level=info ts=2024-07-21T13:09:52.281484846Z caller=main.go:254 msg="connection established" cluster-version=v1.25.2
level=info ts=2024-07-21T13:09:52.485994519Z caller=operator.go:361 component=prometheus-controller msg="Kubernetes API capabilities" endpointslices=true
level=info ts=2024-07-21T13:09:52.496643794Z caller=operator.go:324 component=prometheusagent-controller msg="Kubernetes API capabilities" endpointslices=true
level=info ts=2024-07-21T13:09:52.873953695Z caller=server.go:298 msg="starting insecure server" address=[::]:8080
level=info ts=2024-07-21T13:09:53.97880085Z caller=operator.go:418 component=prometheus-controller msg="successfully synced all caches"
level=info ts=2024-07-21T13:09:54.075848483Z caller=operator.go:313 component=alertmanager-controller msg="successfully synced all caches"
level=info ts=2024-07-21T13:09:54.076937636Z caller=operator.go:283 component=thanos-controller msg="successfully synced all caches"
level=info ts=2024-07-21T13:09:54.079973887Z caller=operator.go:433 component=prometheusagent-controller msg="successfully synced all caches"
```

This looks much better! I'll ignore the `GOMAXPROCS` warning and continue.

### Configuring RBAC

The proposed `prom_rbac.yaml` manifest again includes a reference to a `ServiceAccount` (`prometheus`, created in the same file) in the `default` namespace. I'll change this to refer to my own namespace, then apply:

```
$ sed -i 's/  namespace: default/  namespace: sven-prometheus/g' ./prom_rbac.yaml

$ kubectl apply -f prom_rbac.yaml -n sven-prometheus
serviceaccount/prometheus unchanged
resource mapping not found for name: "prometheus" namespace: "" from "prom_rbac.yaml": no matches for kind "ClusterRole" in version "rbac.authorization.k8s.io/v1beta1"
ensure CRDs are installed first
resource mapping not found for name: "prometheus" namespace: "" from "prom_rbac.yaml": no matches for kind "ClusterRoleBinding" in version "rbac.authorization.k8s.io/v1beta1"
ensure CRDs are installed first
```

That doesn't seem right; let's see what apiVersions are available for `rbac.authorization.k8s.io` and what's in the manifest yaml:
```
$ kubectl api-versions | grep 'rbac.authorization.k8s.io'
rbac.authorization.k8s.io/v1

$ cat prom_rbac.yaml | grep 'apiVersion: rbac.authorization.k8s.io' | sort | uniq
apiVersion: rbac.authorization.k8s.io/v1beta1
```

I probably just need to change that apiVersion to what's available on my system:
```
$ sed -i 's/rbac.authorization.k8s.io\/v1beta1/rbac.authorization.k8s.io\/v1/g' prom_rbac.yaml

$ kubectl apply -f prom_rbac.yaml -n sven-prometheus
serviceaccount/prometheus unchanged
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
```

Now it worked.

### Deploying Prometheus

The next [step](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/configure-infrastructure-manually/prometheus/prometheus-operator/#deploy-prometheus) is the creation of a 2-replica prometheus instance.

```
$ kubectl apply -f prometheus.yaml -n sven-prometheus
prometheus.monitoring.coreos.com/prometheus created
```

The guide states that the manifest
> Instructs Prometheus to automatically pick up all configured ServiceMonitor resources using {}. You’ll create a ServiceMonitor to get Prometheus to scrape its own metrics.

This actually uses the `Prometheus` CRD that was created earlier. The related controller probably creates a pod somewhere. Let's go find them:

```
$ kubectl get prometheus -n sven-prometheus
NAME         VERSION   DESIRED   READY   RECONCILED   AVAILABLE   AGE
prometheus   v2.22.1   2         2       True         True        15m

$ kubectl get pod -A | grep prometheus
sven-prometheus      prometheus-operator-6b88448777-bgh5m              1/1     Running                 0               109m
sven-prometheus      prometheus-prometheus-0                           0/2     Init:ImagePullBackOff   0               3m40s
sven-prometheus      prometheus-prometheus-1                           0/2     Init:ImagePullBackOff   0               3m40s
```

OK, the controller created two prometheus pods in my namespace; it looks like the relevant container image is not available on my kind node.
I use `docker pull` to get the image:
```
$ docker pull quay.io/prometheus/prometheus:v2.22.1
v2.22.1: Pulling from prometheus/prometheus
<...>
Digest: sha256:b899dbd1b9017b9a379f76ce5b40eead01a62762c4f2057eacef945c3c22d210
Status: Downloaded newer image for quay.io/prometheus/prometheus:v2.22.1
quay.io/prometheus/prometheus:v2.22.1
```

Now the prometheus pods are up and running:
```
$ kubectl get pod -A | grep prometheus
sven-prometheus      prometheus-operator-6b88448777-bgh5m              1/1     Running     0               118m
sven-prometheus      prometheus-prometheus-0                           2/2     Running     0               13m
sven-prometheus      prometheus-prometheus-1                           2/2     Running     0               13m
```



