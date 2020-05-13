# deploy stateful app using CSI

## Introduction

This page contains information to deploy a stateful applicatin using CSI storage plugin.

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

## Step1: Create Storage Class for CSI

Create the following file in your system:

cassandra-sc-csi.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cassandra-sc-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.vsphere.vmware.com
parameters:
  datastoreurl: "ds:///vmfs/volumes/vsan:52d8eb4842dbf493-41523be9cd4ff7b7/"
```

In the above exemple:

- **`cassandra-sc-csi`**: unique name for the Storage Class
- **`ds:///vmfs/volumes/vsan:52d8eb4842dbf493-41523be9cd4ff7b7/`**: datastore URL

Note: the annotation was defined to set the Storage Class as the default one.
This means that if a PVC is created without referencing any Storage Class, cassandra-sc-csi will then be selected as the default one.



