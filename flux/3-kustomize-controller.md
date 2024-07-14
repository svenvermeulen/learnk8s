Let's now have a look at the [Kustomize Controller](https://fluxcd.io/flux/components/kustomize/)

> The kustomize-controller is a Kubernetes operator, specialized in running continuous delivery pipelines for infrastructure and workloads defined with Kubernetes manifests and assembled with Kustomize.

Sounds promising? Let's dig into it.

##Overview

This is another instance of the [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/); it is implemented using
- a [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), in particular the [kustomization](https://github.com/fluxcd/kustomize-controller/blob/v1.3.0/config/crd/bases/kustomize.toolkit.fluxcd.io_kustomizations.yaml) CRD
- a [kustomization controller](https://github.com/fluxcd/kustomize-controller/blob/v1.3.0/internal/controller/kustomization_controller.go) which monitors instances of the kustomization CRD and makes sure cluster state reflects what's described in these.


##Code

Let's inspect the kustomization controller code a bit to get a feel for how this should all behave.
Some questions to have in mind while digging in:
- how does the controller detect the difference between the desired and current state of a kustomization?
- how is the kustomization executed? Does it use the `kustomize` command line tool (probably its `build` command), or does it import some library and do the same thing instead?

###KustomizationReconciler

This reconciler populates a `kustomizev1.kustomization`; this is a CRD describing the desired state for a kustomization.
It obtains the source artifact for the object it's reconciling, then makes sure that all dependencies (declared in `spec.dependsOn`) are ready.
The `kustomization` CRD is patched to reflect that reconciliation is in progress.
The current version of the source is obtained from the `source-controller` over http and extracted to a temp directory.
Next, the `reconcile` function makes sure that the kustomization's `spec.path` exists inside that directory.

Now, `reconcile` creates a kubernetes client and calls three interesting methods: `generate`, `build` and `apply`.

`generate` inspects the directory given in `spec.path` and creates a default `kustomization.yaml` if no `kustomization.yaml` or `kustomization.yml` or `kustomization` exists.

`build` now performs a set of actions that correspond to what `kustomize build` does, i.e. it creates the actual manifests based on kustomization.yaml.

Finally, `apply` actually applies the kustomized manifests on the cluster. It groups all resources in the manifests into 3 groups:
- Cluster definitions (namespaces, CRDs)
- Kubernetes Class types (e.g. RuntimeClass, PriorityClass)
- Resources
These groups are then applied on the cluster in order.

After this, `reconcile` makes sure that the reconciled resources are healthy.

##Practice

Let's play around with this a bit:
- Create a directory with a `kustomization` and a deployment
- Upload this to a bucket on a local `minio` pod I have running on my cluster
- Create a `Bucket` to have the `source-controller` fetch data from minio
- Create a `Kustomization` resource on the cluster to have

###Creating the kustomization

Resources adapted from [kubernetes documentation](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/):
```
mkdir mykustomization

cat <<EOF >./mykustomization/kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
EOF

cat <<EOF > ./mykustomization/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF > ./mykustomization/increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  namespace: default
spec:
  replicas: 3
EOF

```

###Upload artifact to minio

I followed [these](https://min.io/docs/minio/kubernetes/upstream/index.html) instructions to install a dev version of minio on my cluster. 
I have used `mc alias set myminio http://localhost:9000 minioadmin minioadmin` to create an `mc` alias `myminio` for it. 
On that minio instance, I used `mc cp ./mykustomization.tar.gz myminio/kustomizebucket` to create a bucket named `kustomizebucket`.

I'll store `mykustomization.tar.gz` in that bucket:
```
$ mc mb myminio/kustomizebucket
Bucket created successfully `myminio/kustomizebucket`.

$ mc cp --recursive ./mykustomization/ myminio/kustomizebucket
```

###Create a Bucket resource on k8s

note: I've created a port-forward to `localhost:9000` for my minio instance
I need a secret to store the username and password for accessing my dev minio instance.
Trigger warning: I use the default username `minioadmin` and password `minioadmin`.

```
cat << EOF | kubectl apply -f - 
apiVersion: v1
data:
  accesskey: bWluaW9hZG1pbg==
  secretkey: bWluaW9hZG1pbg==
kind: Secret
metadata:
  name: minio-bucket-secret
type: Opaque
EOF
```

Finally, I create the `Bucket`:
```
cat << EOF | kubectl apply -f - 
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: Bucket
metadata:
  name: kustomizebucket
spec:
  bucketName: kustomizebucket
  endpoint: 10.244.0.16:9000
  interval: 5m0s
  timeout: 60s
  insecure: true
  secretRef:
    name: minio-bucket-secret
EOF

$ kubectl get bucket kustomizebucket
NAME              ENDPOINT           AGE   READY   STATUS
kustomizebucket   10.244.0.16:9000   46m   True    stored artifact: revision 'sha256:21ed3370404007dfd72b5aacec66f04d624d56d8458bc07a166a3b7038a68bf9'
```

From now on, flux will take care of synchronizing the contents of bucket `kustomizebucket` on my minio instance to the `source-controller`.
When I next create a `Kustomization` resource, the `kustomize-controller` will grab that source and, if it has changed, `build` manifests and `apply` them on the cluster.

###Create a Kustomization resource on k8s

Now for the `Kustomization`. Creating this resource will tell flux to build and apply a kustomization on the cluster based on what it pulls out of `kustomizebucket` - a `Deployment` named `my-nginx`, with 3 replicas.

Let's see it in action:

```
cat << EOF | kubectl apply -f - 
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mytestkustomization
spec:
  interval: 5m0s
  prune: true
  sourceRef:
    kind: Bucket
    namespace: default
    name: kustomizebucket
EOF
```

The kustomization has been applied to the cluster, resulting in the creation of the `Deployment`:

```
$ kubectl get kustomization mytestkustomization
NAME                  AGE   READY   STATUS
mytestkustomization   15s   True    Applied revision: sha256:18616c26738bf62e20a8d2e7b535d91f4a37b04e536016df30fe685d9d80e833
```

and indeed there is now a deployment with 3 replicas present on the cluster:

```
$ kubectl get deployment my-nginx
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
my-nginx   3/3     3            3           33s
```





