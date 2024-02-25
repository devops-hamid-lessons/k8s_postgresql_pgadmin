# How to deploy and manage PostgreSQL clusters in K8s

This tutorial explains how to use `zalando-postgres-operator` and deploy and manage a postgresql cluster in k8s in a scalable fashion. 
We will also deploy `pgAdmin4` as a graphical tool to work with DBs and tables in postgresql.

Steps taken during this tutorial:
1. deployment of `zalando-postgres-operator` using its helm chart
2. deployment of `postgres-operator-ui`, which is a graphical tool for lunching postgresql clusters
3. An example postgresql cluster deployment using a .yaml file
4. postgresql internal and external connection strings notes
5. `pgAdmin4` deployment

## Step 1: `zalando-postgres-operator` installation
### 1.1. Installation
First add helm repo of `zalando` and then install it:
``` 
helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm install postgres-operator postgres-operator-charts/postgres-operator
```
* use `kubectl get pod -l  "app.kubernetes.io/name=postgres-operator"` to see if the operator is created and running.
* use `helm show values postgres-operator-charts/postgres-operator` to see available configurable parameters for `zalando` helm chart such as number of operator replicas. Above we install `zalando` by its default values and do not specify a value file.

## Step 2: `postgres-operator-ui` installation
### 2.1. Installtion
Similarly install `postgres-operator-ui`:
```
helm repo add postgres-operator-ui-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator-ui
helm install postgres-operator-ui postgres-operator-ui-charts/postgres-operator-ui
```
* use `kubectl get pods -l "app.kubernetes.io/name=postgres-operator-ui"` to check if it is running.

### 2.2. Web access to UI
`postgres-operator-ui` is the default created service for UI. However it is a `ClusterIP` type. 
So, for external web access, we need to adopt one of the following methods:
1. Creating a `loadbalancer` service 
2. Using an `ingres`

In this tutorial, we test the first approach.

To do so, use `expose` command to create a new `LoadBalancer` service, exposing the internal port `8081` on pod `postgres-operator-ui-788847ff96-gk5wn` as a loadbalancer service with `88` as external port. Note that you need to replace the pod name with yours.
```
 kubectl expose pod postgres-operator-ui-788847ff96-gk5wn --type=LoadBalancer --target-port=8081 --port 88 --name=operator-ui-server-ext
```
* You could also create a service yaml file for this work.
* Now you can access `postgres-operator-ui` using the external ip of operator-ui-server-ext and port 88: `ip:88`
![alt text](pic/postgreUI.JPG)
  
## Step 3: An example postgresql cluster deployment using a .yaml file
Option 1. If you refer to the `postgres-operator-ui`, it will help you to deploy or destroy a postgresql cluster graphically and much straightforwardly. As pic above, Just Check it, needless to explain more.

Option 2. Write and deploy a yaml file by your self.
* For this option we need to create a yaml file of type `postgresql` and provide required params.
* `postgresql` is a CRD type defined when installing operator. `zalando` operator knows this type and when you apply a yaml file of it, it will grab that yaml file and deploy the defined cluster for you.
* See the provided example file named `postgres_cluster.yaml`.
```
kind: "postgresql"
apiVersion: "acid.zalan.do/v1"

metadata:
  name: "test-cluster"
  namespace: "default"
  labels:
    team: acid

spec:
  teamId: "acid"
  postgresql:
    version: "15"
  numberOfInstances: 3
  volume:
    size: "10Gi"


  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 500m
      memory: 500Mi
```  
* This file will create a cluster having 3 replicas each of them with a storage of size 10Gi. Now, `zalando` operator will take all responsibilities of this stateful app deployment.
* For other configurable params refer to this link: `https://github.com/zalando/postgres-operator/blob/master/manifests/complete-postgres-manifest.yaml`


## Step 4. Postgresql internal and external connection strings notes
Until now, we have deployed a sample cluster. Now we want to connect to the cluster.

### 4.1. Install `psql`
```
sudo apt-get install -y postgresql-client
```
### 4.2. Obtain postgresql password:
```
# user: postgres
# password:
 kubectl get secret postgres.test-cluster.credentials.postgresql.acid.zalan.do   -o 'jsonpath={.data.password}' | base64 -d
```
### 4.3. Internal connection to db 
- Internal connection means inside pods of kubernetes cluster. So you first need to log in to one of cluster pods or create a new one and then install `psql` inside it. If you use postgres pods you do not need to install `psql`
- Connection string format is `postgres://user:pass@servicename:5432/db-name`
- replace service name and pass 
- service name is the same as the name of `posgresql` cluster (`kubectl get svc`)
```
Example:
psql postgres://postgres:UB9GHhXyjdbvXhNkyC7A8JiA9bjqOdHjPyOFud5y@test-cluster:5432/postgres
```
* this will connect you to postgres db, which is the default db.

### 4.4. External connection to db
```
Example:
psql postgres://postgres:pass@ip:port/db-name
```
* Note that this time you need to provide an ip address instead of the service name. To do so, you need to expose postgres master pod as LoadBalancer service and use the related ip and port.
* It is easy, see example below:
```
kubectl expose pod test-cluster-0  --type=LoadBalancer --target-port=5432 --name=test-cluster-ext
psql postgres://postgres:vqRkdMZLDWJyjPoiiW0t7tUB9GHhXyjdbvXhNkyC7A8JiAFud5y1n@143.42.223.192:5432/postgres
```


## Step 5. `pgAdmin4` deployment
### 5.1. installation
```
helm repo add runix https://helm.runix.net
helm install pgadmin runix/pgadmin4
```
### 5.2. Like previous ones, expose its service or pod as a LoadBalancer or use an ingress to access its dashboard.
```
example:
kubectl expose pod pgadmin-pgadmin4-75d65dd8d4-k9kbf   --type=LoadBalancer --target-port=80 --name=pgadmin-ext
```
### 5.3. Follow the steps below in its dashboard to add your cluster
```
1. Open pgAdmin4 dashboard (ip:port, default username: chart@domain.com, default pass: SuperSecret)
2. Right click on "Servers" in Browser panel and select register.
3. Enter some meaningful name for Server eg: "PostgreSQL"
4. Click on "Connection" tab
5. Enter "test-cluster" (using "kube get svc" find service name. It is the same as the name of your postgres yaml file.) 
6. Enter "Port" (default port for PostgreSQL server is "5432")
7. Enter "Maintenance Database" name (default is "postgres")
8. Enter "User Name" (default user is "postgres")
9. Enter postgres "Password" (obtained in step 4.2)
```
![alt text](pic/pgadmin.JPG)
### Now we have a postgresql cluster plus all versatile tools to work with it.