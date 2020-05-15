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

    cassandra-sc-vcp.yaml
    cassandra-statefulset.yaml
    cassandra-headless-service.yaml


## Step1: Create Namespace to deploy the statefulset app

```
kubectl create ns cassandra-vcp
namespace/cassandra-vcp created
```

## Step2: Create Storage Class for vCP

Create the following file in your system (also provided in this Github directory):

cassandra-sc-vcp.yaml
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cassandra-sc-vcp
provisioner: kubernetes.io/vsphere-volume
parameters:
  diskformat: thin
```

In the above exemple:

- **`cassandra-sc-vcp`**: unique name for the Storage Class
- **`thin`**: thin provisioning mode selected for the VMDK file that will be associated with the PV



Apply the manifest file
```
$ kubectl apply -f cassandra-sc-vcp.yaml -n cassandra-vcp
storageclass.storage.k8s.io/cassandra-sc-vcp created
```

Check:
```
$ kubectl get sc -n cassandra-vcp
NAME                         PROVISIONER              AGE
cassandra-sc-vcp   kubernetes.io/vsphere-volume   38s
```



## Step3: Create the headless service

```
$ kubectl apply -f cassandra-headless-service.yaml -n cassandra-vcp
service/cassandra created
```

Check:
```
$ kubectl get svc -n cassandra-vcp
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
cassandra   ClusterIP   None         <none>        <none>    50s
```

## Step4: Deploy the StatefulSet Cassandra DB with 3 replicas

```
$ kubectl apply -f cassandra-statefulset.yaml -n cassandra-vcp
statefulset.apps/cassandra created
```

Check:
```
$ kubectl get statefulset  -n cassandra-vcp
NAME        READY   AGE
cassandra   3/3     55s
```


## Overall Check

Check pods:

```
$ kubectl get pod -n cassandra-vcp
NAME          READY   STATUS    RESTARTS   AGE
cassandra-0   1/1     Running   0          3m13s
cassandra-1   1/1     Running   0          2m7s
cassandra-2   0/1     Running   0          45s
```

Check PVC:

```
$ kubectl get pvc -n cassandra-vcp
NAME                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
persistentvolumeclaim/cassandra-data-cassandra-0   Bound    pvc-625fa575-f7ed-4b00-b8a5-49bc2c22c1de   100Gi      RWO            cassandra-sc-vcp   3m30s
persistentvolumeclaim/cassandra-data-cassandra-1   Bound    pvc-372b7cbb-f600-4a5b-96f2-fb821eafdd0f   100Gi      RWO            cassandra-sc-vcp   2m24s
persistentvolumeclaim/cassandra-data-cassandra-2   Bound    pvc-7e5feaff-86ef-4891-9747-bcee1c973ba2   100Gi      RWO            cassandra-sc-vcp   62s
```

Check PV:

```
$ kubectl get pv -n cassandra-vcp
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                      STORAGECLASS       REASON   AGE
persistentvolume/pvc-372b7cbb-f600-4a5b-96f2-fb821eafdd0f   100Gi      RWO            Delete           Bound    cassandra-vcp/cassandra-data-cassandra-1   cassandra-sc-vcp            2m22s
persistentvolume/pvc-625fa575-f7ed-4b00-b8a5-49bc2c22c1de   100Gi      RWO            Delete           Bound    cassandra-vcp/cassandra-data-cassandra-0   cassandra-sc-vcp            3m28s
persistentvolume/pvc-7e5feaff-86ef-4891-9747-bcee1c973ba2   100Gi      RWO            Delete           Bound    cassandra-vcp/cassandra-data-cassandra-2   cassandra-sc-vcp            60s
```


## Test: Populate Cassandra DB with new data

Create the following file:

cassandra-data.txt
```
CREATE KEYSPACE demodb WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 2 };

use demodb;

CREATE TABLE emp(emp_id int PRIMARY KEY, emp_name text, emp_city text, emp_sal varint,emp_phone varint);


INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500000 , 'emp1', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500001 , 'emp2', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500002 , 'emp3', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500003 , 'emp4', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500004 , 'emp5', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500005 , 'emp6', 'SFO', 999, 1000000);
INSERT INTO emp (emp_id, emp_name, emp_city, emp_phone, emp_sal) VALUES ( 4500006 , 'emp7', 'SFO', 999, 1000000);

exit
```

Populate the database with the above data:
```
cat cassandra-data.txt | kubectl exec -it cassandra-0 -n cassandra-vcp -- cqlsh
```

## Check: Verify Cassandra DB contains the new data

Create the following file:

cassandra-check-data.txt
```
use demodb;
select * from emp;
exit
```

Check the data is correctly stored on all 3 replicas of the Cassandra DB:

```
cat cassandra-check-data.txt | kubectl exec -it cassandra-0 -n cassandra-vcp -- cqlsh
cat cassandra-check-data.txt | kubectl exec -it cassandra-1 -n cassandra-vcp -- cqlsh
cat cassandra-check-data.txt | kubectl exec -it cassandra-2 -n cassandra-vcp -- cqlsh
```