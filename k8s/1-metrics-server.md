## Introduction

I want to install and play around with [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/). One of its prerequisites is the [Metrics Server](https://github.com/kubernetes-sigs/metrics-server):

> Metrics Server is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines.
> 
> Metrics Server collects resource metrics from Kubelets and exposes them in Kubernetes apiserver through Metrics API for use by Horizontal Pod Autoscaler and Vertical Pod Autoscaler. Metrics API can also be accessed by kubectl top, making it easier to debug autoscaling pipelines.

Here, I walk through installing the metrics server on my cluster, take a look at some of its code, and interact with it to see how it behaves.

## Prerequisites

To begin with, metrics server is not installed on my cluster:

```
$ kubectl top pods
error: Metrics API not available
$ kubectl top nodes
error: Metrics API not available
```

Metrics server has some prerequisites of its own:

### The kube-apiserver must enable an aggregation layer.

The appropriate Kubernetes Apiserver flags [need](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/#enable-kubernetes-apiserver-flags) to be enabled:

```
kubectl get pod -n kube-system kube-apiserver-sven-test-control-plane -o yaml | yq ".spec.containers[0].command"
- kube-apiserver
<...>
- --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
- --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
- --requestheader-allowed-names=front-proxy-client
- --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
- --requestheader-extra-headers-prefix=X-Remote-Extra-
- --requestheader-group-headers=X-Remote-Group
- --requestheader-username-headers=X-Remote-User
<...>
```

The required flags are enabled.
 
### Nodes must have Webhook authentication and authorization enabled.




### Kubelet certificate needs to be signed by cluster Certificate Authority 

(or disable certificate validation by passing --kubelet-insecure-tls to Metrics Server)

### Container runtime must implement a container metrics RPCs (or have cAdvisor support)




### Network should support following communication:
- Control plane to Metrics Server. Control plane node needs to reach Metrics Server's pod IP and port 10250 (or node IP and custom port if hostNetwork is enabled). Read more about control plane to node communication.
- Metrics Server to Kubelet on all nodes. Metrics server needs to reach node address and Kubelet port. Addresses and ports are configured in Kubelet and published as part of Node object. Addresses in .status.addresses and port in .status.daemonEndpoints.kubeletEndpoint.port field (default 10250). Metrics Server will pick first node address based on the list provided by kubelet-preferred-address-types command line flag (default InternalIP,ExternalIP,Hostname in manifests).
 
 
## Installation
 
Installation is a matter of applying a [yaml manifest](https://github.com/kubernetes-sigs/metrics-server#installation) on the cluster.
Let's have a look at this file:

```
$ cat components.yaml | grep '^kind:' | sort | uniq -c
     1 kind: APIService
     2 kind: ClusterRole
     2 kind: ClusterRoleBinding
     1 kind: Deployment
     1 kind: RoleBinding
     1 kind: Service
     1 kind: ServiceAccount
```

Of interest are
- A `Deployment` named `metrics-server`
  - this uses the image `registry.k8s.io/metrics-server/metrics-server:v0.7.1`
- A `Service` named `metrics-server` that sends https requests to the `metrics-server` deployment
- An `APIService` named `v1beta1.metrics.k8s.io` that references the `Service` `metrics-server`.
 
Applying the file goes smoothly:

```
$ kubectl apply -f components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

Now let's see if `kubectl top` works:

```
$ kubectl top pods
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get pods.metrics.k8s.io)
$ kubectl top nodes
Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)
```

That still doesn't work, although the error message is different than before.
Let's look at the logs of the pod running metrics server:
```
$ kubectl get pods -A | grep metrics
kube-system          metrics-server-5bc776bf9b-2kxf5                   0/1     Running     0               2m25s

$ kubectl logs -n kube-system metrics-server-5bc776bf9b-2kxf5
I0716 09:29:40.218833       1 serving.go:374] Generated self-signed cert (/tmp/apiserver.crt, /tmp/apiserver.key)
I0716 09:29:40.500829       1 handler.go:275] Adding GroupVersion metrics.k8s.io v1beta1 to ResourceManager
E0716 09:29:40.625004       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.21.0.2:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.21.0.2 because it doesn't contain any IP SANs" node="sven-test-control-plane"
I0716 09:29:40.637675       1 requestheader_controller.go:169] Starting RequestHeaderAuthRequestController
I0716 09:29:40.638288       1 shared_informer.go:311] Waiting for caches to sync for RequestHeaderAuthRequestController
I0716 09:29:40.637792       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file"
I0716 09:29:40.638863       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0716 09:29:40.637791       1 configmap_cafile_content.go:202] "Starting controller" name="client-ca::kube-system::extension-apiserver-authentication::client-ca-file"
I0716 09:29:40.639232       1 shared_informer.go:311] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0716 09:29:40.640000       1 dynamic_serving_content.go:132] "Starting controller" name="serving-cert::/tmp/apiserver.crt::/tmp/apiserver.key"
I0716 09:29:40.643433       1 secure_serving.go:213] Serving securely on [::]:10250
I0716 09:29:40.643593       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
I0716 09:29:40.739735       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file
I0716 09:29:40.739813       1 shared_informer.go:318] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0716 09:29:40.739916       1 shared_informer.go:318] Caches are synced for RequestHeaderAuthRequestController
E0716 09:29:55.621945       1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.21.0.2:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.21.0.2 because it doesn't contain any IP SANs" node="sven-test-control-plane"
I0716 09:30:04.698408       1 server.go:191] "Failed probe" probe="metric-storage-ready" err="no metrics to serve"
```

Looks like a certificate issue when metrics server tries to scrape metrics from the node.

In the [list of requirements](https://github.com/kubernetes-sigs/metrics-server#requirements) for metrics server, there is this point:
> Kubelet certificate needs to be signed by cluster Certificate Authority (or disable certificate validation by passing --kubelet-insecure-tls to Metrics Server)

Let's see what flags are passed to metrics-server:

```
$ kubectl get pod -n kube-system metrics-server-5bc776bf9b-thmv7 -o yaml | yq '.spec.containers[0].args'
- --cert-dir=/tmp
- --secure-port=10250
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --kubelet-use-node-status-port
- --metric-resolution=15s
```

I'll try adding the `--kubelet-insecure-tls` flag - good enough for my kind cluster.
I edit `components.yaml` which I downloaded before, adding a line `- --kubelet-insecure-tls` to the `args` for the `metrics-server` container and apply.
Now the `metrics-server` pod runs - let's see if `kubectl top` works:

```
$ kubectl top pods
NAME                                         CPU(cores)   MEMORY(bytes)   
my-nginx-77d5cb496b-4dbr9                    0m           3Mi             
my-nginx-77d5cb496b-fqspw                    0m           2Mi             
my-nginx-77d5cb496b-j6b29                    0m           2Mi             
my-nginx-77d5cb496b-zkfmb                    0m           2Mi             
svenhelmrelease-svenchart-568697c464-4x7tc   1m           1Mi             
svenhelmrelease-svenchart-568697c464-84vgg   1m           1Mi             
svenhelmrelease-svenchart-568697c464-jffkl   1m           1Mi             
svenhelmrelease-svenchart-568697c464-nxfjr   1m           2Mi             
svenhelmrelease-svenchart-568697c464-zs6bm   1m           1Mi             

