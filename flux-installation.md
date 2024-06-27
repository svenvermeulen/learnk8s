#Flux installation

This post describes my investigation into what happens when I install flux on a k8s cluster. 
The goal is not to teach you - I don't feel I have the authority to do that - but rather to document my own learning.

So without further ado, let's dive into what happens when we install [flux](https://fluxcd.io/) onto a Kubernetes cluster.

##Preparation

- Have a kubernetes cluster to work with; you could also use [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster) to set up a local cluster inside Docker.
- [Install](https://fluxcd.io/flux/installation/#install-the-flux-cli) the Flux CLI on your local machine
- Grab the flux [source code](https://github.com/fluxcd/flux2) to follow along

##Expectations

Let's set a few expectations before applying changes on the cluster; then we verify these expectations afterwards.
[This list](https://github.com/fluxcd/flux2#components) shows that installing helm to our cluster will result in the creation of at least 5 controllers and 13 Custom Resource Definitions.
We'll be able to verify this by using `kubectl get crd` and by looking for corresponding pods running controllers. These are probably created in some flux-specific namespace.

##Operator Pattern

Helm creates a few `CustomResourceDefinitions` and controllers in the cluster. This hints at a very common design pattern in Kubernetes named `Operator`, which allows you to extend the Kubernetes API. To implement it, you create `CustomResourceDefinitions` (CRD) and controllers.
The `CustomResourceDefinitions` provide the API for your functionality by defining new resource types: after installing the CRDs on their cluster, users will be able to instantiate resources using their existing tooling, for example `kubectl`. These resources declare the desired state of the cluster.
The controller is a software component responsible for bringing the current state of the cluster into the desired one as described by the created resources.


##Installation

Before installing, let's verify that there are no `CustomResourceDefinitions` on the cluster:
```
$ kubectl get crd
No resources found
```

###What to expect

The [flux documentation](https://fluxcd.io/flux/installation/) specifies multiple ways to install flux to the cluster: [Bootstrap with Flux CLI](https://fluxcd.io/flux/installation/#bootstrap-with-flux-cli), [Bootstrap with Terraform](https://fluxcd.io/flux/installation/#bootstrap-with-terraform) and [Dev install](https://fluxcd.io/flux/installation/#dev-install) which is what we'll be using here; we just want to see CRDs and controllers being created.

The `Dev Install` documentation gives multiple options; in my opinion, the one we can most easily learn from is "Install with `kubectl`":
```
kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
```

Let's look at that [install.yaml](https://github.com/fluxcd/flux2/releases/latest/download/install.yaml). It's expected to create some namespace and a few CRDs, and install some controllers.

```
$ yq e -N '.kind' ~/Downloads/install.yaml | sort | uniq -c
      3 ClusterRole
      2 ClusterRoleBinding
     13 CustomResourceDefinition
      6 Deployment
      1 Namespace
      3 NetworkPolicy
      1 ResourceQuota
      3 Service
      6 ServiceAccount

```
The expected 13 `CustomResourceDefinitions` are here.
The controllers are likely implemented as code running in pods managed by the 6 `Deployments` that will be created.

###Actually installing
Let's just do the installation and see what pops up:

```
$ kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
namespace/flux-system created
resourcequota/critical-pods created
customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/buckets.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/gitrepositories.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmcharts.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmreleases.helm.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/helmrepositories.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/imagepolicies.image.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/imagerepositories.image.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/imageupdateautomations.image.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/kustomizations.kustomize.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/ocirepositories.source.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/providers.notification.toolkit.fluxcd.io created
customresourcedefinition.apiextensions.k8s.io/receivers.notification.toolkit.fluxcd.io created
serviceaccount/helm-controller created
serviceaccount/image-automation-controller created
serviceaccount/image-reflector-controller created
serviceaccount/kustomize-controller created
serviceaccount/notification-controller created
serviceaccount/source-controller created
clusterrole.rbac.authorization.k8s.io/crd-controller created
clusterrole.rbac.authorization.k8s.io/flux-edit created
clusterrole.rbac.authorization.k8s.io/flux-view created
clusterrolebinding.rbac.authorization.k8s.io/cluster-reconciler created
clusterrolebinding.rbac.authorization.k8s.io/crd-controller created
service/notification-controller created
service/source-controller created
service/webhook-receiver created
deployment.apps/helm-controller created
deployment.apps/image-automation-controller created
deployment.apps/image-reflector-controller created
deployment.apps/kustomize-controller created
deployment.apps/notification-controller created
deployment.apps/source-controller created
networkpolicy.networking.k8s.io/allow-egress created
networkpolicy.networking.k8s.io/allow-scraping created
networkpolicy.networking.k8s.io/allow-webhooks created
```

As expected, 13 `CustomResourceDefinitions` and 6 deployments related to controllers were created (at least 5 were expected based on the documentation); these sit in the `flux-system` namespace which is created first.

###Looking at the CRDs
Let's have a look at the `CustomResourceDefinitions`:

```
$ kubectl get crds
NAME                                             CREATED AT
alerts.notification.toolkit.fluxcd.io            2024-06-27T10:14:57Z
buckets.source.toolkit.fluxcd.io                 2024-06-27T10:14:57Z
gitrepositories.source.toolkit.fluxcd.io         2024-06-27T10:14:57Z
helmcharts.source.toolkit.fluxcd.io              2024-06-27T10:14:57Z
helmreleases.helm.toolkit.fluxcd.io              2024-06-27T10:14:57Z
helmrepositories.source.toolkit.fluxcd.io        2024-06-27T10:14:58Z
imagepolicies.image.toolkit.fluxcd.io            2024-06-27T10:14:58Z
imagerepositories.image.toolkit.fluxcd.io        2024-06-27T10:14:58Z
imageupdateautomations.image.toolkit.fluxcd.io   2024-06-27T10:14:58Z
kustomizations.kustomize.toolkit.fluxcd.io       2024-06-27T10:14:58Z
ocirepositories.source.toolkit.fluxcd.io         2024-06-27T10:14:58Z
providers.notification.toolkit.fluxcd.io         2024-06-27T10:14:58Z
receivers.notification.toolkit.fluxcd.io         2024-06-27T10:14:58Z
```

These can be used with `kubectl` from now on. Let's have a quick look at the `gitrepositories.source.toolkit.fluxcd.io` CRD with kubectl:

```
$ kubectl explain gitrepositories.source.toolkit.fluxcd.io.spec
KIND:     GitRepository
VERSION:  source.toolkit.fluxcd.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     GitRepositorySpec specifies the required configuration to produce an
     Artifact for a Git repository.

FIELDS:
   ignore	<string>
     Ignore overrides the set of excluded patterns in the .sourceignore format
     (which is the same as .gitignore). If not provided, a default will be used,
     consult the documentation for your version to find out what those are.

   include	<[]Object>
     Include specifies a list of GitRepository resources which Artifacts should
     be included in the Artifact produced for this GitRepository.

   interval	<string> -required-
     Interval at which the GitRepository URL is checked for updates. This
     interval is approximate and may be subject to jitter to ensure efficient
     use of resources.

   proxySecretRef	<Object>
     ProxySecretRef specifies the Secret containing the proxy configuration to
     use while communicating with the Git server.

   recurseSubmodules	<boolean>
     RecurseSubmodules enables the initialization of all submodules within the
     GitRepository as cloned from the URL, using their default settings.

   ref	<Object>
     Reference specifies the Git reference to resolve and monitor for changes,
     defaults to the 'master' branch.

   secretRef	<Object>
     SecretRef specifies the Secret containing authentication credentials for
     the GitRepository. For HTTPS repositories the Secret must contain
     'username' and 'password' fields for basic auth or 'bearerToken' field for
     token auth. For SSH repositories the Secret must contain 'identity' and
     'known_hosts' fields.

   suspend	<boolean>
     Suspend tells the controller to suspend the reconciliation of this
     GitRepository.

   timeout	<string>
     Timeout for Git operations like cloning, defaults to 60s.

   url	<string> -required-
     URL specifies the Git repository URL, it can be an HTTP/S or SSH address.

   verify	<Object>
     Verification specifies the configuration to verify the Git commit
     signature(s).
```

This immediately displays some juicy information and gives us spoilers about what lies ahead.

###Looking at the deployments
Let's also have a look at the deployments:

```
$ kubectl get deployment -n flux-system
NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
helm-controller               1/1     1            1           46m
image-automation-controller   1/1     1            1           46m
image-reflector-controller    1/1     1            1           46m
kustomize-controller          1/1     1            1           46m
notification-controller       1/1     1            1           46m
source-controller             1/1     1            1           46m
```

At this point, the controllers probably just sit there doing nothing. We can have a look at the pods' logs; the pod managed by the `source-controller` deployment for example is `source-controller-f7c9b977f-sk5pt` on my cluster:

```
$ kubectl logs -n flux-system source-controller-f7c9b977f-sk5pt
{"level":"info","ts":"2024-06-27T10:16:08.477Z","logger":"setup","msg":"caching of Helm index files is disabled"}
{"level":"info","ts":"2024-06-27T10:16:08.478Z","logger":"setup","msg":"starting manager"}
{"level":"info","ts":"2024-06-27T10:16:08.479Z","logger":"controller-runtime.metrics","msg":"Starting metrics server"}
{"level":"info","ts":"2024-06-27T10:16:08.479Z","logger":"controller-runtime.metrics","msg":"Serving metrics server","bindAddress":":8080","secure":false}
{"level":"info","ts":"2024-06-27T10:16:08.480Z","msg":"starting server","name":"health probe","addr":"[::]:9440"}
{"level":"info","ts":"2024-06-27T10:16:08.582Z","logger":"runtime","msg":"attempting to acquire leader lease flux-system/source-controller-leader-election..."}
{"level":"info","ts":"2024-06-27T10:16:08.597Z","logger":"runtime","msg":"successfully acquired lease flux-system/source-controller-leader-election"}
{"level":"info","ts":"2024-06-27T10:16:08.598Z","msg":"Starting EventSource","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository","source":"kind source: *v1.GitRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.598Z","msg":"Starting Controller","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.600Z","msg":"Starting EventSource","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart","source":"kind source: *v1.HelmChart"}
{"level":"info","ts":"2024-06-27T10:16:08.600Z","msg":"Starting EventSource","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart","source":"kind source: *v1.HelmRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.600Z","msg":"Starting EventSource","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart","source":"kind source: *v1.GitRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.600Z","msg":"Starting EventSource","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart","source":"kind source: *v1beta2.Bucket"}
{"level":"info","ts":"2024-06-27T10:16:08.600Z","msg":"Starting Controller","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart"}
{"level":"info","ts":"2024-06-27T10:16:08.602Z","msg":"Starting EventSource","controller":"helmrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmRepository","source":"kind source: *v1.HelmRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.602Z","msg":"Starting Controller","controller":"helmrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.603Z","msg":"Starting EventSource","controller":"ocirepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"OCIRepository","source":"kind source: *v1beta2.OCIRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.603Z","msg":"Starting Controller","controller":"ocirepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"OCIRepository"}
{"level":"info","ts":"2024-06-27T10:16:08.605Z","msg":"Starting EventSource","controller":"bucket","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"Bucket","source":"kind source: *v1beta2.Bucket"}
{"level":"info","ts":"2024-06-27T10:16:08.606Z","msg":"Starting Controller","controller":"bucket","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"Bucket"}
{"level":"info","ts":"2024-06-27T10:16:08.607Z","logger":"setup","msg":"starting file server"}
{"level":"info","ts":"2024-06-27T10:16:08.699Z","msg":"Starting workers","controller":"gitrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"GitRepository","worker count":2}
{"level":"info","ts":"2024-06-27T10:16:08.704Z","msg":"Starting workers","controller":"helmrepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmRepository","worker count":2}
{"level":"info","ts":"2024-06-27T10:16:08.705Z","msg":"Starting workers","controller":"ocirepository","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"OCIRepository","worker count":2}
{"level":"info","ts":"2024-06-27T10:16:08.705Z","msg":"Starting workers","controller":"helmchart","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"HelmChart","worker count":2}
{"level":"info","ts":"2024-06-27T10:16:08.707Z","msg":"Starting workers","controller":"bucket","controllerGroup":"source.toolkit.fluxcd.io","controllerKind":"Bucket","worker count":2}
```

This looks like a pod that happily started up and does nothing in particular.
In the next part, let's see how we can get it to actually do something.

##Resources

- Ibryam and Huss, "Kubernetes Patterns: Reusable Elements for Designing Cloud Native Applications (Second Edition)", O'Reilly publishing - [free download](https://www.redhat.com/en/engage/kubernetes-containers-architecture-s-201910240918?sc_cid=7013a000003SxS6AAK&gad_source=1&gclid=CjwKCAjwm_SzBhAsEiwAXE2Cv2D0dUK_8_D_KNsxd0qsb_cbjCJ3s-tyPmx8-HlDKjOKqqEzs71vHRoCWEcQAvD_BwE&gclsrc=aw.ds) courtesy of RedHat

