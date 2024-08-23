## Introduction

[Aqua security](https://www.aquasec.com/) specialize in security solutions for Cloud Native, Container and Serverless. I ran into a [talk](https://www.youtube.com/watch?v=7mgBxr4D-xs) by one of their employees, Benjy Portnoy. `kube-bench` is shown in this talk, so I decided to have a look.


## Getting started

- repo: https://github.com/aquasecurity/kube-bench
- have a kubernetes cluster at hand

## Running kube-bench on my local cluster

The instructions in the README are simple enough:

```
$ kubectl apply -f job.yaml
job.batch/kube-bench created

$ kubectl get pods
NAME                                         READY   STATUS              RESTARTS       AGE
kube-bench-b8nhs                             0/1     ContainerCreating   0              7s
...


$ kubectl logs kube-bench-b8nhs
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.3 Ensure that the controller manager pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.4 Ensure that the controller manager pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.5 Ensure that the scheduler pod specification file permissions are set to 600 or more restrictive (Automated)
...
...
...
```

Let's look at all these summaries:

```
$ kubectl logs kube-bench-b8nhs | grep -A5 "^== Summary"
== Summary master ==
39 checks PASS
9 checks FAIL
13 checks WARN
0 checks INFO

--
== Summary etcd ==
7 checks PASS
0 checks FAIL
0 checks WARN
0 checks INFO

--
== Summary controlplane ==
0 checks PASS
0 checks FAIL
5 checks WARN
0 checks INFO

--
== Summary node ==
16 checks PASS
1 checks FAIL
7 checks WARN
0 checks INFO

--
== Summary policies ==
0 checks PASS
0 checks FAIL
35 checks WARN
0 checks INFO

== Summary total ==
61 checks PASS
10 checks FAIL
60 checks WARN
0 checks INFO
```

Nice, a whole lot of warnings and 10 failures. Let's have a look at them:

```
$ kubectl logs kube-bench-b8nhs | grep '^\[FAIL\]'
[FAIL] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Automated)
[FAIL] 1.2.5 Ensure that the --kubelet-certificate-authority argument is set as appropriate (Automated)
[FAIL] 1.2.17 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.2.18 Ensure that the --audit-log-path argument is set (Automated)
[FAIL] 1.2.19 Ensure that the --audit-log-maxage argument is set to 30 or as appropriate (Automated)
[FAIL] 1.2.20 Ensure that the --audit-log-maxbackup argument is set to 10 or as appropriate (Automated)
[FAIL] 1.2.21 Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)
[FAIL] 1.3.2 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 1.4.1 Ensure that the --profiling argument is set to false (Automated)
[FAIL] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
```


## Fixing an issue and re-running

Let's zoom in on the last failure:

```
[FAIL] 4.1.1 Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)
```

In the kube-bench logs, there is more detailed information:

```
$ kubectl logs kube-bench-b8nhs | grep -A2 '^4\.1\.1\s'
4.1.1 Run the below command (based on the file location on your system) on the each worker node.
For example, chmod 600 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Let's see the permissions on this file:

```
$ docker exec -it sven-test-control-plane ls -l /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
-rw-r--r-- 1 root root 1763 Jun 23  2022 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Now I can set the permissions to the proposed value and verify:

```
$ docker exec -it sven-test-control-plane chmod 600 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

$ docker exec -it sven-test-control-plane ls -l /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
-rw------- 1 root root 1763 Jun 23  2022 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

If I run `kube-bench` again, I expect one less failure:

```
$ kubectl delete job kube-bench
job.batch "kube-bench" deleted

$ kubectl apply -f job.yaml
job.batch/kube-bench created

$ kubectl get pods
NAME                                         READY   STATUS      RESTARTS      AGE
kube-bench-s5tlg                             0/1     Completed   0             5s
```

Let's check out the logs:
```
$ kubectl logs kube-bench-s5tlg | grep '^\[FAIL\]' | wc -l
9

$ kubectl logs kube-bench-s5tlg | grep -A2 '^4\.1\.1\s'
$
```

Nice!

## Looking at the code

`kube-bench` uses [cobra](https://github.com/spf13/cobra) to parse its command line arguments.
The [root command](https://github.com/aquasecurity/kube-bench/blob/v0.8.0/cmd/root.go#L66-L141) is run when no arguments are present; this is the case when we `kubectl apply -f job.yaml`.

Various calls to `runChecks` are made, each time passing in a different node type ("master", "node", "etcd", etc.).
Let's zoom in on the `check.NODE` case:

```
		glog.V(1).Info("== Running node checks ==")
		runChecks(check.NODE, loadConfig(check.NODE, bv), detecetedKubeVersion)
```

`loadConfig` does what its name implies; in our case, it loads `node.yaml` from the appropriate subdirectory of `cfg`, based on the benchmark version specified. To see the actual config file used, I can enable high verbosity on `kube-bench` by providing the `-v 3` on the command line. I do this by editing `job.yaml`:

```
...
    spec:
      containers:
        - command: ["kube-bench"]
          args: ["-v", "3"]
          image: docker.io/aquasec/kube-bench:v0.8.0
          name: kube-bench
...
```

Now, when running `kube-bench` again, I see this output:
```
...
I0823 12:07:27.057683   22318 util.go:132] Looking for config specific CIS version "cis-1.7"
I0823 12:07:27.057691   22318 util.go:136] Looking for file: cfg/cis-1.7/controlplane.yaml
I0823 12:07:27.057787   22318 common.go:274] Using config file: cfg/cis-1.7/config.yaml
I0823 12:07:27.057831   22318 common.go:78] Using test file: cfg/cis-1.7/controlplane.yaml
...
```

Let's look at `cfg/cis-1.7/node.yaml` - it starts off with control `4.1.1` which we initially failed and remediated:
```
---
controls:
version: "cis-1.7"
id: 4
text: "Worker Node Security Configuration"
type: "node"
groups:
  - id: 4.1
    text: "Worker Node Configuration Files"
    checks:
      - id: 4.1.1
        text: "Ensure that the kubelet service file permissions are set to 600 or more restrictive (Automated)"
        audit: '/bin/sh -c ''if test -e $kubeletsvc; then stat -c permissions=%a $kubeletsvc; fi'' '
        tests:
          test_items:
            - flag: "permissions"
              compare:
                op: bitmask
                value: "600"
        remediation: |
          Run the below command (based on the file location on your system) on the each worker node.
          For example, chmod 600 $kubeletsvc
        scored: true

...
```

This uses a simple shell command to audit the file permissions on the file we're interested in.
The corresponding go type is `Check`, defined [here](https://github.com/aquasecurity/kube-bench/blob/v0.8.0/check/check.go#L67-L88).
Then, `TestItem`, defined [here](https://github.com/aquasecurity/kube-bench/blob/v0.8.0/check/test.go#L59-L69) is used to test the results of the audit (in our case, the result of auditing the file should be that the `permissions` bitmask equals `600`).

## Conclusion

`kube-bench` provides an easy way to audit kubernetes clusters against CIS security benchmarks. Auditing is set up using a set of yaml files, which presribe shell commands to obtain data on the cluster's nodes.



