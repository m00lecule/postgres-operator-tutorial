# postgres-operator-tutorial

## requirements
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) - `0.11.0`
- docker - `20.10.7`
- psql 

init zalando-operator submodules from [official repository](https://github.com/zalando/postgres-operator)
```console
git submodule init
```

# setup K8S cluster

During this tutorial we will be using `kind` cluster, with following configuration:
- 1 master node
- 3 worker nodes

## 1.1. verify if any cluster is available:
```zsh
kind get clusters
```
## 1.2. create cluster
```zsh
kind create cluster --config manifests/kind/cluster.yml
```

## 1.3. verify if `kubectl` context is switched to proper cluster

```zsh
$ kubectl get nodes                                            
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   54s   v1.21.1
kind-worker          Ready    <none>                 25s   v1.21.1
kind-worker2         Ready    <none>                 25s   v1.21.1
kind-worker3         Ready    <none>                 25s   v1.21.1
```

# zalando-operator setup

```console
cd postgres-operator
kubectl create -f manifests/configmap.yaml
kubectl create -f manifests/operator-service-account-rbac.yaml
kubectl create -f manifests/postgres-operator.yaml
kubectl create -f manifests/api-service.yaml 
```
Check if operator is running:

```console
kubectl get pod -l name=postgres-operator
NAME                                 READY   STATUS    RESTARTS   AGE
postgres-operator-55b8549cff-84gp7   1/1     Running   0          118s
```

## deploy the oparator UI

```console
kubectl apply -f postgres-operator/ui/manifests/
```

verify if its running:
```console
kubectl get pod -l name=postgres-operator-ui
```

port-forward the operator UI service to `localhost`

```console
kubectl port-forward svc/postgres-operator-ui 8081:80
```

then go to `http://localhost:8081` and explore


## deploy first PostgreSQL cluster

```console
kubectl apply -f manifests/postgres-12-cluster.yml
```

## connect to PostgreSQL master

port-forward the master node to `localhost`
```console
export PGPASSWORD=$(kubectl get secret postgres.acid-first.credentials -o 'jsonpath={.data.password}' | base64 -d)

export PGMASTER=$(kubectl get pods -o jsonpath={.items..metadata.name} -l application=spilo,cluster-name=acid-first,spilo-role=master -n default)

kubectl port-forward $PGMASTER 6432:5432 -n default
```
connect to master instance
```console
psql -U postgres -h localhost -p 6432
psql (12.7 (Ubuntu 12.7-0ubuntu0.20.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.
```

verify the instance version
```console
SELECT version();
```
cerate sample table within new database
```zsh
CREATE DATABASE ssu_operator_labs; 
 \c ssu_operator_labs 
 CREATE TABLE ssu_tab (random_str text);

 #list all tables that matches following regex
 \dt ssu*
```
seed sample database with data
```console
# seed the values
INSERT INTO ssu_tab (random_str) VALUES ('Magiera'), ('Dygas'), ('Bodera');

 SELECT * FROM ssu_tab;
```

verify if the data was replicated to the slave cluster
```console
kubectl port-forward acid-first-1 6433:5432 -n default
```
select values 
```console
 \c ssu_operator_labs 
 SELECT * FROM ssu_tab;
```

## upgrade an existing PostgreSQL cluster

open one console and watch all pods
```console
kubectl get pods  -w 
```
and wait till all pods will be running

change the `spec.postgresql.version` from `12` to `13` in `postgres/v12-cluster.yml` (example in `postgres/v13-cluster.yml`) and apply edited config:

```console
kubectl apply -f manifests/postgres-13-cluster.yml 
```
### verify the upgrade process on master node

```console
SELECT version();
```