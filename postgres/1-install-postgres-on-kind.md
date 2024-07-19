# Installing Postgres on Kind

## Introduction

I'm following [these](https://www.digitalocean.com/community/tutorials/how-to-deploy-postgres-to-kubernetes-cluster) instructions on installing postgres on my kind kubernetes installation, digging a bit deeper into how it works and how it behaves along the way.

## Installation

As the guide mentions,
> Storing sensitive data in a ConfigMap is not recommended due to security concerns. When handling sensitive data within Kubernetes, it’s essential to use Secrets and follow security best practices to ensure the protection and confidentiality of your data.

Still, the example creates a configmap with the credentials in it, so I'll follow the guide first and worry about the secrets later.

What I will do differently, though, is that I'll create all postgres-related resources in their own namespace, `sven-postgres`.

```
$ kubectl create ns sven-postgres
namespace/sven-postgres created
$ kubectl apply -f postgres-configmap.yaml -n sven-postgres
configmap/postgres-secret created
$ kubectl get configmap -n sven-postgres
NAME               DATA   AGE
kube-root-ca.crt   1      80s
postgres-secret    3      67s
```

Creating the `PersistentVolume`:
```
$ kubectl apply -f psql-pv.yaml -n sven-postgres
persistentvolume/postgres-volume created

$ kubectl get pv postgres-volume
NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
postgres-volume   10Gi       RWX            Retain           Available           manual                  20m
```

Creating the `PersistentVolumeClaim`:
```
$ kubectl apply -f psql-claim.yaml -n sven-postgres
persistentvolumeclaim/postgres-volume-claim created
$ kubectl get pv postgres-volume
NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                 STORAGECLASS   REASON   AGE
postgres-volume   10Gi       RWX            Retain           Bound    sven-postgres/postgres-volume-claim   manual                  22m
```
Note that the volume is now in status `Bound`; also, the path refered to in hostPath does not exist:
```
$ kubectl get pv postgres-volume -o jsonpath="{.spec.hostPath.path}"
/data/postgresql

$ ls -l -a /data/postgresql
ls: cannot access '/data/postgresql': No such file or directory
```

Creating the deployment:
```
$ kubectl apply -n sven-postgres -f ps-deployment.yaml 
deployment.apps/postgres created

$ kubectl get deploy -n sven-postgres
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
postgres   0/3     3            0           30s

$ kubectl get po -n sven-postgres
NAME                       READY   STATUS              RESTARTS   AGE
postgres-c6858b8d9-hkgps   0/1     ContainerCreating   0          52s
postgres-c6858b8d9-mhrrh   0/1     ContainerCreating   0          52s
postgres-c6858b8d9-wtgtt   0/1     ContainerCreating   0          52s
```

After a while, all replicas are running:
```
$ kubectl get po -n sven-postgres
NAME                       READY   STATUS    RESTARTS        AGE
postgres-c6858b8d9-hkgps   1/1     Running   0               5m21s
postgres-c6858b8d9-mhrrh   1/1     Running   1 (2m13s ago)   5m21s
postgres-c6858b8d9-wtgtt   1/1     Running   0               5m21s
```

Creating the service:
```
$ kubectl apply -f ps-service.yaml -n sven-postgres
service/postgres created
```

Connect to the db with the `psql` client inside one of the postgres pods:
```
$ kubectl exec -it -n sven-postgres postgres-c6858b8d9-hkgps -- psql -h localhost -U ps_user --password -p 5432 ps_db
Password: 
psql (14.12 (Debian 14.12-1.pgdg120+1))
Type "help" for help.

ps_db=# \conninfo
You are connected to database "ps_db" as user "ps_user" on host "localhost" (address "::1") at port "5432".

```

### Where is the volume?

```
$ kubectl describe pod postgres-c6858b8d9-hkgps -n sven-postgres
Name:             postgres-c6858b8d9-hkgps
Namespace:        sven-postgres
       <...>
Containers:
  postgres:
    Container ID:   containerd://e036dc1db1e798cd966c1e49e868485c4820598cd4df5d83c4004d6b8730d05e
    Image:          postgres:14
    Image ID:       docker.io/library/postgres@sha256:2f7365d1f574dba34f6332978169afe60af9de9608fffbbfecb7d04cc5233698
    Port:           5432/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 19 Jul 2024 17:44:03 +0300
    Ready:          True
    Restart Count:  0
    Environment Variables from:
      postgres-secret  ConfigMap  Optional: false
    Environment:       <none>
    Mounts:
      /var/lib/postgresql/data from postgresdata (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-tbtjb (ro)
Conditions:
       <...>
Volumes:
  postgresdata:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  postgres-volume-claim
    ReadOnly:   false
       <...>
```

On the pod, a volume is mounted at `/var/lib/postgresql/data` from `postgresdata` - the `PVC` is `postgres-volume-claim`:
```
$ kubectl get pvc -n sven-postgres
NAME                    STATUS   VOLUME            CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgres-volume-claim   Bound    postgres-volume   10Gi       RWX            manual         39m
```

The volume, `postgres-volume`:
```
$ kubectl get pv -n sven-postgres postgres-volume -o jsonpath="{.spec.hostPath.path}"
/data/postgresql
```

Refers to a path `/data/postgresql`, which, confusingly, still does not exist.






## Next Steps


