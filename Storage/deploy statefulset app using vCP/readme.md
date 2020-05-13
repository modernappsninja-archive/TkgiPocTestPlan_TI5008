# Deploy StatefulSet App Using vCP

## Introduction

This page contains information to deploy a statefulset application using vCP storage plugin.
vCP is the in-tree storage plugin available natively with K8s bits and works out of the box in a vSphere environment.

In this lab, we are going to deploy a Cassandra Statefulset app with 3 replicas and we are going to populate the distributed database with some data in order to check everything is working fine.


### Pre-requisites

- CSI storage plugin deployed on the K8s cluster.

Please refer to this link for the CSI storage plugin installation:

![deploy vSphere CSI Storage Plugin on a TKGI based K8s Cluster](https://github.com/ModernAppsNinja/TkgiPocTestPlan_TI5008/tree/master/Storage/deploy%20CSI%20Storage%20Plugin)


### Manifest Files

All the following manifests files are provided inside this directory:

    cassandra-sc-csi.yaml
    cassandra-statefulset.yaml
    cassandra-headless-service.yaml


## Step1: Create Namespace to deploy the statefulset app

```
kubectl create ns cassandra-csi
namespace/cassandra-csi created
```

## Step2: Create Storage Class for CSI

Create the following file in your system (also provided in this Github directory):
