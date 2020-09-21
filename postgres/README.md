# PostGres deployment on Platform9 Managed Kubernetes/Platform9 Managed Kubernetes Free Tier

This post will demonstrate how to deploy Postgres database on a Kubernetes cluster. It has been
verified on a PMK/PMKFT cluster but should be valid for any Kubernetes cluster. For the sake of simplicity,
we will be using a hostPath volume but you can choose any StorageClass you have deployed in the cluster.

## Prerequisites
1.  Working Kubernetes cluster 1.16+  with access to kubeconfig.
2.  Access to one of the nodes so we can access the postgres service using a NodePort type.
3. Helm 2.12+ or Helm 3.0-beta3+ (for HA deployment only)

## Deployment Steps


### Deploying for testing/Stage/Dev Environment

**NOTE**: This section is for those who want to try PostGres for testing/Stage/Dev environment and deploys PostGres as a deployment object and not as a Statefulset. The storageClass assumed in this section is hostPath which stores the data on the node's local disk. If the node crashes, the postgres data on the node will not be recoverable.

1. Clone the KoolKubernetes db repository.
```bash
$   git clone https://github.com/KoolKubernetes/db.git
```

2. Create a generic secret so we can specify postgres database Name, postgres username and password. These parameters will be used by postgres deployment for deploying PostgresDB.  

```bash
$ kubectl create secret generic postgres-secret  --from-literal=POSTGRES_DB='postgresdb' \
--from-literal=POSTGRES_USER='<username>'  --from-literal=POSTGRES_PASSWORD='<password>'
```
For eg.

```bash
$ kubectl create secret generic postgres-secret  --from-literal=POSTGRES_DB='postgresdb' \
--from-literal=POSTGRES_USER='postgresadmin'  --from-literal=POSTGRES_PASSWORD='admin123'
```

3. Before applying the yaml file that creates Postgres deployment, persistent volume (PV) and its associated persistent volume claim (PVC) along with the nodePort service, here are some of the parameters//fields that you can tweak
to choose postgresDB version etc.

  i. This walk-through uses the latest 12.3 postgresDB image but you can choose to select a particular version of PostGres from the available Docker [images](https://hub.docker.com/_/postgres?tab=tags). You can edit the image by editing the **image** field on line 52 under the Deployment object ( .spec.template.spec.containers.image field)

  ii. You  can change the persistent volume type and select any existing StorageClass that you have deployed in your cluster. It can be changed in PersistentVolumeClaim and PersistentVolume objects with the field name **storageClassName** on lines 10 and 25 (.spec.storageClassName)

  iii. You can change the size of the PersistentVolume by editing the field **storage** under PersistentVolumeClaim and PersistentVolume objects on lines 12 and line 30.

  4. Apply the deploy.yaml on the cluster by browsing to the directory where KoolKubernetes repo was  cloned and browsing to the subdirectory - <Location of KoolKubernetes_repo>/db/postgres/yaml

  ```bash
$ kubectl apply -f deploy.yaml
```

```bash
persistentvolume/postgres-pv-volume created
persistentvolumeclaim/postgres-pv-claim created
deployment.apps/postgres created
service/postgres configured
```
5. Run the following command to ensure that postgres pod transitions into a 'Running' state

```bash
$ kubectl get pods -l app=postgres
```

6. Once the pod is transitioned into a 'running' state,  you can access this pod by logging onto a worker node and accessing the postgresDB pod by running the [psql client](https://blog.timescale.com/tutorials/how-to-install-psql-on-mac-ubuntu-debian-windows/) and hitting the port 31070 where NodePort service is listening.

(**NOTE**: You can change the serviceType to loadbalancer or any other type as per your application needs. NodePort type has been choosen here only for simplicity sake.)

```bash
$ sudo psql -h localhost -U <username_specified_in_step2> --password -p 31070 <DBname_specified_in_step2>
```   

If you are following the example mentioned above,
```bash
$ sudo psql -h localhost -U postgresadmin --password -p 31070 postgresdb
```
You will be asked for a password, enter the DBpassword that was specified during Secret creation in Step 2.

7. Once the authentication is complete, you'll get a postgresDB prompt where you can Create databases,tables etc. as per your application needs.

```bash
Password for user postgresadmin:
psql (9.5.21, server 12.3 (Debian 12.3-1.pgdg100+1))
WARNING: psql major version 9.5, server major version 12.
         Some psql features might not work.
Type "help" for help.

postgresdb=#
```


### Cleanup dev/test/staging postgres setup

Run the following command to cleanup the created Kubernetes objects -

```bash
$ kubectl delete -f deploy.yaml
```

### Deploying PostGres in HA setup
**NOTE**: This section is for those who want to deploy Postgres in an HA setup where loss of one PostGres pod doesn't result in a  data loss. Please note that you need a StorageClass that provides Shared Storage, Replication and redundancy. This guide   is using [Rook](https://github.com/KoolKubernetes/csi/tree/master/rook/internal-ceph) and we'll be using StatefulSet instead of a deployment object in the earlier section.


1.  This tutorial assumes you have Helm [installed](https://helm.sh/docs/intro/install/) from where you can access the Kubernetes cluster.

2. Ensure that the default storageClass is set to SharedStorage,Replicated and Redundant or you can specify it while deploying Helm chart.
You can do it by passing the parameter --set global.storageClass=<storageClassName>

3. Deploy the Helm chart by running the command -
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql-ha bitnami/postgresql-ha
```

(NOTE: you can pass additional values  referred [here](https://github.com/bitnami/charts/tree/master/bitnami/postgresql-ha/#installing-the-chart) if needed)

4. You should be able to observe 2 replicas by default one is the master and the second is standby.

```bash
kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
my-release-postgresql-ha-pgpool-768898c659-xlrqn   1/1     Running   0          156m
my-release-postgresql-ha-postgresql-0              1/1     Running   1          147m
my-release-postgresql-ha-postgresql-1              1/1     Running   0          155m
```

5. Checking the logs on the master pod,here's what you can observe -

```bash
[2020-08-20 08:16:02] [NOTICE] starting monitoring of node "my-release-postgresql-ha-postgresql-2" (ID: 1001)
```
On the standby pod, you can observe the logs as mentioned below

```bash
[2020-08-20 08:14:46] [NOTICE] monitoring cluster primary "my-release-postgresql-ha-postgresql-1" (ID: 1000)
[2020-08-20 08:14:52] [NOTICE] new standby "my-release-postgresql-ha-postgresql-2" (ID: 1001) has connected
```

### Cleanup the HA Postgres setup

```bash
helm delete  postgresql-ha
```
