
During the [installation](TODO) of flux on the cluster, we saw that a number of `CustomResourceDefinitions` and `Deployments` were created.
Let's dive a bit deeper into one of the `Deployments`, which manages a pod running the `source-controller`.

## Source Controller Deployment

The `source-controller` deployment in the `flux-system` namespace manages a pod based on the `ghcr.io/fluxcd/source-controller:v1.3.0` image:

```
$ kubectl get deployment -n flux-system source-controller -o=jsonpath='{$.spec.template.spec.containers[:1].image}'
ghcr.io/fluxcd/source-controller:v1.3.0
```

`fluxcd/source-controller` is published to the GitHub Container Registry (`ghcr.io`) when a [release](https://github.com/fluxcd/source-controller/releases/tag/v1.3.0) of fluxcd `source-controller` is created (When a tag starting with `v` is pushed, `docker/build-push-action` is [used](https://github.com/fluxcd/source-controller/blob/v1.3.0/.github/workflows/release.yml#L69-L81) on the project's `Dockerfile`.

`Dockerfile` [adds](https://github.com/fluxcd/source-controller/blob/v1.3.0/Dockerfile) `source-controller` on top of a plain linux distro. 

## Source Controller Code

`source-controller` (I'm looking at the [v1.3.0 code](https://github.com/fluxcd/source-controller/tree/v1.3.0)) implements a controller that monitors cluster state for a set of custom resources and reconciles actual state towards the desired state.

### Expectations

- `source-controller` probably knows a few different types of repositories (GIT, S3 buckets, etc) and 
- There are probably going to be some go types that correspond to the CustomResourceDefinitions in the flux-system namespace so that the go code can easily work with resources. 
- There should be a control loop in there somewhere that compares the desired state of the cluster (as declared in the resources created on it) with the actual state of the cluster.
- There should be some discrete code for reconciling cluster state to move it towards the desired state


### Actual code

`source-controller` includes a few `CustomResourceDefinitions` [here](https://github.com/fluxcd/source-controller/blob/v1.3.0/config/crd/kustomization.yaml). These are defined in `source-controller` and referenced in `flux` through [this manifest](https://github.com/fluxcd/flux2/blob/main/manifests/bases/source-controller/kustomization.yaml#L4)

Flux also references the source controller deployment [here](https://github.com/fluxcd/flux2/blob/main/manifests/bases/source-controller/kustomization.yaml#L5)

In `main.go`, after some initial preparations, `source-controller` [creates](https://github.com/fluxcd/source-controller/blob/v1.3.0/main.go#L192-L265) the following reconcilers:
- `controller.GitRepositoryReconciler`
- `controller.HelmRepositoryReconciler`
- `controller.BucketReconciler`
- `controller.OCIRepositoryReconciler`

All these Reconcilers are set up to be controlled by a `controllerManager` which is part of [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)

Then, a `http.FileServer` is started up in the background (If and when the current instance is elected leader) and finally the controller manager is started.


#### Reconcilers

The four reconcilers spun up in `main.go` match nicely to four of the CRDs [defined](https://github.com/fluxcd/source-controller/tree/v1.3.0/config/crd/bases) in `source-controller`:

- `GitRepositoryReconciler` <-> `GitRepository`
- `HelmRepositoryReconciler` <-> `HelmRepository`
- `BucketReconciler` <-> `Bucket`
- `OCIRepositoryReconciler` <-> `ocirepositories.source.toolkit.fluxcd.io`

There is a fifth CRD, `HelmChart` which is probably also taken care of by `HelmRepositoryReconciler` - we can confirm this further down the road when looking at the reconcilers one by one.

##### GitRepositoryReconciler
`GitRepositoryReconciler` provides a public method `Reconcile` which in turn calls a private method `reconcile`.

`Reconcile` receives a representation of the `GitRepository` resource it should reconcile.
If this resource is not suspended, it then proceeds to call private `reconcile`, passing in four `gitRepositoryReconcileFunc`s: 
- `reconcileStorage`
- `reconcileSource`
- `reconcileInclude`
- `reconcileArtifact`

`reconcile` creates a temp directory, then calls each of these reconcile functions; in the end, it notifies of the results. Effectively, the result of these actions is that a local version of the source code from the referenced git repository is up-to-date with that repository.

`reconcileStorage` checks the presence and integrity of the artifact in the pod's local storage. If the artifact's digest does not match its contents, it is deleted and re-fetched.

`reconcileSource` takes care of cloning the remote git repository locally, with various optimizations

`reconcileInclude` takes care of subcontents specified in `.spec.include` as detailed [here](https://fluxcd.io/flux/components/source/gitrepositories/#include)

`reconcileArtifact` takes care of archiving the cloned source repository in a tarball. It [sets](https://github.com/fluxcd/source-controller/blob/v1.3.0/internal/controller/gitrepository_controller.go#L681-L690) the `ArtifactInStorageCondition` on the GitRepository's status to `true`, and produces the well-known `stored artifact for revision '<branch>@sha1:<digest>'` message.

##### BucketReconciler

`BucketReconciler` monitors and reconciles `Bucket` type resources. 
It checks if the monitored resource is suspended, and if not, proceeds to apply a series of reconciliation steps:
- reconcileStorage
- reconcileSource
- reconcileArtifact

Finally, it summarizes and patches the resource.
The reconciliation steps act much like they do for `GitRepositoryReconciler`. 


### In Practice

#### GitRepositoryReconciler

Let's see this in practice. Let's try [this example](https://fluxcd.io/flux/components/source/gitrepositories/#example) GitRepository manifest from the fluxcd documentation:

```
$ cat gitrepository.yaml 
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: podinfo
  namespace: default
spec:
  interval: 5m0s
  url: https://github.com/stefanprodan/podinfo
  ref:
    branch: master
```

Apply:
```
$ kubectl apply -f gitrepository.yaml 
gitrepository.source.toolkit.fluxcd.io/podinfo created
```

Inspect the GitRepository resource: 
```
$ kubectl get gitrepository podinfo -o yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"source.toolkit.fluxcd.io/v1","kind":"GitRepository","metadata":{"annotations":{},"name":"podinfo","namespace":"default"},"spec":{"interval":"5m0s","ref":{"branch":"master"},"url":"https://github.com/stefanprodan/podinfo"}}
  creationTimestamp: "2024-07-04T11:51:36Z"
  finalizers:
  - finalizers.fluxcd.io
  generation: 1
  name: podinfo
  namespace: default
  resourceVersion: "863627"
  uid: 3c00bb0b-fcbb-4921-a00b-ae98898878d9
spec:
  interval: 5m0s
  ref:
    branch: master
  timeout: 60s
  url: https://github.com/stefanprodan/podinfo
status:
  artifact:
    digest: sha256:dd99c1e723ce4df59eb06e98aa2e4a57a35fa79f9775c7d57cbb02ee35709f50
    lastUpdateTime: "2024-07-04T11:51:38Z"
    path: gitrepository/default/podinfo/0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
    revision: master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6
    size: 300950
    url: http://source-controller.flux-system.svc.cluster.local./gitrepository/default/podinfo/0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
  conditions:
  - lastTransitionTime: "2024-07-04T11:51:38Z"
    message: stored artifact for revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'
    observedGeneration: 1
    reason: Succeeded
    status: "True"
    type: Ready
  - lastTransitionTime: "2024-07-04T11:51:38Z"
    message: stored artifact for revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'
    observedGeneration: 1
    reason: Succeeded
    status: "True"
    type: ArtifactInStorage
  observedGeneration: 1

```

`spec` describes the desired state of the cluster, while `status` describes the observed state.
`status.artifact.url` points to the http file server running inside the `source-controller` pod; the artifact can be downloaded trivially from pods within the cluster

```
$ wget http://source-controller.flux-system.svc.cluster.local./gitrepository/default/podinfo/0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
```


Based on the `source-controller` code and `status.artifact.url`, we should be able to connect to the source controller pod and locate the artifact on its storage volume.


Let's find the pod in question:
```
$ kubectl get pods -n flux-system | grep source-controller
source-controller-f7c9b977f-sk5pt              1/1     Running   3 (5d7h ago)    7d1h

```

Let's have a quick look at its logs; since I created a `GitRepository` resource, there should be entries related to its reconciliation

```
$ kubectl logs -n flux-system source-controller-f7c9b977f-sk5pt
....
{"level":"info","ts":"2024-07-04T11:51:38.882Z","msg":"stored artifact for commit 'Merge pull request #374 from stefanprodan/release-...'","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository","GitRepository":{"name":"podinfo","namespace":"default"},"namespace":"default","name":"podinfo","reconcileID":"d919ba06-f863-4e01-b1bf-fa210e80dc8d"}
{"level":"info","ts":"2024-07-04T11:56:51.013Z","msg":"no changes since last reconcilation: observed revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository","GitRepository":{"name":"podinfo","namespace":"default"},"namespace":"default","name":"podinfo","reconcileID":"8cf425cb-c10c-4d21-8e5a-b2020af8ee2e"}
{"level":"info","ts":"2024-07-04T12:02:07.337Z","msg":"no changes since last reconcilation: observed revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository","GitRepository":{"name":"podinfo","namespace":"default"},"namespace":"default","name":"podinfo","reconcileID":"469b949b-ed61-4cc5-9ff5-a49186bac0d6"}

```
As per the `GitRepository` spec, the controller performs a reconciliation every five minutes. At first reconciliation, it stores an artifact on its filesystem; subsequent reconciliations (after five minutes every time) tell us nothing has changed.

Let's see if we can find the artifact on the `sourcecontroller` pod's filesystem and play around with it a bit:

```
(inside the source-controller pod)
$ ls -l /data/gitrepository/default/podinfo
total 296
-rw-------    1 nobody   1337        300950 Jul  4 11:51 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
-rw-r--r--    1 nobody   1337             0 Jul  4 11:51 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz.lock
```

`reconcileStorage` amongst other things [verifies](https://github.com/fluxcd/source-controller/blob/v1.3.0/internal/controller/gitrepository_controller.go#L399-L417) the digest of the stored artifact and if it's incorrect, removes the artifact (which will the cause it to obtain the proper artifact again). An event stating `failed to verify integrity of artifact` should be emitted along the way, probably complaining about the checksum of the file being wrong. Let's try it:

```
(inside the source-controller pod)
$ echo 'X' >> /data/gitrepository/default/podinfo/0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
$ ls -l /data/gitrepository/default/podinfo
total 296
-rw-------    1 nobody   1337        300952 Jul  4 12:23 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
-rw-r--r--    1 nobody   1337             0 Jul  4 11:51 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz.lock
```
Notice the file size has increased and the next checksum operation is expected to fail.

The events for the `GitRepository`: 
```
$ kubectl events --for gitrepository/podinfo
LAST SEEN             TYPE      REASON                       OBJECT                  MESSAGE
39m                   Normal    NewArtifact                  GitRepository/podinfo   stored artifact for commit 'Merge pull request #374 from stefanprodan/release-...'
9m38s (x6 over 34m)   Normal    GitOperationSucceeded        GitRepository/podinfo   no changes since last reconcilation: observed revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'
4m50s                 Warning   ArtifactVerificationFailed   GitRepository/podinfo   failed to verify integrity of artifact: computed digest doesn't match 'sha256:dd99c1e723ce4df59eb06e98aa2e4a57a35fa79f9775c7d57cbb02ee35709f50'
```

And indeed the artifact in storage has been replaced with the correct version:
```
$ kubectl exec -n flux-system -it source-controller-f7c9b977f-sk5pt -- /bin/sh
~ $ ls -l /data/gitrepository/default/podinfo
total 296
-rw-------    1 nobody   1337        300950 Jul  4 12:26 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
-rw-r--r--    1 nobody   1337             0 Jul  4 11:51 0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz.lock
```

Likewise, there is a [check](https://github.com/fluxcd/source-controller/blob/v1.3.0/internal/controller/gitrepository_controller.go#L420-L431) in `reconcileStorage` which emits an event `building artifact: disappeared from storage` if the artifact is missing. Let's delete it and watch that event:

```
$ kubectl exec -n flux-system -it source-controller-f7c9b977f-sk5pt -- /bin/sh
~ $ rm /data/gitrepository/default/podinfo/0b1481aa8ed0a6c34af84f779824a74200d5c1d6.tar.gz
```

After a while, when reconciliation runs, the artifact is re-created by the code but the only message I can find about it is `stored artifact for revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'` in the GitRepository's `status.conditions`; no event shows up with `kubectl get events`.

#### BucketReconciler

Let's look at an [example](https://fluxcd.io/flux/components/source/buckets/#example) resource from the flux documentation.

I quickly set up a minio bucket on my cluster following [this quickstart](https://min.io/docs/minio/kubernetes/upstream/)
I put some data in a `TESTFILE.md` and copy it to my bucket `testbucket` on minio instance `myminio`:

```
$ cat 11111 > TESTFILE.md
$ mc cp ./TESTFILE.md myminio/testbucket/
...Documents/selfdev/TESTFILE.md: 6 B / 6 B 
```

I now create an instance of the `Bucket` CRD on my cluster; I expect a corresponding `BucketReconciler` running in a pod to synchronize files from that bucket to its local filesystem. Let's see it in action. My `bucket.yaml` looks like this:





An entry shows up in `kubectl get events` about the new artifact downloaded from `minio`:

```
49s                    Normal    NewArtifact             Bucket/minio-bucket     stored artifact with 1 fetched files from 'testbucket' bucket
``` 

The file is indeed present on the local filesystem of the `source-controller` pod:
```
$ kubectl exec -n flux-system -it source-controller-f7c9b977f-sk5pt -- /bin/sh
~ $ ls
~ $ ls /data/bucket/default/minio-bucket
bc653c4fe645683dca61d42ca0103a3c6929d067235ec8c7e790685df80ade34.tar.gz       latest.tar.gz
bc653c4fe645683dca61d42ca0103a3c6929d067235ec8c7e790685df80ade34.tar.gz.lock

~ $ tar x -O -f /data/bucket/default/minio-bucket/bc653c4fe645683dca61d42ca0103a3c6929d067235ec8c7e790685df80ade34.tar.gz TESTFILE.md
11111
~ $ tar x -O -f /data/bucket/default/minio-bucket/latest.tar.gz TESTFILE.md
11111
```

THe files are exposed through a built-in http server running inside the `source-controller` pod; I can access them from a random pod in the `flux-system` namespace that has `wget` available:

```
~ $ wget -O - http://source-controller.flux-system.svc.cluster.local./bucket/default/minio-bucket/latest.tar.gz | tar zxfO - TESTFILE.md 
Connecting to source-controller.flux-system.svc.cluster.local. (10.96.35.133:80)
writing to stdout
-                    100% |****************************************************************************************************************************************|   123  0:00:00 ETA
written to stdout
11111
```



Now, when I change the contents of `TESTFILE.md` locally and upload to my bucket, flux's `BucketReconciler` will take care of updating the files inside the pod.

```
$ echo 22222 > TESTFILE.md 
$ mc cp ./TESTFILE.md myminio/testbucket/
...Documents/selfdev/TESTFILE.md: 6 B / 6 B ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓
```

Reconciliation has not yet run: 

```
$ kubectl get events
LAST SEEN   TYPE      REASON                  OBJECT                  MESSAGE
19m         Normal    NewArtifact             bucket/minio-bucket     stored artifact with 1 fetched files from 'testbucket' bucket
3m39s       Normal    ArtifactUpToDate        bucket/minio-bucket     artifact up-to-date with remote revision: 'sha256:bc653c4fe645683dca61d42ca0103a3c6929d067235ec8c7e790685df80ade34'
4m30s       Normal    GitOperationSucceeded   gitrepository/podinfo   no changes since last reconcilation: observed revision 'master@sha1:0b1481aa8ed0a6c34af84f779824a74200d5c1d6'
```

A bit later though, reconciliation has run:

```
$ kubectl get events
LAST SEEN   TYPE      REASON                  OBJECT                  MESSAGE
8s          Normal    NewArtifact             bucket/minio-bucket     stored artifact with 1 fetched files from 'testbucket' bucket
5m11s       Normal    ArtifactUpToDate        bucket/minio-bucket     artifact up-to-date with remote revision: 'sha256:bc653c4fe645683dca61d42ca0103a3c6929d067235ec8c7e790685df80ade34'
```

The expectation is now that fetching the `latest.tar.gz` file through `source-controller`'s http server will return the new file contents:

```
~ $ wget -O - http://source-controller.flux-system.svc.cluster.local./bucket/default/minio-bucket/latest.tar.gz | tar zxfO - TESTFILE.md
Connecting to source-controller.flux-system.svc.cluster.local. (10.96.35.133:80)
writing to stdout
-                    100% |****************************************************************************************************************************************|   123  0:00:00 ETA
written to stdout
22222
```

Enough for the `source-controller`. For now, anyway.

