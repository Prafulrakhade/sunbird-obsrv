# OBSRV K8S Cluster Setup using RKE

## Overview
This is a guide to set up Kubernetes cluster on Virtual Machines using [RKE](https://www.rancher.com/products/rke). This cluster would host all OBSRV modules.

## Prerequisites
Obsrv runs completely on a Kubernetes cluster. A completely functional Kubernetes cluster is expected for a seamless Obsrv installation.

* Ansible
* Docker
* RKE (rke version: v1.3.10)
* Istioctl (istioctl version: v1.15.0)
* Helm

## Create Rancher Cluster
To create a Rancher Cluster please follow the [documentation](https://github.com/mosip/k8s-infra/blob/main/rancher/on-prem/README.md)

## Create OBSRV Cluster
To create a OBSRV Cluster please follow the [documentation]()

## global configmaps
* Update the domain name in global configmap
   * api.sandbox.xyz.net
   * api-internal.sandbox.xyz.net
   * postgres.sandbox.xyz.net
   * kafka.sandbox.xyz.net
   * druid.sandbox.xyz.net
   * secor.sandbox.xyz.net
   * superset.sandbox.xyz.net

## Istio
* If you want to go with Istio as a ingress controller, then follow the  [documentation](https://github.com/mosip/k8s-infra/blob/main/mosip/on-prem/istio/README.md)

## NGINX Reverse Proxy Setup
* If u want to go with nginx as a load balancer then follow the [documentation](https://github.com/mosip/k8s-infra/blob/develop/mosip/on-prem/nginx/README.md)

## Deployement Stpes for persistent storage and object storage (s3)
### Longhorn
* Install [Longhorn](https://github.com/mosip/k8s-infra/blob/main/longhorn/README.md) for persistent storage. OR you can Install Longhorn from RancherUI also.

### MinIO
* Install [MinIO](https://github.com/mosip/mosip-infra/tree/master/deployment/v3/external/object-store/minio) for high performance, distributed object storage system.
  - To create minio buckets from CLI, run below command
```bash
mc alias set obsrvs3 https://obsrv.mosip.net:9000/
```
```bash
mc mb obsrvs3/obsrv --region="us-east-2"
```
```bash
mc mb obsrvs3/flink-checkpoints --region="us-east-2"
```
```bash
mc mb obsrvs3/velero-backup --region="us-east-2"
```

## Install OBSRV Services using Helm
To install OBSRV services follow the [documentation](https://obsrv.sunbird.org/use/obsrv-2.0-installation-guide/obsrv-deployment-using-helm) and update accordingly
### Helm Dependencies
* Run the following helm repo add command to download the required dependencies for running Obsrv.
  - monitoring -
  - redis -
  - loki (version - 4.8.0 ) -
  - promtail (version - 6.9.3) -
  - velero (version - 3.1.6 ) -
```bash
helm repo add prometheus https://prometheus-community.github.io/helm-charts
helm repo update
```
```bash
helm repo add https://charts.bitnami.com/bitnami
helm repo update
```

```bash
helm repo add https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm repo add https://grafana.github.io/helm-charts
helm repo update
```

```bash
helm repo add https://vmware-tanzu.github.io/helm-charts
helm repo update
```
## Source Code
* Clone the [obsrv-automation](https://github.com/Sunbird-Obsrv/obsrv-automation) github repository. The required list of helm charts to deploy Obsrv will be under the terraform/modules/helm directory.
```bash
git clone https://github.com/Sunbird-Obsrv/obsrv-automation.git
cd obsrv-automation/terraform/modules/helm
```
## Deployment Instructions
Helm package manager provides an easy way to install specific components using a generic command. Configurations can be overriden by updating the values.yaml file in the respective Helm charts.
```bash
Example
helm upgrade --install --atomic <release_name> <chart_name> -n <namespace> -f <path/values.yaml> --create-namespace --debug
```
### Postgres
Postgres is a RDBMS database which is used as the metadata store
```bash
helm upgrade --install --atomic obsrv-redis redis/redis -n redis -f redis/values.yaml --create-namespace --debug
```
### Redis
Redis is an in-memory key-value store primarily used as a distributed cache
```bash
helm upgrade --install --atomic obsrv-redis redis/redis -n redis -f redis/values.yaml --create-namespace --debug
```
### Prometheus
Prometheus is a monitoring system with a dimensional data model, flexible query language, efficient time series database and modern alerting approach.
```bash
helm upgrade --install --atomic monitoring monitoring/kube-prometheus-stack -n monitoring -f monitoring/values.yaml --create-namespace --debug
```
### Kafka
Apache Kafka is a distributed event store and stream-processing platform.
```bash
helm upgrade --install --atomic kafka kafka/kafka-helm-chart -n kafka --create-namespace --debug
```
The following list of kafka topics are created by default. If you would like to add more topics to the list, you can do so by adding it to provisioning.topics configuration in the [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/kafka/kafka-helm-chart/values.yaml) file.
* dev.ingest
* masterdata.ingest

### Druid
Druid is a high performance, real-time analytics database that delivers sub-second queries on streaming and batch data at scale
#### Druid CRD
```bash
helm upgrade --install --atomic druid-operator druid_operator/druid-operator-helm-chart -n druid-raw --create-namespace --debug
```

#### Druid Cluster
* Druid requires the following set of configurations to be provided for specific storage systems such as AWS S3, MinIO

  - Create `Accress Key` and `Secret Key` from the MinIO
  - Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/druid_raw_cluster/druid-raw-cluster-helm-chart/values.yaml) file and update as per your persistence storage, MinIO access and secret keys etc.
  - replace following placeholders:
```bash
     storageClass: "gp2" as storageClass: "longhorn"`
     druid_metadata_storage_connector_password: "druidraw123"` as `druid_metadata_storage_connector_password: "druid_raw"`
   ```

```bash
# AWS S3 Details
# Use the ClusterIP of the MinIO service instead of the Kubernetes service name in the endpoint_url
s3_access_key: "KP335MO3XT1A1Z7XPSWI"
s3_secret_key: "r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
s3_bucket: "obsrv"
druid_s3_endpoint_url: "http://10.43.209.124:9000/" 
druid_s3_endpoint_signingRegion: "us-east-2"
```
* Add `druid_indexer_logs_s3Bucket:"obsrv"
  `, `druid_indexer_logs_directory:"backups/druid/druid-task-logs"`,  lines as given below in values.yaml file.

* Replace `druid_broker_max_heap_size: 512M` as `druid_broker_max_heap_size: 1024M
  ` and `druid_broker_pod_memory_limit: 750Mi` as `druid_broker_pod_memory_limit: 1300Mi` in values.yaml file.
```bash
# Indexing service logs
# For local disk (only viable in a cluster_type if this is a network mount):
druid_indexer_logs_type: "s3"
druid_indexer_logs_s3Bucket: "obsrv"
druid_indexer_logs_container: ""
druid_indexer_logs_prefix: "backups/druid/druid-task-logs"
druid_indexer_logs_directory: "backups/druid/druid-task-logs"
```

* Edit [druid_statefulset.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/druid_raw_cluster/druid-raw-cluster-helm-chart/templates/druid_statefulset.yaml) file as well.
  - Add `druid.indexer.logs.s3Bucket={{ .Values.druid_indexer_logs_s3Bucket }}`, `    druid.indexer.logs.s3Prefix={{ .Values.druid_indexer_logs_prefix }}` and `	    druid.indexer.logs.directory={{ .Values.druid_indexer_logs_directory }}` in druid_statefulset.yaml file
```bash
    # # Indexing service logs
    # # For local disk (only viable in a cluster if this is a network mount):
    druid.indexer.logs.type={{ .Values.druid_indexer_logs_type }}
    druid.indexer.logs.container={{ .Values.druid_indexer_logs_container }}
    druid.indexer.logs.prefix={{ .Values.druid_indexer_logs_prefix }}
    druid.indexer.logs.s3Bucket={{ .Values.druid_indexer_logs_s3Bucket }}
    druid.indexer.logs.s3Prefix={{ .Values.druid_indexer_logs_prefix }}
    druid.indexer.logs.directory={{ .Values.druid_indexer_logs_directory }}
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic druid-raw druid_raw_cluster/druid-raw-cluster-helm-chart -n druid-raw --create-namespace --debug
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: druid
  namespace: druid-raw
spec:
  selector:
    istio: ingressgateway-internal
  servers:
  - hosts:
    - druid.obsrv.mosip.net
    port:
      name: http
      number: 80
      protocol: HTTP
 ```
- VirtualServices

```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  labels:
    app.kubernetes.io/component: mosip
  name: druid
  namespace: druid-raw
spec:
  gateways:
  - druid
  hosts:
  - '*'
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: druid-raw-routers
        port:
          number: 8888
```
## Dataset-API
* This service provides metadata APIs related to various resources such as datasets/datasources in Obsrv. The following configurations need to be specified in the [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/dataset_api/dataset-api-helm-chart/values.yaml) file.
  - Edit values.yaml file and update as per requirement
  - replace following placeholders:
```bash
  CONTAINER: "obsrv-sb-dev-165052186109" --> CONTAINER: "obsrv"
  ```
* Edit [deployment.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/dataset_api/dataset-api-helm-chart/templates/deployment.yaml) file as well.
  - Replace `type: "LoadBalancer"` as `type: ClusterIP`
  - Add `port` and `targetPort` as given below.
```bash
	spec:
  type: ClusterIP
  ports:
    - name: http-{{ .Chart.Name }}
      protocol: TCP
      port: {{ .Values.network.port }}
      targetPort: {{ .Values.network.targetport }}
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic dataset-api dataset_api/dataset-api-helm-chart -n dataset-api --create-namespace --debug 
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  annotations:
    meta.helm.sh/release-namespace: dataset-api
  name: dataset-api
  namespace: dataset-api
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - dataset-api.obsrv.mosip.net
    port:
      name: dataset-api
      number: 80
      protocol: HTTP
```
* VirtualServices
```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: dataset-api
  namespace: dataset-api
spec:
  gateways:
  - dataset-api
  hosts:
  - '*'
  http:
  - corsPolicy:
      allowCredentials: true
      allowHeaders:
      - Accept
      - Accept-Encoding
      - Accept-Language
      - Connection
      - Content-Type
      - Cookie
      - Host
      - Referer
      - Sec-Fetch-Dest
      - Sec-Fetch-Mode
      - Sec-Fetch-Site
      - Sec-Fetch-User
      - Origin
      - Upgrade-Insecure-Requests
      - User-Agent
      - sec-ch-ua
      - sec-ch-ua-mobile
      - sec-ch-ua-platform
      - x-xsrf-token
      - xsrf-token
      allowMethods:
      - GET
      - POST
      - PATCH
      - PUT
      - DELETE
      allowOrigins:
      - prefix: https://dataset-api.obsrv.mosip.net
    headers:
      request:
        set:
          x-forwarded-proto: https
    match:
    - uri:
        prefix: /
    route:
    - destination:
        host: dataset-api-service
        port:
          number: 3000
```
## Flink Streaming Jobs
Flink jobs are used to process and enrich the data ingested into Obsrv in near-realtime.
### Configuration Overrides

- Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/flink/flink-helm-chart/values.yaml) as per requirement
```bash
# AWS S3 Details
s3_access_key: "admin"
s3_secret_key: "sDABZCoVOK"
s3_endpoint: "http://10.43.209.124:9000"
s3_path_style_access: ""

# Under base_config in the values.yaml
base.url: s3://flink-checkpoints

# Under redis and redis-meta in the values.yaml
host = redis-master.redis.svc.cluster.local
```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic merged-pipeline flink/flink-helm-chart -n flink --set image.registry=sunbird --set image.repository=sb-obsrv-merged-pipeline --create-namespace --debug
```
## Secor
* Secor backups are performed from various kafka topics which are part of the data processing pipeline. The following list of backup names need to be replaced in the below mentioned command.
  - Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/secor/secor-helm-chart/values.yaml) file as per requirement
  - Add below mention preoperties in values.yaml file
```bash
aws_access_key: "KP335MO3XT1A1Z7XPSWI"
aws_secret_key: "r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
aws_region: "us-east-2"
aws_endpoint: "http://10.43.209.124:9000/"
secor_s3_bucket: "secor"
```
- Replace `cloud_storage_bucket: "obsrv-sb-dev-165052186109"` as `cloud_storage_bucket: "secor"` and `storageClass: "gp2"` as `storageClass: "longhorn"`
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```

###### NOTE: Deploy secor with above bucket name, add bucket name in the below command
```bash
helm upgrade --install --atomic <backup_name> secor/secor-helm-chart -n secor --create-namespace
```
## Velero

* Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/velero/values.yaml) file as given below

```bash
	  secretContents:
    cloud: |
      [default]
      aws_access_key_id="KP335MO3XT1A1Z7XPSWI"
      aws_secret_access_key="r793O1yhMmNCIhZovjw2W858ZNic0KUmQ0bMYeLp"
configuration:
  provider: "aws"
  backupStorageLocation:
    bucket: "velero"
    config:
      region: ""
  volumeSnapshotLocation:
    name: default
    config:
      region: ""
 ```
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
```
```bash
helm upgrade --install --atomic velero velero/velero -n velero -f velero/values.yaml --create-namespace --debug --version 3.1.6
```

## Monitoring Services
#### Monitoring Dashboards
```bash
helm upgrade --install --atomic grafana-configs grafana_configs/grafana-configs-helm-chart -n monitoring --create-namespace --debug
```
#### Monitoring Alert Rules
```bash
helm upgrade --install --atomic alertrules alert_rules/alert-rules-helm-chart -n monitoring --create-namespace --debug
```
#### Druid Exporter
```bash
helm upgrade --install --atomic druid-exporter druid_exporter/druid-exporter-helm-chart -n druid-raw --create-namespace --debug
```
#### Kafka Exporter
```bash
helm upgrade --install --atomic kafka-exporter kafka_exporter/kafka-exporter-helm-chart -n kafka --create-namespace --debug
```

#### Postgres Exporter
* Replace Service type `type: NodePort` as `type: ClusterIP`
* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
````
```bash
helm upgrade --install --atomic postgresql-exporter postgresql_exporter/postgresql-exporter-helm-chart -n postgresql --create-namespace --debug
```
#### Loki
```bash
helm upgrade --install --atomic loki loki/loki -n loki -f loki/values.yaml --create-namespace --debug --version 4.8.0
```
### Ingestion
* This helm chart is used to submit the default ingestion tasks required for the system statistics events
```bash
helm upgrade --install --atomic submit-ingestion submit_ingestion/submit-ingestion-helm-chart -n submit-ingestion --create-namespace --debug 
```
## Visualization
#### Superset
* Edit [values.yaml](https://github.com/Sunbird-Obsrv/obsrv-automation/blob/main/terraform/modules/helm/superset/superset-helm-chart/values.yaml) file as per requirement
* Replace redis-host `	redis_host: obsrv-redis-master.redis.svc.cluster.local` as `	redis_host: redis-master.redis.svc.cluster.local` and service `
  type: LoadBalancer` as `
  type: ClusterIP`

* After saving the changes run below commands.
```bash
helm dependency build
helm repo update
````
```bash
helm upgrade --install --atomic superset superset/superset-helm-chart -n superset --create-namespace --debug
```
* Go to the rancher and Import `Gateways` and `VirtualServices`Yaml files in istio, which given below
* Gateways
```bash
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: superset
  namespace: superset
spec:
  selector:
    istio: ingressgateway-internal
  servers:
  - hosts:
    - superset.obsrv.mosip.net
    port:
      name: http
      number: 80
      protocol: HTTP
```
* VirtualServices
```bash
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  labels:
    app.kubernetes.io/component: mosip
  name: superset
  namespace: superset
spec:
  gateways:
  - superset
  hosts:
  - '*'
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: superset
        port:
          number: 8088 
```
