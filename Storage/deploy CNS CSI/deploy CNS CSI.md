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

## Step1: deploy a K8s cluster

```
#pks create-cluster tkgi-cluster-1 --external-hostname tkgi-cluster-1 --plan large
```

Reminder:
TKGI plan named `large` has the option `Allowed Provileged` enabled 

## Step2: 