$ kubectl top nodes
NAME                      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
sven-test-control-plane   415m         10%    1220Mi          15%   
```
 
Looks like installation was successful. 

To make it easier to see what's going on in the metrics server, it would be nice to set logging verbosity to a higher level. The README [tells us](https://github.com/kubernetes-sigs/metrics-server/tree/v0.7.1#configuration) we can get detailed information about the available flags by running:
```
docker run --rm registry.k8s.io/metrics-server/metrics-server:v0.7.1 --help
```

Leaving out many other interesting but less relevant entries, I can see 
```
  -v, --v Level                              number for the log level verbosity
```
So adding an additional argument to the deployment's `args` array will increase verbosity of the log entries:

```
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        - --v=6
```

Let's play around with it a bit.

## Looking at some code

After a bit of digging starting at `func main`, it looks like the interesting funcs that get called are
- `config.Complete` ([code](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/server/config.go#L47)).
- `server.RunUntil` ([code](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/server/server.go#L90)).


### config.Complete

`config.Complete` instantiates a `Server`, injecting its dependencies, which it first sets up:

- a pair of `Informer`s, one for pods, one for nodes
- a HTTP handler to serve `metrics`
- a Scraper
- a Storage (for scraped data)

The `Informers` help reduce the number of calls to the Kubernetes API, so that when the metrics server repeatedly retrieves information about nodes and pods, that information is served from a cache which only gets updated based on events when the information actually changes.

The `Scraper` iterates over all nodes selected by its `selector` and gets metrics from it through a kubelet client.
It then iterates over all pods on the node, and stores metrics for each pod in a separate map.

Finally, `config.Complete` sets up the health probes for the metrics server.


### server.RunUntil

`RunUntil` starts data collection processes in the background on a few goroutines:
- the `Informers` that keep track of nodes and pods on the cluster
- the `Scraper` that collects data from nodes at set intervals 
  - `Storage.Store` stores the scraped per-node and per-pod metrics in separate maps; they can be retrieved using `Storage.GetNodeMetrics` and `Storage.GetPodMetrics`. These methods end up getting called by `api.nodeMetrics.List`;  


## Metrics

Metrics Server exposes metrics about nodes and pods for consumption, but it also [exposes metrics](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/server/metrics.go#L29-L49) about itself:

[server](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/server/server.go) exposes metrics about itself:
- a `Histogram` for the total time spent collecting and storing metrics

[scraper](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/scraper/scraper.go) exposes metrics about its scraping process:
- a `HistogramVec` for request duration
- a `CounterVec` for request count
- a `GaugeVec` for last request time

[api](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/api/monitoring.go) exposes
- a `HistogramVec` for the freshness of the exported metrics

[storage](https://github.com/kubernetes-sigs/metrics-server/blob/v0.7.1/pkg/storage/monitoring.go) exposes
- a `GaugeVec` for the number of metric points stored

## In practice

Metrics are scraped from the nodes and pods and stored in a 15-second cycle:

```
$ kubectl logs -n kube-system metrics-server-85994cf464-vqtth | grep -e "Storing metrics" -e "Scraping metrics" | tail -n 9
I0717 08:42:30.513473       1 server.go:136] "Scraping metrics"
I0717 08:42:30.513906       1 scraper.go:121] "Scraping metrics from nodes" nodes=["sven-test-control-plane"] nodeCount=1 nodeSelector=""
I0717 08:42:30.684661       1 server.go:139] "Storing metrics"
I0717 08:42:45.513031       1 server.go:136] "Scraping metrics"
I0717 08:42:45.513068       1 scraper.go:121] "Scraping metrics from nodes" nodes=["sven-test-control-plane"] nodeCount=1 nodeSelector=""
I0717 08:42:45.546484       1 server.go:139] "Storing metrics"
I0717 08:43:00.513672       1 server.go:136] "Scraping metrics"
I0717 08:43:00.513710       1 scraper.go:121] "Scraping metrics from nodes" nodes=["sven-test-control-plane"] nodeCount=1 nodeSelector=""
I0717 08:43:00.562084       1 server.go:139] "Storing metrics"
```

Executing `kubectl top pods` causes a few http requests to be logged by metrics-server:
```
$ kubectl logs -n kube-system metrics-server-85994cf464-vqtth | grep -e "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods"
I0717 08:15:07.812344       1 handler.go:143] metrics-server: GET "/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" satisfied by gorestful with webservice /apis/metrics.k8s.io/v1beta1
I0717 08:15:07.812702       1 httplog.go:132] "HTTP" verb="LIST" URI="/apis/metrics.k8s.io/v1beta1/namespaces/default/pods" latency="700.49µs" userAgent="kubectl/v1.26.0 (linux/amd64) kubernetes/b46a3f8" audit-ID="d1d6bcff-aee5-45b0-a885-3d5a601c7205" srcIP="10.244.0.1:25364" resp=200
```

Same for `kubectl top nodes`:
```
$ cat metrics-server-log | grep "/apis/metrics.k8s.io/v1beta1/nodes"
$ kubectl logs -n kube-system metrics-server-85994cf464-vqtth | grep -e '/apis/metrics.k8s.io/v1beta1/nodes'
I0717 08:15:05.286614       1 handler.go:143] metrics-server: GET "/apis/metrics.k8s.io/v1beta1/nodes" satisfied by gorestful with webservice /apis/metrics.k8s.io/v1beta1
I0717 08:15:05.286952       1 httplog.go:132] "HTTP" verb="LIST" URI="/apis/metrics.k8s.io/v1beta1/nodes" latency="715.296µs" userAgent="kubectl/v1.26.0 (linux/amd64) kubernetes/b46a3f8" audit-ID="49200f6f-771b-427e-bdde-02991f159013" srcIP="10.244.0.1:25364" resp=200
```

This is probably how the `Horizontal Pod Autoscaler` interacts with `metrics-server` too; when playing with `HPA`, I might be able to see the same log entries when *that* contacts metrics server.



 





