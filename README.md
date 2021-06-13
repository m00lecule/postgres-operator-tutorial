# postgres-operator-tutorial

## requirements
- kind - `0.11.0`
- docker - `20.10.7`
- psql 

init zalando-operator submodules from [official repository](https://github.com/zalando/postgres-operator)
```bash
git submodule init
```

# setup k8s cluster on localhost

During this tutorial we will be using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) to provision k8s cluster on `docker`, with following configuration:
- 1 master node
- 3 worker nodes

verify if any cluster is available
```zsh
kind get clusters
```
create k8s cluster
```bash
kind create cluster --config manifests/kind/cluster.yml
```

verify if `kubectl` context is switched to proper cluster

```zsh
$ kubectl get nodes                                            
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   54s   v1.21.1
kind-worker          Ready    <none>                 25s   v1.21.1
kind-worker2         Ready    <none>                 25s   v1.21.1
kind-worker3         Ready    <none>                 25s   v1.21.1
```

# setup zalando-operator

## provision PostgreSQL operator

```bash
cd postgres-operator
kubectl create -f manifests/configmap.yaml
kubectl create -f manifests/operator-service-account-rbac.yaml
kubectl create -f manifests/postgres-operator.yaml
kubectl create -f manifests/api-service.yaml 
```

check if operator is running

```bash
kubectl get pod -l name=postgres-operator
NAME                                 READY   STATUS    RESTARTS   AGE
postgres-operator-55b8549cff-84gp7   1/1     Running   0          118s
```

## deploy the oparator UI

```bash
kubectl apply -f postgres-operator/ui/manifests/
```

verify if its running
```bash
kubectl get pod -l name=postgres-operator-ui
```

port-forward the operator UI service to `localhost`

```bash
kubectl port-forward svc/postgres-operator-ui 8081:80
```

then go to `http://localhost:8081` and explore


## deploy first PostgreSQL cluster

```bash
kubectl apply -f manifests/postgres-12-cluster.yml
```

## connect to PostgreSQL master

port-forward the master node to `localhost`
```bash
export PGPASSWORD=$(kubectl get secret postgres.acid-first.credentials -o 'jsonpath={.data.password}' | base64 -d)

export PGMASTER=$(kubectl get pods -o jsonpath={.items..metadata.name} -l application=spilo,cluster-name=acid-first,spilo-role=master -n default)

kubectl port-forward $PGMASTER 6432:5432 -n default
```
connect to master instance
```bash
$ psql -U postgres -h localhost -p 6432
psql (12.7 (Ubuntu 12.7-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```

verify the instance version
```sql
SELECT version();
```
cerate sample table within new database
```sql
CREATE DATABASE ssu_operator_labs; 

#connect to created database
\c ssu_operator_labs 

CREATE TABLE ssu_tab (random_str text);

#list all tables that matches following regex
\dt ssu*
```
seed sample database with data
```sql
# seed the values
INSERT INTO ssu_tab (random_str) VALUES ('Magiera'), ('Dygas'), ('Bodera');

SELECT * FROM ssu_tab;

 random_str 
------------
 Magiera
 Dygas
 Bodera
(3 rows)
```

verify if the data was replicated to the slave cluster
```bash
kubectl port-forward acid-first-1 6433:5432 -n default
```
and then inspect the values
```sql
\c ssu_operator_labs 
SELECT * FROM ssu_tab;
```

## upgrade an existing PostgreSQL cluster version

open another terminal session and watch pods status
```
kubectl get pods  -w 
```

change the `spec.postgresql.version` from `12` to `13` in `postgres/v12-cluster.yml` (example in `postgres/v13-cluster.yml`) and apply edited config:

```bash
kubectl apply -f manifests/postgres-13-cluster.yml 
```

and **wait** till all pods will be in running state again

 then verify the version on master node

```sql
SELECT version();
```