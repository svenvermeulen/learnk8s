# UNDER CONSTRUCTION

# Horizontal Pod Autoscaler

## First look at the scaler

```
$ kubectl explain horizontalpodautoscaler
KIND:     HorizontalPodAutoscaler
VERSION:  autoscaling/v2

DESCRIPTION:
     HorizontalPodAutoscaler is the configuration for a horizontal pod
     autoscaler, which automatically manages the replica count of any resource
     implementing the scale subresource based on the metrics specified.
```

Let's find the controller that manages `HorizontalPodAutoscaler` resources.
An immediate suspect is `cmd/kube-controller-manager/app/autoscaling.go`; a `ControllerDescriptor` for the `HorizontalPodAutoscalerController` is registered by the Controller Manager [here](https://github.com/kubernetes/kubernetes/blob/release-1.30/cmd/kube-controller-manager/app/controllermanager.go#L530).

Let's see if it logs anything interesting:

```
$ kubectl logs -n kube-system kube-controller-manager-sven-test-control-plane | grep -i -e "scal" -e "horizontal" -e "HPA"
I0714 12:38:39.122318       1 controllermanager.go:602] Started "horizontalpodautoscaling"
I0714 12:38:39.122371       1 horizontal.go:168] Starting HPA controller
I0714 12:38:39.122382       1 shared_informer.go:255] Waiting for caches to sync for HPA
I0714 12:38:39.783151       1 resource_quota_monitor.go:218] QuotaMonitor created object count evaluator for horizontalpodautoscalers.autoscaling
I0714 12:38:40.023034       1 shared_informer.go:262] Caches are synced for HPA
```

Looks like horizontal pod autoscaling is running on my cluster, but not doing much work yet.
It does seem to also use an informer, my guess would be that it uses that to cache queries agains Kubernetes API for objects that need scaling.

Let's see if we can get it to scale a pod based on the pod's resource usage.

## Interacting with the scaler

To create a stressed pod that would trigger horizontal auto-scaling I'll be using `progrium/stress` ([source](https://github.com/progrium/docker-stress/blob/master/Dockerfile)).
First, I need to make the image available on my kind cluster:
```
$ kind load docker-image -n sven-test --nodes sven-test-control-plane progrium/stress
Image: "progrium/stress" with ID "sha256:db646a8f40875981809f754e28a3834e856727b12e7662dad573b6b243e3fba4" not yet present on node "sven-test-control-plane", loading...
```

Then I create a pod (without scaling, for now), to see if it works:
```
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: stresspod
spec:
  containers:
  - name: stress
    image: progrium/stress
    command: ["stress", "--cpu", "1", "--timeout", "180s"]
EOF
```
Let's see the CPU usage for this pod (this may take a few seconds to actually get metrics - be patient and try again):
```
$ kubectl top pod stresspod
NAME        CPU(cores)   MEMORY(bytes)   
stresspod   1956m        2Mi             
```
Fine, this uses pretty close to 2 CPU cores, as expected. I get rid of the pod
```
kubectl delete pod stresspod
```

Now let's create a `Deployment` to manage such add the scaling section to allow the HPA to do its thing. Let's take a look at the documentation:

```
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress
spec:
  selector:
    matchLabels:
      run: stress
  template:
    metadata:
      labels:
        run: stress
    spec:
      containers:
      - name: stress
        image: progrium/stress
        command: ["stress", "--cpu", "1", "--timeout", "120s"]
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
EOF
```
This is running now, taking up just about 500cpus (the limit for the deployment):
```
$ kubectl get pod | grep stress
stress-66767865fd-8vq4x                      1/1     Running     0               26s

$ kubectl top pod | grep stress
stress-66767865fd-8vq4x                      500m         1Mi  
                      499m         1Mi        
```


Let's add an autoscaler which scales to up to 10 pods with 100mcpu target average cpu utilization: 
```
cat << EOF | kubectl apply -f - 
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: stress-autoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: stress
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 100
EOF
```

Soon after, we can see the autoscaler is scaling up the deployment:
```
$ kubectl get pods | grep stress
stress-66767865fd-7h4q6                      1/1     Running             0               21s
stress-66767865fd-8vq4x                      1/1     Running             0               97s
stress-66767865fd-9pwkh                      1/1     Running             0               21s
stress-66767865fd-d2vbc                      1/1     Running             0               51s
stress-66767865fd-fv4vc                      1/1     Running             0               51s
stress-66767865fd-ldzsg                      0/1     ContainerCreating   0               6s
stress-66767865fd-mxmjj                      0/1     ContainerCreating   0               6s
stress-66767865fd-q7qrh                      1/1     Running             0               6s

$ kubectl describe hpa stress-autoscaler
Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler
Name:                                                  stress-autoscaler
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Wed, 17 Jul 2024 17:28:20 +0300
Reference:                                             Deployment/stress
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  248% (497m) / 100%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 5 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    SucceededRescale    the HPA controller was able to update the target scale to 5
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type     Reason                        Age                From                       Message
  ----     ------                        ----               ----                       -------
  Normal   SuccessfulRescale             37s                horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
  Normal   SuccessfulRescale             7s                 horizontal-pod-autoscaler  New size: 5; reason: cpu resource utilization (percentage of request) above target

```

Corresponding log entries can be seen:
```
$ kubectl logs -n kube-system kube-controller-manager-sven-test-control-plane | grep -i -e "Successful rescale of stress-autoscaler"
I0717 14:29:36.052502       1 horizontal.go:691] Successful rescale of stress-autoscaler, old size: 1, new size: 3, reason: cpu resource utilization (percentage of request) above target
I0717 14:30:06.190744       1 horizontal.go:691] Successful rescale of stress-autoscaler, old size: 3, new size: 5, reason: cpu resource utilization (percentage of request) above target
I0717 14:30:21.250972       1 horizontal.go:691] Successful rescale of stress-autoscaler, old size: 5, new size: 8, reason: cpu resource utilization (percentage of request) above target
I0717 14:30:51.422557       1 horizontal.go:691] Successful rescale of stress-autoscaler, old size: 8, new size: 10, reason: cpu resource utilization (percentage of request) above target
```

## A look at the code

Let's take a closer look at the code for the [autoscaler](https://github.com/kubernetes/kubernetes/blob/release-1.30/cmd/kube-controller-manager/app/autoscaling.go).
`startHPAControllerWithMetricsClient` is called, passing in a few dependencies; Then, `Run` is called on the `HorizontalController`.
in `horizontal.go` ([code](https://github.com/kubernetes/kubernetes/tree/release-1.30/pkg/controller/podautoscaler/horizontal.go)) we run into `NewHorizontalController`, which puts together various dependencies and constructs a `HorizontalController`.

The following dependencies look interesting:
- eventRecorder: used to publish the events like 'SueccessfulRescale'
- monitor: publishes some metrics about the HPA (total reconciliations and their duration; total metrics calculated and duration)
- queue: a rate-limited queue of work items
- mapper
some additional dependencies are set after construction of `HorizontalController`:
- hpaLister
- podLister
- replicaCalc
- 


The main work of `HorizontalController` seems to be carried out in `reconcileAutoscaler` TODO

### The Informer

What is it used for? I had said 
> my guess would be that it uses that to cache queries agains Kubernetes API for objects that need scaling.

