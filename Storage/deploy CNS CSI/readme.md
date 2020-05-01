# deploy CNS CSI

## Introduction
This page contains information to deploy CNS CSI on a K8s cluster deployed by TKGI.

Very useful link to understand CSI more in details:
<https://tanzu.vmware.com/content/blog/supercharging-kubernetes-storage-with-csi>


## Pre-requisites

- TKGI version 1.7.0
- vSphere 6.7U3 (or later)
- `Allowed Provileged` enabled on the TKGI plan used to deploy the K8s cluster

### Network Requirement

- All TKGI k8s worker node VMs will need access to vCenter. 

In case of NSX-T integration, what is means is the Floating IP allocated to the SNAT of this namespace in the T0 (or T1 if shared T1 model is used) MUST be able to reach vCenter.

## Manifest Files

- vsphere-csi-controller-rbac.yaml
- vsphere-csi-controller-ss-data-1.yaml
- vsphere-csi-node-ds-data.yaml

## Step1: Deploy a K8s Cluster

```
$ pks create-cluster tkgi-cluster-1 --external-hostname tkgi-cluster-1 --plan large
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
$ kubectl create secret generic vsphere-config-secret --from-file=csi-vsphere.conf --namespace=kube-system
secret/vsphere-config-secret created
```

Check:
```
$ kubectl get secret/vsphere-config-secret -n kube-system
NAME                    TYPE     DATA   AGE
vsphere-config-secret   Opaque   1      37s
```

## Step3: Create Roles, ServiceAccount and ClusterRoleBinding

```
$ kubectl apply -f vsphere-csi-controller-rbac.yaml
serviceaccount/vsphere-csi-controller created
clusterrole.rbac.authorization.k8s.io/vsphere-csi-controller-role created
clusterrolebinding.rbac.authorization.k8s.io/vsphere-csi-controller-binding created
```

Check:

list the service account:
```
$ kubectl get sa -n kube-system | grep csi
vsphere-csi-controller               1         50s
```

list the cluster roles:
```
$ kubectl get clusterrole -n kube-system | grep csi
system:csi-external-attacher                                           23h   
system:csi-external-provisioner                                        23h 
vsphere-csi-controller-role                                            29s
```

list the cluster role binding:
```
$ kubectl get clusterrolebinding -n kube-system | grep csi
vsphere-csi-controller-binding                         3m39s
```

## Step4: Install the vSphere CSI Controller and CSI driver

```
$ kubectl apply -f vsphere-csi-controller-ss-data-1.yaml
statefulset.apps/vsphere-csi-controller created
csidriver.storage.k8s.io/csi.vsphere.vmware.com created
```

Check:

list the pod:
```
$ kubectl get pod -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
<SNIP>
vsphere-csi-controller-0          5/5     Running   0          2m53s
```

list the stateful set:
```
$ kubectl get statefulset -n kube-system
NAME                     READY   AGE
vsphere-csi-controller   1/1     24s
```

list the CSI driver:
```
$ kubectl get csidriver
NAME                     CREATED AT
csi.vsphere.vmware.com   2020-04-01T17:56:59Z
```

## Step5: Install the vSphere CSI Node

```
$ kubectl apply -f vsphere-csi-node-ds-data.yaml
daemonset.apps/vsphere-csi-node created
```

Check:

list the pods:
```
$ kubectl get pod -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
<SNIP>
vsphere-csi-controller-0          5/5     Running   0          6m37s
vsphere-csi-node-22wbz            3/3     Running   0          35s
vsphere-csi-node-4cs9k            3/3     Running   0          35s
vsphere-csi-node-xgl9q            3/3     Running   0          35s
```

list the daemonset:
```
$ kubectl get daemonset -n kube-system
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
vsphere-csi-node   3         3         3       3            3           <none>          86s
```

list the statefulset:
```
$ kubectl get statefulset --namespace=kube-system
NAME                     READY   AGE
vsphere-csi-controller   1/1     13m
```

list CSI nodes:
```
$ kubectl get CSINode
NAME                                   CREATED AT
095cbdfb-674c-47dd-884d-ed7232debe6d   2020-04-01T18:03:08Z
a4061fe4-7560-419a-91cc-2f92c4db5593   2020-04-01T18:03:13Z
bf5fcafa-1fcd-4676-a8c8-917c84223c89   2020-04-01T18:03:13Z
```

list CSI driver:
```
$ kubectl get csidrivers
NAME                     CREATED AT
csi.vsphere.vmware.com   2020-04-01T17:56:59Z
```

verify ProviderID has been added the nodes:

```
$ kubectl describe nodes | grep "ProviderID"
ProviderID:                  vsphere://4210371f-bf87-3892-6e18-f8d739b9f099
ProviderID:                  vsphere://4210fbb5-e5a5-3435-85ce-8d1c23c8dbbf
ProviderID:                  vsphere://4210cdb6-e4d3-336c-4f70-9037356be36d
```