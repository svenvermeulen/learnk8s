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
- An `APIService` names `v1beta1.metrics.k8s.io` that references the `Service` `metrics-server`.
 
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

Looks like a certificate issue.

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
I edit `components.yaml` I downloaded before, adding a line `- --kubelet-insecure-tls` to the `args` for the `metrics-server` container and apply.
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
 
Looks like installation was successful. Let's play around with it a bit.


