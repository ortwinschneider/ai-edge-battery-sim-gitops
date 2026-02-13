# GitOps repo for battery-simulation demo

## Components

- AMQ Broker Operator
- AMQ Broker: Used as MQTT broker
- Battery simulator: Quarkus component that simulates the battery of a driving Tesla S. The BMS is sending telemetry data to the MQTT broker.
- InfluxDB: Time series database, configured with auto setup
- data-ingester: Quarkus component that reads data from MQTT and stores it InfluxDB
- BMS Dashboard: Angular component that displays the battery telemetry data in realtime and serves as a frontend for a GenAI chatbot.
- mqtt2ws: Camel Quarkus component that reads data from MQTT and exposes it as Websocket for the BMS Dashboard

## Installation

### Install Minio Argo Application

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

Install the argo app:

````shellscript
oc apply -f minio-app.yaml -n openshift-gitops
````

### Install Battery Demo Application

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

Install the argo app:

````shellscript
oc apply -f battery-demo-app.yaml -n openshift-gitops
````

## Exploring the data in InfluxDB

Login to the InfluxDB Web UI with user `admin` and password `password`.
Navigate to `Explore` and create a query with `Script Editor`. 
Use this query to find the data from the last hour:

````shellscript
from(bucket: "bms")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "battery_data")
````