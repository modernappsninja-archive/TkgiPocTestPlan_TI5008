# deploy vSphere CSI Storage Plugin on a TKGI based K8s Cluster

## Introduction

This page contains information to deploy vSphere CSI storage plugin on a K8s cluster created by TKGI.

Version of vSphere CSI Driver Image is 1.0.2.


### CSI Architecture

Very useful link to understand CSI more in details:

<https://tanzu.vmware.com/content/blog/supercharging-kubernetes-storage-with-csi>

![CSI architecture](https://github.com/ModernAppsNinja/TkgiPocTestPlan_TI5008/blob/master/Storage/deploy%20CSI%20Storage%20Plugin/CSI.png)

## Pre-requisites

- TKGI version 1.7.0 and above
- vSphere 6.7U3 and above
- `Allowed Provileged` enabled on the TKGI plan used to deploy the K8s cluster

### Network Requirement

- TKGI with Flannel: TKGI k8s worker nodes will need access to vCenter. In fact, only pod vsphere-csi-controller-0 needs access to vCenter but because it can be scheduled on any worker node, then the requirement for all worker nodes applies.

- TKGI with NSX-T integration: the Floating IP allocated to the SNAT for the namespace kube-system in  T0 (or T1 if shared T1 model is used) MUST be able to reach vCenter.

## Manifest Files

All the following manifests files are provided inside this directory:

- vsphere-csi-controller-rbac.yaml
- vsphere-csi-controller-ss-data-1.yaml
- vsphere-csi-node-ds-data.yaml

## Step1: Deploy a K8s Cluster

```
$ pks create-cluster tkgi-cluster-1 --external-hostname tkgi-cluster-1 --plan large
```

Reminder:
TKGI plan named `large` has the option `Allowed Provileged` enabled.

## Step2: Create a CSI Secret

Create the following file anywhere in your system:

csi-vsphere.conf

```
[Global]
cluster-id = "pks-cluster-1-shared-t1"

[VirtualCenter "10.1.1.1"]
insecure-flag = "true"
user = "administrator@vsphere.local"
password = "password"
port = "443"
datacenters = "vSAN_Datacenter"
```

In the above exemple:

- **`pks-cluster-1-shared-t1`**: unique ID for the K8s cluster (must be unique per K8s cluster)
- **`10.1.1.1`**: IP of vCenter
- **`administrator@vsphere.local`**: vCenter username
- **`password`**: vCenter username password
- **`vSAN_Datacenter`**: name of the vCenter datacenter


Note: in case a specific vCenter user with less privileges than the administrator must be used, creater a user and then assign one of the following role as listed here:
[Cloud Native Storage Roles and Privileges](https://docs.vmware.com/en/VMware-vSphere/6.7/Cloud-Native-Storage/GUID-AEB07597-F303-4FDD-87D9-0FDA4836E5BB.html)




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

list the ServiceAccount:
```
$ kubectl get sa -n kube-system | grep csi
vsphere-csi-controller               1         50s
```

list the ClusterRole:
```
$ kubectl get clusterrole -n kube-system | grep csi
system:csi-external-attacher                                           23h   
system:csi-external-provisioner                                        23h 
vsphere-csi-controller-role                                            29s
```

list the ClusterRoleBinding:
```
$ kubectl get clusterrolebinding -n kube-system | grep csi
vsphere-csi-controller-binding                         3m39s
```

## Step4: Install the vSphere CSI Controller StatefulSet and CSI driver

```
$ kubectl apply -f vsphere-csi-controller-ss-data-1.yaml
statefulset.apps/vsphere-csi-controller created
csidriver.storage.k8s.io/csi.vsphere.vmware.com created
```

Check:

list the CSI Controller StatefulSet:
```
$ kubectl get statefulset -n kube-system
NAME                     READY   AGE
vsphere-csi-controller   1/1     24s
```

list the CSI Controller pod:
```
$ kubectl get pod -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
<SNIP>
vsphere-csi-controller-0          5/5     Running   0          2m53s
```


list the CSI driver:
```
$ kubectl get csidriver
NAME                     CREATED AT
csi.vsphere.vmware.com   2020-04-01T17:56:59Z
```

## Step5: Install the vSphere CSI Node Daemonset

```
$ kubectl apply -f vsphere-csi-node-ds-data.yaml
daemonset.apps/vsphere-csi-node created
```

Check:

list the CSI Node Daemonset:
```
$ kubectl get daemonset -n kube-system
NAME               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
vsphere-csi-node   3         3         3       3            3           <none>          86s
```

list the CSI Node pods:
```
$ kubectl get pod -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
<SNIP>
vsphere-csi-controller-0          5/5     Running   0          6m37s
vsphere-csi-node-22wbz            3/3     Running   0          35s
vsphere-csi-node-4cs9k            3/3     Running   0          35s
vsphere-csi-node-xgl9q            3/3     Running   0          35s
```


## Overall Check

list CSI nodes:
```
$ kubectl get CSINode
NAME                                   CREATED AT
095cbdfb-674c-47dd-884d-ed7232debe6d   2020-04-01T18:03:08Z
a4061fe4-7560-419a-91cc-2f92c4db5593   2020-04-01T18:03:13Z
bf5fcafa-1fcd-4676-a8c8-917c84223c89   2020-04-01T18:03:13Z
```


verify ProviderID has been added the nodes:
```
$ kubectl describe nodes | grep "ProviderID"
ProviderID:                  vsphere://4210371f-bf87-3892-6e18-f8d739b9f099
ProviderID:                  vsphere://4210fbb5-e5a5-3435-85ce-8d1c23c8dbbf
ProviderID:                  vsphere://4210cdb6-e4d3-336c-4f70-9037356be36d
```



## Annex: Containers inside CSI Controller pod and CSI Node pod


CSI Controller Pod:
```
$ kubectl describe pod vsphere-csi-controller-0 -n kube-system | grep Image:
Containers:
  csi-attacher:
    Container ID:  docker://ada8e66ef776adc8884e3a26af3381668766ce31a3e320305643445525019e04
    Image:         quay.io/k8scsi/csi-attacher:v1.1.1
  vsphere-csi-controller:
    Container ID:  docker://15a16675a58f6ee6beabdc5324851cdf059820c554bf98e5b76416e0f26744d6
    Image:         gcr.io/cloud-provider-vsphere/csi/release/driver:v1.0.2
  liveness-probe:
    Container ID:  docker://57c9498613fcb30b67963797a28a125bae2bb3d21e6ff9d18b521236be54a576
    Image:         quay.io/k8scsi/livenessprobe:v1.1.0
  vsphere-syncer:
    Container ID:  docker://4196c2be0f21cc34b15677715b9edd529fd031c8fc667380847635180fbfe553
    Image:         gcr.io/cloud-provider-vsphere/csi/release/syncer:v1.0.2
  csi-provisioner:
    Container ID:  docker://9eab0125e0486500301c6c799631cd776b34ca7eb4aa2f2e54624f90bed2fd76
    Image:         quay.io/k8scsi/csi-provisioner:v1.2.2



```


CSI Node Pod:
```
$ kubectl describe pod vsphere-csi-node-4cs9k -n kube-system | grep Image:
Containers:
  node-driver-registrar:
    Container ID:  docker://8f8c7893ebf4282faf33e0eafe7580a0db7ef4ce19a722d6ea32b2f929f4a4a8
    Image:         quay.io/k8scsi/csi-node-driver-registrar:v1.1.0
  vsphere-csi-node:
    Container ID:  docker://d0e2c5bf08d507a4dd077659fcc331a4f9a09ae6cb4f7ee9622f502cefeebc3b
    Image:         gcr.io/cloud-provider-vsphere/csi/release/driver:v1.0.2
  liveness-probe:
    Container ID:  docker://09f24ce44e926b505ca3843fa42ca27a7a88299e2a1fe47c482fb28429345b4e
    Image:         quay.io/k8scsi/livenessprobe:v1.1.0
```

