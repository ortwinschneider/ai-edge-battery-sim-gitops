# GitOps repository for the transportation robot battery-simulation demo on OpenShift

## Application components

- AMQ Broker Operator: Manages the AMQ broker StatefulSet
- AMQ Broker: Configured as a MQTT broker
- Battery simulator: Quarkus component that simulates the battery of a driving transportation robot in a plant (shop floor). The BMS of the device is sending telemetry data to the device local MQTT broker.
- InfluxDB: Local time series database, configured with auto setup.
- influx-exporter: Exports time series data every 10 minutes to a S3 bucket for AI model training. Has a REST API for manual export.
- data-ingester: Quarkus component that reads realtime data from MQTT and stores it InfluxDB.
- BMS Device Dashboard: Angular component that displays the realtime battery telemetry data and provides AI inference capabilities. Can also simulate battery temperature anomalies.
- mqtt2ws: Camel Quarkus component that reads realtime data from MQTT and exposes it as Websocket for the BMS Dashboard for visualization.

## Demo Installation

The demo application requires an S3 compatible Object storage. We'll use MinIO for the demo.

### 1. Install MinIO Argo application

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minio
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: ''
    server: https://kubernetes.default.svc
  source:
    path: apps/minio
    repoURL: https://github.com/ortwinschneider/ai-edge-battery-sim-gitops
    targetRevision: HEAD
  sources: []
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

Install the MinIO argo application:

````shellscript
oc apply -f minio-app.yaml -n openshift-gitops
````

### 2. Install the transportation robot battery-simulation Argo application

Create a new Argo application that points to `apps/battery-demo/groups/dev`. It will install all the components in the `battery-demo` namespace

````yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: battery-demo
  namespace: openshift-gitops
spec:
  destination:
    name: ''
    namespace: ''
    server: https://kubernetes.default.svc
  source:
    path: apps/battery-demo/groups/dev
    repoURL: https://github.com/ortwinschneider/ai-edge-battery-sim-gitops
    targetRevision: HEAD
  sources: []
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
````

Install the transportation robot battery-simulation Argo application:

````shellscript
oc apply -f battery-demo-app.yaml -n openshift-gitops
````

## MinIO Login

You can log in to the MinIO Web UI with user `minio` and password `minio123`.

## Exploring the data in InfluxDB

Login to the InfluxDB Web UI with user `admin` and password `password`.
Navigate to `Explore` and create a query with `Script Editor`.
Use this query to find the data from the last hour:

````shellscript
from(bucket: "bms")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "battery_data")
````