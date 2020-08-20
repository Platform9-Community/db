# PostGres deployment on Platform9 Managed Kubernetes/Platform9 Managed Kubernetes Free Tier

This post will demonstrate how to deploy Postgres database on a Kubernetes cluster. It has bee
verified on a PMK/PMKFT cluster but should be valid for any Kubernetes cluster. For the sake of simplicity,
we will be using a hostPath volume but you can choose any storageClass you have deployed in the cluster.

## Prerequisites
1.  Working Kubernetes cluster with access to kubeconfig.
2.  Access to one of the nodes so we can access the postgres service using a NodePort type.

## Deployment Steps

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


## Cleanup

Run the following command to cleanup the created Kubernetes objects -

```bash
$ kubectl delete -f deploy.yaml
```
