# ONLYOFFICE Docs for Kubernetes

This repository contains a set of files to deploy ONLYOFFICE Docs into a Kubernetes cluster or OpenShift cluster.

## Contents
- [Introduction](#introduction)
- [Deploy prerequisites](#deploy-prerequisites)
  * [1. Add Helm repositories](#1-add-helm-repositories)
  * [2. Install Persistent Storage](#2-install-persistent-storage)
  * [3. Deploy RabbitMQ](#3-deploy-rabbitmq)
  * [4. Deploy Redis](#4-deploy-redis)
  * [5. Deploy PostgreSQL](#5-deploy-postgresql)
  * [6. Deploy StatsD exporter](#6-deploy-statsd-exporter)
    + [6.1 Add Helm repositories](#61-add-helm-repositories)
    + [6.2 Installing Prometheus](#62-installing-prometheus)
    + [6.3 Installing StatsD exporter](#63-installing-statsd-exporter)
- [Deploy ONLYOFFICE Docs](#deploy-onlyoffice-docs)
  * [1. Deploy the ONLYOFFICE Docs license](#1-deploy-the-onlyoffice-docs-license)
  * [2. Deploy ONLYOFFICE Docs](#2-deploy-onlyoffice-docs)
  * [3. Uninstall ONLYOFFICE Docs](#3-uninstall-onlyoffice-docs)
  * [4. Parameters](#4-parameters)
  * [5. Configuration and installation details](#5-configuration-and-installation-details)
  * [5.1 Example deployment (optional)](#51-example-deployment-optional)
  * [5.2 Metrics deployment (optional)](#52-metrics-deployment-optional)
  * [5.3 Expose DocumentServer](#53-expose-documentserver)
    + [5.3.1 Expose DocumentServer via Service (HTTP Only)](#531-expose-documentserver-via-service-http-only)
    + [5.3.2 Expose DocumentServer via Ingress](#532-expose-documentserver-via-ingress)
    + [5.3.2.1 Installing the Kubernetes Nginx Ingress Controller](#5321-installing-the-kubernetes-nginx-ingress-controller)
    + [5.3.2.2 Expose DocumentServer via HTTP](#5322-expose-documentserver-via-http)
    + [5.3.2.3 Expose DocumentServer via HTTPS](#5323-expose-documentserver-via-https)
  * [6. Scale DocumentServer (optional)](#6-scale-documentserver-optional) 
      + [6.1 Manual scaling](#61-manual-scaling) 
  * [7. Update ONLYOFFICE Docs](#7-update-onlyoffice-docs)
      + [7.1 Updating using a script](#71-updating-using-a-script)
      + [7.2 Updating using helm upgrade](#72-updating-using-helm-upgrade)
  * [8. Shutdown ONLYOFFICE Docs (optional)](#8-shutdown-onlyoffice-docs-optional)
  * [9. Update ONLYOFFICE Docs license (optional)](#9-update-onlyoffice-docs-license-optional)
- [Using Grafana to visualize metrics (optional)](#using-grafana-to-visualize-metrics-optional)
  * [1. Deploy Grafana](#1-deploy-grafana)
    + [1.1 Deploy Grafana without installing ready-made dashboards](#11-deploy-grafana-without-installing-ready-made-dashboards)
    + [1.2 Deploy Grafana with the installation of ready-made dashboards](#12-deploy-grafana-with-the-installation-of-ready-made-dashboards)
  * [2 Access to Grafana via Ingress](#2-access-to-grafana-via-ingress)
  * [3. View gathered metrics in Grafana](#3-view-gathered-metrics-in-grafana)

## Introduction

- You must have a Kubernetes or OpenShift cluster installed. Please, checkout [the reference](https://kubernetes.io/docs/setup/) to set up Kubernetes. Please, checkout [the reference](https://docs.openshift.com/container-platform/4.7/installing/index.html) to setup OpenShift.
- You should also have a local configured copy of `kubectl`. See [this](https://kubernetes.io/docs/tasks/tools/install-kubectl/) guide how to install and configure `kubectl`.
- You should install Helm v3. Please follow the instruction [here](https://helm.sh/docs/intro/install/) to install it.
- If you use OpenShift, you can use both `oc` and `kubectl` to manage deploy. 
- If the installation of components external to ‘Docs’ is performed from Helm Chart in an OpenShift cluster, then it is recommended to install them from a user who has the `cluster-admin` role, in order to avoid possible problems with access rights. See [this](https://docs.openshift.com/container-platform/4.7/authentication/using-rbac.html) guide to add the necessary roles to the user.

## Deploy prerequisites

Open the repo directory

Note: When installing to an OpenShift cluster, you must apply the `SecurityContextConstraints` policy, which adds permission to run containers from a user whose `ID = 1001`.

To do this, run the following commands:
```
$ oc apply -f ./sources/scc/helm-components.yaml
$ oc adm policy add-scc-to-group scc-helm-components system:authenticated
```

### 1. Add Helm repositories

```bash
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo add stable https://charts.helm.sh/stable
$ helm repo update
```

### 2. Install Persistent Storage

Install NFS Server Provisioner

Note: When installing NFS Server Provisioner, Storage Classes - `NFS` is created. When installing to an OpenShift cluster, the user must have a role that allows you to create Storage Classes in the cluster. Read more [here](https://docs.openshift.com/container-platform/4.7/storage/dynamic-provisioning.html).

```bash
$ helm install nfs-server stable/nfs-server-provisioner \
  --set persistence.enabled=true \
  --set persistence.storageClass=PERSISTENT_STORAGE_CLASS \
  --set persistence.size=PERSISTENT_SIZE
```

- `PERSISTENT_STORAGE_CLASS` is a Persistent Storage Class available in your Kubernetes cluster.

  Persistent Storage Classes for different providers:
  - Amazon EKS: `gp2`
  - Digital Ocean: `do-block-storage`
  - IBM Cloud: Default `ibmc-file-bronze`. [More storage classes](https://cloud.ibm.com/docs/containers?topic=containers-file_storage)
  - Yandex Cloud: `yc-network-hdd` or `yc-network-ssd`. [More details](https://cloud.yandex.ru/docs/managed-kubernetes/operations/volumes/manage-storage-class)
  - minikube: `standard`

- `PERSISTENT_SIZE` is the total size of all Persistent Storages for the nfs Persistent Storage Class. You can express the size as a plain integer with one of these suffixes: `T`, `G`, `M`, `Ti`, `Gi`, `Mi`. For example: `9Gi`.

See more details about installing NFS Server Provisioner via Helm [here](https://github.com/helm/charts/tree/master/stable/nfs-server-provisioner#nfs-server-provisioner).

Configure a Persistent Volume Claim

Note: The default `nfs` Persistent Volume Claim is 8Gi. You can change it in the `values.yaml` file in the `persistence.storageClass` and `persistence.size` section. It should be less than `PERSISTENT_SIZE` at least by about 5%. It's recommended to use 8Gi or more for persistent storage for every 100 active users of ONLYOFFICE Docs.


### 3. Deploy RabbitMQ

To install RabbitMQ to your cluster, run the following command:

```bash
$ helm install rabbitmq bitnami/rabbitmq \
  --set metrics.enabled=false
```

Note: Set the `metrics.enabled=true` to enable exposing RabbitMQ metrics to be gathered by Prometheus.

See more details about installing RabbitMQ via Helm [here](https://github.com/bitnami/charts/tree/master/bitnami/rabbitmq#rabbitmq).

### 4. Deploy Redis

To install Redis to your cluster, run the following command:

```bash
$ helm install redis bitnami/redis \
  --set architecture=standalone \
  --set auth.enabled=false \
  --set image.tag=5.0.7-debian-10-r51 \
  --set metrics.enabled=false
```

Note: Set the `metrics.enabled=true` to enable exposing Redis metrics to be gathered by Prometheus.

See more details about installing Redis via Helm [here](https://github.com/bitnami/charts/tree/master/bitnami/redis).

### 5. Deploy PostgreSQL

Download the ONLYOFFICE Docs database scheme:

```bash
wget -O createdb.sql https://raw.githubusercontent.com/ONLYOFFICE/server/master/schema/postgresql/createdb.sql
```

Create a configmap from it:

```bash
$ kubectl create configmap init-db-scripts \
  --from-file=./createdb.sql
```

To install PostgreSQL to your cluster, run the following command:

```
$ helm install postgresql bitnami/postgresql \
  --set primary.initdb.scriptsConfigMap=init-db-scripts \
  --set auth.database=postgres \
  --set primary.persistence.size=PERSISTENT_SIZE \
  --set metrics.enabled=false
```

Here `PERSISTENT_SIZE` is a size for the PostgreSQL persistent volume. For example: `8Gi`.

It's recommended to use at least 2Gi of persistent storage for every 100 active users of ONLYOFFICE Docs.

Note: Set the `metrics.enabled=true` to enable exposing PostgreSQL metrics to be gathered by Prometheus.

See more details about installing PostgreSQL via Helm [here](https://github.com/bitnami/charts/tree/master/bitnami/postgresql#postgresql).

### 6. Deploy StatsD exporter
*This step is optional. You can skip step [#6](#6-deploy-statsd-exporter) at all if you don't want to run StatsD exporter*

#### 6.1 Add Helm repositories

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics
$ helm repo update
```

#### 6.2 Installing Prometheus

To install Prometheus to your cluster, run the following command:

```bash
$ helm install prometheus -f ./sources/extraScrapeConfigs.yaml prometheus-community/prometheus
```

See more details about installing Prometheus via Helm [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus).

#### 6.3 Installing StatsD exporter

To install StatsD exporter to your cluster, run the following command:
```
$ helm install statsd-exporter prometheus-community/prometheus-statsd-exporter \
  --set statsd.udpPort=8125 \
  --set statsd.tcpPort=8126 \
  --set statsd.eventFlushInterval=30000ms
```

See more details about installing Prometheus StatsD exporter via Helm [here](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-statsd-exporter).

To allow the StatsD metrics in ONLYOFFICE Docs, follow step [5.2](#52-metrics-deployment-optional)

## Deploy ONLYOFFICE Docs

Note: When installing to an OpenShift cluster, you must apply the `SecurityContextConstraints` policy, which adds permission to run containers from a user whose `ID = 101`.

To do this, run the following commands:
```
$ oc apply -f ./sources/scc/docs-components.yaml
$ oc adm policy add-scc-to-group scc-docs-components system:authenticated
```
Also, you must set the `securityContext.enabled` parameter to `true`:
```
$ helm install documentserver ./ --set securityContext.enabled=true
```
### 1. Deploy the ONLYOFFICE Docs license

If you have a valid ONLYOFFICE Docs license, add it to the directory `sources/license`.

Note: The source license file name should be 'license.lic' because this name would be used as a field in the created secret.

- If you have no ONLYOFFICE Docs license, leave the directory empty.

### 2. Deploy ONLYOFFICE Docs

To deploy DocumentServer with the release name `documentserver`:

```bash

$ helm install documentserver ./
```

The command deploys DocumentServer on the Kubernetes cluster in the default configuration. The Parameters section lists the parameters that can be configured during installation.

### 3. Uninstall ONLYOFFICE Docs

To uninstall/delete the `documentserver` deployment:

```bash
$ helm delete documentserver

```

The command removes all the Kubernetes components associated with the chart and deletes the release.

### 4. Parameters

| Parameter                             | Description                                                                                                              | Default                                    |
|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| connections.dbHost                    | IP address or the name of the database                                                                                   | postgresql                                 |
| connections.dbUser                    | database user                                                                                                            | postgres                                   |
| connections.dbPort                    | database server port number                                                                                              | 5432                                       |
| connections.dbName                    | Name of the PostgreSQL database to which the application will connect                                                    | postgres                                   |
| connections.dbPassword                | PostgreSQL user password. If set to, it takes priority over the `connections.dbExistingSecret`                           | ""                                         |
| connections.dbSecretKeyName           | The name of the key that contains the PostgreSQL user password                                                           | postgres-password                          |
| connections.dbExistingSecret          | Name of existing secret to use for PostgreSQL passwords. Must contain the key specified in `connections.dbSecretKeyName` | postgresql                                 |
| connections.redistHost                | IP address or the name of the redis host                                                                                 | redis-master                               |
| connections.amqpHost                  | IP address or the name of the message-broker                                                                             | rabbitmq                                   |
| connections.amqpUser                  | messabe-broker user                                                                                                      | user                                       |
| connections.amqpProto                 | messabe-broker protocol                                                                                                  | amqp                                       |
| connections.amqpPassword              | RabbitMQ user password. If set to, it takes priority over the `connections.amqpExistingSecret`                           | ""                                         |
| connections.amqpSecretKeyName         | The name of the key that contains the RabbitMQ user password                                                             | rabbitmq-password                          |
| connections.amqpExistingSecret        | Name of existing secret to use for RabbitMQ passwords. Must contain the key specified in `connections.amqpSecretKeyName` | rabbitmq                                   |
| persistence.existingClaim             | Name of an existing PVC to use. If not specified, a PVC named "ds-files" will be created                                 | ""                                         |
| persistence.storageClass              | storage class name                                                                                                       | nfs                                        |
| persistence.size                      | storage volume size                                                                                                      | 8Gi                                        |
| log.level                             | Defines the type and severity of a logged event                                                                          | WARN                                       |
| metrics.enabled                       | Statsd installation                                                                                                      | false                                      |
| metrics.host                          | Defines StatsD listening host                                                                                            | statsd-exporter-prometheus-statsd-exporter |
| metrics.port                          | Defines StatsD listening port                                                                                            | 8125                                       |
| metrics.prefix                        | Defines StatsD metrics prefix for backend services                                                                       | ds.                                        |
| example.enabled                       | Choise of example installation                                                                                           | false                                      |
| example.containerImage                | example container image name                                                                                             | onlyoffice/docs-example:6.4.2.6            |
| example.imagePullPolicy               | Example container image pull policy                                                                                      | IfNotPresent                               |
| example.resources.requests.memory     | memory request                                                                                                           | 128Mi                                      |
| example.resources.requests.cpu        | cpu request                                                                                                              | 100m                                       |
| example.resources.limits.memory       | memory limit                                                                                                             | 128Mi                                      |
| example.resources.limits.cpu          | cpu limit                                                                                                                | 250m                                       |
| extraConf.configMap                   | The name of the ConfigMap containing the json file that override the default values                                      | ""                                         |
| extraConf.filename                    | The name of the json file that contains custom values. Must be the same as the `key` name in `extraConf.ConfigMap`       | local.json                                 |
| antiAffinity.type                     | Types of Pod antiaffinity. Allowed values: `soft` or `hard`                                                              | soft                                       |
| antiAffinity.topologyKey              | Node label key to match                                                                                                  | kubernetes.io/hostname                     |
| antiAffinity.weight                   | Priority when selecting node. It is in the range from 1 to 100                                                           | 100                                        |
| docservice.replicas                   | docservice replicas quantity                                                                                             | 2                                          |
| docservice.containerImage             | docservice container image name                                                                                          | onlyoffice/docs-docservice-de:6.4.2.6      |
| docservice.imagePullPolicy            | Docservice container image pull policy                                                                                   | IfNotPresent                               |
| docservice.resources.requests.memory  | memory request                                                                                                           | 256Mi                                      |
| docservice.resources.requests.cpu     | cpu request                                                                                                              | 100m                                       |
| docservice.resources.limits.memory    | memory limit                                                                                                             | 2Gi                                        |
| docservice.resources.limits.cpu       | cpu limit                                                                                                                | 1000m                                      |
| docservice.readinessProbeEnabled      | Enable readinessProbe for docservice                                                                                     | true                                       |
| docservice.livenessProbeEnabled       | Enable livenessProbe for docservice                                                                                      | true                                       |
| docservice.startupProbeEnabled        | Enable startupProbe for docservice                                                                                       | true                                       |
| proxy.gzipProxied                     | Defines the nginx config [gzip_proxied](https://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_proxied) directive | off                                        |
| proxy.proxyContainerImage             | docservice proxy container image name                                                                                    | onlyoffice/docs-proxy-de:6.4.2.6           |
| proxy.imagePullPolicy                 | Docservice proxy container image pull policy                                                                             | IfNotPresent                               |
| proxy.resources.requests.memory       | memory request                                                                                                           | 256Mi                                      |
| proxy.resources.requests.cpu          | cpu request                                                                                                              | 100m                                       |
| proxy.resources.limits.memory         | memory limit                                                                                                             | 2Gi                                        |
| proxy.resources.limits.cpu            | cpu limit                                                                                                                | 1000m                                      |
| proxy.livenessProbeEnabled            | Enable livenessProbe for proxy                                                                                           | true                                       |
| proxy.startupProbeEnabled             | Enable startupProbe for proxy                                                                                            | true                                       |
| converter.replicas                    | converter replicas quantity                                                                                              | 2                                          |
| converter.containerImage              | converter container image name                                                                                           | onlyoffice/docs-converter-de:6.4.2.6       |
| converter.imagePullPolicy             | Converter container image pull policy                                                                                    | IfNotPresent                               |
| converter.requests.memory             | memory request                                                                                                           | 256Mi                                      |
| converter.requests.cpu                | cpu request                                                                                                              | 100m                                       |
| converter.limits.memory               | memory limit                                                                                                             | 2Gi                                        |
| converter.limits.cpu                  | cpu limit                                                                                                                | 1000m                                      |
| jwt.enabled                           | jwt enabling parameter                                                                                                   | true                                       |
| jwt.secret                            | jwt secret                                                                                                               | MYSECRET                                   |
| jwt.header                            | Defines the http header that will be used to send the JSON Web Token                                                     | Authorization                              |
| jwt.inBody                            | Specifies the enabling the token validation in the request body to the ONLYOFFICE Docs                                   | false                                      |
| service.type                          | documentserver service type                                                                                              | ClusterIP                                  |
| service.port                          | documentserver service port                                                                                              | 8888                                       |
| ingress.enabled                       | installation of ingress service                                                                                          | false                                      |
| ingress.host                          | Ingress hostname for the documentserver ingress                                                                          | ""                                         |
| ingress.ssl.enabled                   | installation ssl for ingress service                                                                                     | false                                      |
| ingress.ssl.secret                    | secret name for ssl                                                                                                      | tls                                        |
| grafana_ingress.enabled               | installation grafana of ingress service                                                                                  | false                                      |
| securityContext.enabled               | Enable security context for the pods                                                                                     | false                                      |
| securityContext.converter.runAsUser   | Set converter containers' Security Context runAsUser                                                                     | 101                                        |
| securityContext.converter.runAsGroup  | Set converter containers' Security Context runAsGroup                                                                    | 101                                        |
| securityContext.docservice.runAsUser  | Set docservice containers' Security Context runAsUser                                                                    | 101                                        |
| securityContext.docservice.runAsGroup | Set docservice containers' Security Context runAsGroup                                                                   | 101                                        |
| securityContext.example.runAsUser     | Set example containers' Security Context runAsUser                                                                       | 1001                                       |
| securityContext.example.runAsGroup    | Set example containers' Security Context runAsGroup                                                                      | 1001                                       |

Specify each parameter using the --set key=value[,key=value] argument to helm install. For example,

```bash
$ helm install documentserver ./ --set ingress.enabled=true,ingress.ssl.enabled=true,ingress.host=example.com
```

This command gives expose documentServer via HTTPS.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```bash
$ helm install documentserver -f values.yaml ./
```

> **Tip**: You can use the default [values.yaml](values.yaml)

### 5. Configuration and installation details

### 5.1 Example deployment (optional)

To deploy the example, set the `example.enabled` parameter to true:

```bash
$ helm install documentserver ./ --set example.enabled=true
```

### 5.2 Metrics deployment (optional)
To deploy metrics, set `metrics.enabled` to true:

```bash
$ helm install documentserver ./ --set metrics.enabled=true
```
If you want to use nginx ingress, set `grafana_ingress.enabled` to true:

```bash
$ helm install documentserver ./ --set grafana_ingress.enabled=true
```

### 5.3 Expose DocumentServer

#### 5.3.1 Expose DocumentServer via Service (HTTP Only)
*You should skip step[#5.3.1](#531-expose-documentserver-via-service-http-only) if you are going to expose DocumentServer via HTTPS*

This type of exposure has the least overheads of performance, it creates a loadbalancer to get access to DocumentServer.
Use this type of exposure if you use external TLS termination, and don't have another WEB application in the k8s cluster.

To expose DocumentServer via service, set the `service.type` parameter to LoadBalancer:

```bash
$ helm install documentserver ./ --set service.type=LoadBalancer,service.port=80

```


Run the following command to get the `documentserver` service IP:

```bash
$ kubectl get service documentserver -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

After that, ONLYOFFICE Docs will be available at `http://DOCUMENTSERVER-SERVICE-IP/`.

If the service IP is empty, try getting the `documentserver` service hostname:

```bash
$ kubectl get service documentserver -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"
```

In this case, ONLYOFFICE Docs will be available at `http://DOCUMENTSERVER-SERVICE-HOSTNAME/`.


#### 5.3.2 Expose DocumentServer via Ingress

#### 5.3.2.1 Installing the Kubernetes Nginx Ingress Controller

To install the Nginx Ingress Controller to your cluster, run the following command:

```bash
$ helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true,controller.replicaCount=2
```

Note: To install Nginx Ingress with the same parameters and to enable exposing ingress-nginx metrics to be gathered by Prometheus, run the following command:

```bash
$ helm install nginx-ingress -f ./sources/ingress_values.yaml ingress-nginx/ingress-nginx
```

See more detail about installing Nginx Ingress via Helm [here](https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx).

#### 5.3.2.2 Expose DocumentServer via HTTP

*You should skip step[5.3.2.2](#5322-expose-documentserver-via-http) if you are going to expose DocumentServer via HTTPS*

This type of exposure has more overheads of performance compared with exposure via service, it also creates a loadbalancer to get access to DocumentServer. 
Use this type if you use external TLS termination and when you have several WEB applications in the k8s cluster. You can use the one set of ingress instances and the one loadbalancer for those. It can optimize the entry point performance and reduce your cluster payments, cause providers can charge a fee for each loadbalancer.

To expose DocumentServer via ingress HTTP, set the `ingress.enabled` parameter to true:

```bash
$ helm install documentserver ./ --set ingress.enabled=true

```

Run the following command to get the `documentserver` ingress IP:

```bash
$ kubectl get ingress documentserver -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

After that, ONLYOFFICE Docs will be available at `http://DOCUMENTSERVER-INGRESS-IP/`.

If the ingress IP is empty, try getting the `documentserver` ingress hostname:

```bash
$ kubectl get ingress documentserver -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"
```

In this case, ONLYOFFICE Docs will be available at `http://DOCUMENTSERVER-INGRESS-HOSTNAME/`.

#### 5.3.2.3 Expose DocumentServer via HTTPS

This type of exposure allows you to enable internal TLS termination for DocumentServer.

Create the `tls` secret with an ssl certificate inside.

Put the ssl certificate and the private key into the `tls.crt` and `tls.key` files and then run:

```bash
$ kubectl create secret generic tls \
  --from-file=./tls.crt \
  --from-file=./tls.key
```

```bash
$ helm install documentserver ./ --set ingress.enabled=true,ingress.ssl.enabled=true,ingress.host=example.com

```

Run the following command to get the `documentserver` ingress IP:

```bash
$ kubectl get ingress documentserver -o jsonpath="{.status.loadBalancer.ingress[*].ip}"
```

If the ingress IP is empty, try getting the `documentserver` ingress hostname:

```bash
$ kubectl get ingress documentserver -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"
```

Associate the `documentserver` ingress IP or hostname with your domain name through your DNS provider.

After that, ONLYOFFICE Docs will be available at `https://your-domain-name/`.

### 6. Scale DocumentServer (optional)

*This step is optional. You can skip step [6](#6-scale-documentserver-optional) at all if you want to use default deployment settings.*

#### 6.1 Manual scaling
The `docservice` and `converter` deployments consist of 2 pods each other by default.

To scale the `docservice` deployment, use the following command:

```bash
$ kubectl scale -n default deployment docservice --replicas=POD_COUNT
```

where `POD_COUNT` is а number of the `docservice` pods.

Do the same to scale the `converter` deployment:

```bash
$ kubectl scale -n default deployment converter --replicas=POD_COUNT
```

### 7. Update ONLYOFFICE Docs

There are two possible options for updating ONLYOFFICE Docs, which are presented below.

#### 7.1 Updating using a script

To perform the update, run the following script:

```bash
$ ./sources/scripts/update-ds.sh -dv [DOCUMENTSERVER_VERSION] -ns <NAMESPACE>
```

Where:
 - `dv` - new version of docker images for ONLYOFFICE Docs.
 - `ns` - Namespace where ONLYOFFICE Docs is installed. If not specified, the default value will be used: `default`.

For example:
```bash
$ ./sources/scripts/update-ds.sh -dv 7.0.0.132 -ns onlyoffice
```

#### 7.2 Updating using helm upgrade

It's necessary to set the parameters for updating. For example,

```bash
$ helm upgrade documentserver ./ \
  --set docservice.containerImage=[image]:[version]
  ```
  
  > **Note**: also need to specify the parameters that were specified during installation
  
  Or modify the values.yaml file and run the command:
  
  ```bash
  $ helm upgrade documentserver ./
  ```
  
Running the helm upgrade command runs a hook that shuts down the documentserver and cleans up the database. This is needed when updating the version of documentserver. The default hook execution time is 300s.
The execution time can be changed using --timeout [time], for example

```bash
helm upgrade documentserver ./ --timeout 15m
```

If you want to update any parameter other than the version of the DocumentServer, then run the `helm upgrade` command without hooks, for example:

```bash
helm upgrade documentserver ./ --set jwt.enabled=false --no-hooks
```
  
### 8. Shutdown ONLYOFFICE Docs (optional)

To perform the shutdown, run the following script:

```bash
$ ./sources/scripts/shutdown-ds.sh -ns <NAMESPACE>
```

Where:
 - `ns` - Namespace where ONLYOFFICE Docs is installed. If not specified, the default value will be used: `default`.

For example:
```bash
$ ./sources/scripts/shutdown-ds.sh -ns onlyoffice
```

### 9. Update ONLYOFFICE Docs license (optional)

In order to update the license, you need to perform the following steps:
 - Place the license.lic file containing the new key in some directory
 - Run the following commands:
   ```bash
   $ kubectl delete secret license
   $ kubectl create secret generic license --from-file=path/to/license.lic
   ```
 - Restart `docservice` and `converter` pods. For example, using the following command:
   ```bash
   $ kubectl delete pod converter-*** docservice-***
   ```

## Using Grafana to visualize metrics (optional)

*This step is optional. You can skip this section if you don't want to install Grafana*

### 1. Deploy Grafana

Note: It is assumed that step [#6.2](#62-installing-prometheus) has already been completed.

#### 1.1 Deploy Grafana without installing ready-made dashboards

*You should skip step [#2.1](#21-deploy-grafana-without-installing-ready-made-dashboards) if you want to Deploy Grafana with the installation of ready-made dashboards*

To install Grafana to your cluster, run the following command:

```bash
$ helm install grafana bitnami/grafana \
  --set service.ports.grafana=80 \
  --set config.useGrafanaIniFile=true \
  --set config.grafanaIniConfigMap=grafana-ini \
  --set datasources.secretName=grafana-datasource
```

#### 1.2 Deploy Grafana with the installation of ready-made dashboards

Run the `./sources/metrics/get_dashboard.sh` script, which will download ready-made dashboards in the `JSON` format from the Grafana [website](https://grafana.com/grafana/dashboards),
make the necessary edits to them and create a configmap from them. A dashboard will also be added to visualize metrics coming from the DocumentServer (it is assumed that step [#6](#6-deploy-statsd-exporter) has already been completed).

```
$ ./sources/metrics/get_dashboard.sh
```

To install Grafana to your cluster, run the following command:

```bash
$ helm install grafana bitnami/grafana \
  --set service.ports.grafana=80 \
  --set config.useGrafanaIniFile=true \
  --set config.grafanaIniConfigMap=grafana-ini \
  --set datasources.secretName=grafana-datasource \
  --set dashboardsProvider.enabled=true \
  --set dashboardsConfigMaps[0].configMapName=dashboard-node-exporter \
  --set dashboardsConfigMaps[0].fileName=dashboard-node-exporter.json \
  --set dashboardsConfigMaps[1].configMapName=dashboard-deployment \
  --set dashboardsConfigMaps[1].fileName=dashboard-deployment.json \
  --set dashboardsConfigMaps[2].configMapName=dashboard-redis \
  --set dashboardsConfigMaps[2].fileName=dashboard-redis.json \
  --set dashboardsConfigMaps[3].configMapName=dashboard-rabbitmq \
  --set dashboardsConfigMaps[3].fileName=dashboard-rabbitmq.json \
  --set dashboardsConfigMaps[4].configMapName=dashboard-postgresql \
  --set dashboardsConfigMaps[4].fileName=dashboard-postgresql.json \
  --set dashboardsConfigMaps[5].configMapName=dashboard-nginx-ingress \
  --set dashboardsConfigMaps[5].fileName=dashboard-nginx-ingress.json \
  --set dashboardsConfigMaps[6].configMapName=dashboard-documentserver \
  --set dashboardsConfigMaps[6].fileName=documentserver-statsd-exporter.json
```

After executing this command, the following dashboards will be imported into Grafana:

  - Node Exporter
  - Deployment Statefulset Daemonset
  - Redis Dashboard for Prometheus Redis Exporter
  - RabbitMQ-Overview
  - PostgreSQL Database
  - NGINX Ingress controller
  - DocumentServer

See more details about installing Grafana via Helm [here](https://github.com/bitnami/charts/tree/master/bitnami/grafana).

### 2 Access to Grafana via Ingress

Note: It is assumed that step [#5.3.2.1](#5321-installing-the-kubernetes-nginx-ingress-controller) has already been completed.

If DocumentServer was installed with the parameter `grafana_ingress.enabled=true` (step [#5.2](#52-metrics-deployment-optional)) then access to Grafana will be at: `http://INGRESS-ADDRESS/grafana/`

If Ingres was installed using a secure connection (step [#5.3.2.3](#5323-expose-documentserver-via-https)), then access to Grafana will be at: `https://your-domain-name/grafana/`

### 3. View gathered metrics in Grafana

Go to the address `http(s)://your-domain-name/grafana/`

Login - admin

To get the password, run the following command:
```
$ kubectl get secret grafana-admin --namespace default -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode
```

In the dashboard section, you will see the added dashboards that will display the metrics received from Prometheus.
