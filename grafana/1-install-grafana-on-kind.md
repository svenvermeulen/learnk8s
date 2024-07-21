# Installing Grafana on Kind

I have a kind cluster running locally; these are my notes following [instructions](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/) from grafana.com for local installation of Grafana on that cluster. 

## Prerequisites

I'm skeptical about the hardware requirements, since I'm running Kind on a puny little laptop.
The Grafana page [states](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/#minimum-hardware-requirements) 
> CPU: 250m (approx 2.5 cores)

I'd expect 250m to correspond to 0.25 cores. So I'm not sure if it's that, or 2500m. We'll see how it runs.

As for the database used by Grafana, [supposedly](https://grafana.com/docs/grafana/latest/setup-grafana/installation/#supported-databases) 
> By default Grafana uses an embedded SQLite database, which is stored in the Grafana installation location.

For the needs of playing around locally, this should be good enough.

## Installation

I'll be using the following namespace:
```
$ kubectl create namespace sven-grafana
namespace/sven-grafana created
```

Then, the instructions tell me to copy-paste some manifests into a file `grafana.yaml`. A quick look at the resources defined in that file shows that the image used for pods managed by the grafana Deployment is `grafana/grafana:latest`. I'll have to load this image onto my kind cluster.

```
$ kubectl apply -f grafana.yaml --namespace=sven-grafana
persistentvolumeclaim/grafana-pvc created
deployment.apps/grafana created
service/grafana created
```

I fully expect a problem in container creation for the `grafana` pod. Let's see:
```
$ kubectl get pod -n sven-grafana | grep grafana
grafana-69946c9bd6-twvbj   0/1     ContainerCreating   0          50s

$ kubectl events -n sven-grafana Pod/grafana-69946c9bd6-twvbj
LAST SEEN             TYPE     REASON                  OBJECT                              MESSAGE
2m7s                  Normal   SuccessfulCreate        ReplicaSet/grafana-69946c9bd6       Created pod: grafana-69946c9bd6-twvbj
2m7s                  Normal   WaitForFirstConsumer    PersistentVolumeClaim/grafana-pvc   waiting for first consumer to be created before binding
2m7s                  Normal   ScalingReplicaSet       Deployment/grafana                  Scaled up replica set grafana-69946c9bd6 to 1
2m6s (x2 over 2m7s)   Normal   ExternalProvisioning    PersistentVolumeClaim/grafana-pvc   waiting for a volume to be created, either by external provisioner "rancher.io/local-path" or manually created by system administrator
2m6s                  Normal   Provisioning            PersistentVolumeClaim/grafana-pvc   External provisioner is provisioning volume for claim "sven-grafana/grafana-pvc"
119s                  Normal   Scheduled               Pod/grafana-69946c9bd6-twvbj        Successfully assigned sven-grafana/grafana-69946c9bd6-twvbj to sven-test-control-plane
119s                  Normal   ProvisioningSucceeded   PersistentVolumeClaim/grafana-pvc   Successfully provisioned volume pvc-a8d03c24-dc5f-4b33-a0fc-14ddffba43d8
118s                  Normal   Pulling                 Pod/grafana-69946c9bd6-twvbj        Pulling image "grafana/grafana:latest"
```

It looks like it's not breaking yet. Why? 

```
$ docker exec -ti sven-test-control-plane crictl images | grep grafana
docker.io/grafana/grafana                       latest               c42c21cd0ebcb       123MB```

OK, that's nice, it was already there. It just needed a bit of patience:

```
$ kubectl get pod -n sven-grafana | grep grafana
grafana-69946c9bd6-twvbj   1/1     Running   0          5m37s
```

I set up a port-forward to this pod to access the grafana UI running on it:
```
$ kubectl port-forward pod/grafana-69946c9bd6-twvbj --namespace=sven-grafana --address 0.0.0.0 3000:3000
Forwarding from 0.0.0.0:3000 -> 3000
```

Nice - now I can access Grafana's UI from my browser. The default credentials are username `admin` and password `admin`. Grafana prompts me to change these to something safe.

It just sits there now - it doesn't seem to put any crazy load on my laptop just by existing:
```
$ kubectl top pods -n sven-grafana
NAME                       CPU(cores)   MEMORY(bytes)   
grafana-69946c9bd6-twvbj   2m           45Mi            
```

## Interacting with Grafana

Let's have a look at Grafana's logs

```
$ kubectl logs --namespace=sven-grafana deploy/grafana | head
logger=settings t=2024-07-19T09:56:50.903247162Z level=info msg="Starting Grafana" version=11.1.0 commit=5b85c4c2fcf5d32d4f68aaef345c53096359b2f1 branch=HEAD compiled=2024-07-19T09:56:50Z
logger=settings t=2024-07-19T09:56:50.903729863Z level=info msg="Config loaded from" file=/usr/share/grafana/conf/defaults.ini
logger=settings t=2024-07-19T09:56:50.903744989Z level=info msg="Config loaded from" file=/etc/grafana/grafana.ini
logger=settings t=2024-07-19T09:56:50.90375274Z level=info msg="Config overridden from command line" arg="default.paths.data=/var/lib/grafana"
logger=settings t=2024-07-19T09:56:50.903758935Z level=info msg="Config overridden from command line" arg="default.paths.logs=/var/log/grafana"
logger=settings t=2024-07-19T09:56:50.903764764Z level=info msg="Config overridden from command line" arg="default.paths.plugins=/var/lib/grafana/plugins"
logger=settings t=2024-07-19T09:56:50.903772414Z level=info msg="Config overridden from command line" arg="default.paths.provisioning=/etc/grafana/provisioning"
logger=settings t=2024-07-19T09:56:50.903779513Z level=info msg="Config overridden from command line" arg="default.log.mode=console"
logger=settings t=2024-07-19T09:56:50.903787359Z level=info msg="Config overridden from Environment variable" var="GF_PATHS_DATA=/var/lib/grafana"
logger=settings t=2024-07-19T09:56:50.903793391Z level=info msg="Config overridden from Environment variable" var="GF_PATHS_LOGS=/var/log/grafana"
```

What values are there for the `logger` field?
```
$ kubectl logs --namespace=sven-grafana deploy/grafana | grep -o -P '(?<=logger=)\S*' | sort | uniq | paste -s -d' '
cleanup context featuremgmt grafana-apiserver grafanaStorageLogger grafana.update.checker http.server infra.usagestats infra.usagestats.collector live.push_http local.finder migrator ngalert.multiorg.alertmanager ngalert.notifier.alertmanager ngalert.scheduler ngalert.state.manager plugin.angulardetectorsprovider.dynamic plugins.initialization plugins.registration plugin.store plugins.update.checker provisioning.alerting provisioning.dashboard query_data secrets settings sqlstore sqlstore.transactions ticker
```

Let's inspect a few. Entries with `logger=migrator`, as expected, seem related to db migrations:
```
$ kubectl logs --namespace=sven-grafana deploy/grafana | grep 'logger=migrator' | head -n 10
logger=migrator t=2024-07-19T09:56:50.906936942Z level=info msg="Locking database"
logger=migrator t=2024-07-19T09:56:50.906955141Z level=info msg="Starting DB migrations"
logger=migrator t=2024-07-19T09:56:50.90787523Z level=info msg="Executing migration" id="create migration_log table"
logger=migrator t=2024-07-19T09:56:50.909097182Z level=info msg="Migration successfully executed" id="create migration_log table" duration=1.221842ms
logger=migrator t=2024-07-19T09:56:50.915166827Z level=info msg="Executing migration" id="create user table"
logger=migrator t=2024-07-19T09:56:50.916469623Z level=info msg="Migration successfully executed" id="create user table" duration=1.301534ms
logger=migrator t=2024-07-19T09:56:50.920162075Z level=info msg="Executing migration" id="add unique index user.login"
logger=migrator t=2024-07-19T09:56:50.921202006Z level=info msg="Migration successfully executed" id="add unique index user.login" duration=1.041ms
logger=migrator t=2024-07-19T09:56:50.924820986Z level=info msg="Executing migration" id="add unique index user.email"
logger=migrator t=2024-07-19T09:56:50.926186591Z level=info msg="Migration successfully executed" id="add unique index user.email" duration=1.364673ms
```

`logger=sqlstore` gives us some details about the DB - as expected, Grafana is using a SQLite instance by default:
```
$ kubectl logs --namespace=sven-grafana deploy/grafana | grep 'logger=sqlstore\s' | head -n 10
logger=sqlstore t=2024-07-19T09:56:50.904587023Z level=info msg="Connecting to DB" dbtype=sqlite3
logger=sqlstore t=2024-07-19T09:56:50.904607391Z level=info msg="Creating SQLite database file" path=/var/lib/grafana/grafana.db
logger=sqlstore t=2024-07-19T09:57:00.281665366Z level=info msg="Created default admin" user=admin
logger=sqlstore t=2024-07-19T09:57:00.282020162Z level=info msg="Created default organization"
```

There are entries related to alerting that may become useful in the future; for now, there's nothing of interest there.

Finally, filtering on `logger=settings` gives some interesting output:
```
kubectl logs --namespace=sven-grafana deploy/grafana | grep 'logger=settings\s' | head -n 10
logger=settings t=2024-07-19T09:56:50.903247162Z level=info msg="Starting Grafana" version=11.1.0 commit=5b85c4c2fcf5d32d4f68aaef345c53096359b2f1 branch=HEAD compiled=2024-07-19T09:56:50Z
logger=settings t=2024-07-19T09:56:50.903729863Z level=info msg="Config loaded from" file=/usr/share/grafana/conf/defaults.ini
logger=settings t=2024-07-19T09:56:50.903744989Z level=info msg="Config loaded from" file=/etc/grafana/grafana.ini
logger=settings t=2024-07-19T09:56:50.90375274Z level=info msg="Config overridden from command line" arg="default.paths.data=/var/lib/grafana"
logger=settings t=2024-07-19T09:56:50.903758935Z level=info msg="Config overridden from command line" arg="default.paths.logs=/var/log/grafana"
logger=settings t=2024-07-19T09:56:50.903764764Z level=info msg="Config overridden from command line" arg="default.paths.plugins=/var/lib/grafana/plugins"
logger=settings t=2024-07-19T09:56:50.903772414Z level=info msg="Config overridden from command line" arg="default.paths.provisioning=/etc/grafana/provisioning"
logger=settings t=2024-07-19T09:56:50.903779513Z level=info msg="Config overridden from command line" arg="default.log.mode=console"
logger=settings t=2024-07-19T09:56:50.903787359Z level=info msg="Config overridden from Environment variable" var="GF_PATHS_DATA=/var/lib/grafana"
logger=settings t=2024-07-19T09:56:50.903793391Z level=info msg="Config overridden from Environment variable" var="GF_PATHS_LOGS=/var/log/grafana"
```

With this output we can see the effective configuration Grafana is running with.

## The filesystem

A volume is mounted at `/var/lib/grafana`.
Let's have a look around there:

```
$ kubectl exec -it -n sven-grafana grafana-69946c9bd6-twvbj -- tree /var/lib/grafana
/var/lib/grafana
├── csv
├── grafana.db
├── pdf
├── plugins
└── png

4 directories, 1 files
```

There's not much here apart from the (sqlite) database.

## The database

A quick try shows that the `sqlite3` command line tool is not available within the grafana container.
I can copy the database to my computer and inspect it locally, though:

```
$ kubectl cp sven-grafana/grafana-69946c9bd6-twvbj:/var/lib/grafana/grafana.db ./grafana.db
tar: removing leading '/' from member names

$ sqlite3 ./grafana.db
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .tables
alert                        library_element_connection 
alert_configuration          login_attempt              
alert_configuration_history  migration_log              
alert_image                  ngalert_configuration      
alert_instance               org                        
alert_notification           org_user                   
alert_notification_state     permission                 
alert_rule                   playlist                   
alert_rule_tag               playlist_item              
alert_rule_version           plugin_setting             
annotation                   preferences                
annotation_tag               provenance_type            
anon_device                  query_history              
api_key                      query_history_star         
builtin_role                 quota                      
cache_data                   role                       
cloud_migration              secrets                    
cloud_migration_run          seed_assignment            
correlation                  server_lock                
dashboard                    session                    
dashboard_acl                short_url                  
dashboard_provisioning       signing_key                
dashboard_public             sso_setting                
dashboard_snapshot           star                       
dashboard_tag                tag                        
dashboard_version            team                       
data_keys                    team_member                
data_source                  team_role                  
entity_event                 temp_user                  
file                         test_data                  
file_meta                    user                       
folder                       user_auth                  
kv_store                     user_auth_token            
library_element              user_role                  
```

There's not much data there, yet; Grafana did create a default admin user upon installation:

```
sqlite> select * from user;
1|0|admin|admin@localhost||aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa|bbbbbbbbbb|XXXXXXXXXX||1|1|0||2024-07-19 09:57:00|2024-07-19 10:12:30|0|2024-07-19 11:34:58|0|0|
```

I can revisit the database after some dashboards and maybe alerts have been setup. 
This is all for this first investigation.




 

