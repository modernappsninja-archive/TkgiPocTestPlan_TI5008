# deploy stateful app using CSI

## Introduction

This page contains information to deploy a stateful applicatin using CSI storage plugin.


### Pre-requisites

- CSI storage plugin deployed on the K8s cluster.

Please refer to this link for the CSI storage plugin installation:

![deploy vSphere CSI Storage Plugin on a TKGI based K8s Cluster](https://github.com/ModernAppsNinja/TkgiPocTestPlan_TI5008/tree/master/Storage/deploy%20CSI%20Storage%20Plugin)


### Manifest Files

All the following manifests files are provided inside this directory:

    vsphere-csi-controller-rbac.yaml
    vsphere-csi-controller-ss-data-1.yaml
    vsphere-csi-node-ds-data.yaml

## Step1: Deploy a K8s Cluster

