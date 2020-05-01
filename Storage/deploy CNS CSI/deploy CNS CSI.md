# deploy CNS CSI

## Introduction
This page contains information to deploy CNS CSI on a K8s cluster deployed by TKGI

## Pre-requisites

- TKGI version 1.7.0
- vSphere 6.7U3 (or later)
- `Allowed Provileged` enabled on the TKGI plan used to deploy the K8s cluster

### Network Requirement

- All TKGI k8s worker node VMs will need access to vCenter. 

In case of NSX-T integration, what is means is the Floating IP allocated to the SNAT of this namespace in the T0 (or T1 if shared T1 model is used) MUST be able to reach vCenter.

## Manifest Files

- File1.yaml
- File2.yaml

## Step1: Deploy a K8s Cluster

```
#pks create-cluster tkgi-cluster-1 --external-hostname tkgi-cluster-1 --plan large
```

Reminder:
TKGI plan named `large` has the option `Allowed Provileged` enabled 

## Step2: Create a CSI Secret

Create the following file anywhere in your system:

csi-vsphere.conf
```
[Global]
cluster-id = "pks-cluster-1-shared-t1"

[VirtualCenter "10.1.1.1"]
insecure-flag = "true"
user = "administrator@vsphere.local"
password = "VMware1!"
port = "443"
datacenters = "vSAN_Datacenter"
```

Then issue the command:

```
$kubectl create secret generic vsphere-config-secret --from-file=csi-vsphere.conf --namespace=kube-system
secret/vsphere-config-secret created
```