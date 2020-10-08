# Simple stateful Mysql DB deployment on Platform9 Managed Kubernetes Freedom Plan


Here we are going to deploy Mysql 5.7 database server on top of Platform9 Managed kubernetes 4.4. The deployment will be backed by a persistent volume of the type 'hostPath'. [Rook](https://github.com/KoolKubernetes/csi/tree/master/rook/) can also be used as persistent storage CSI backend.  Slight changes in pv and pvc definition portions of MySQL deployment manifest will be needed to use it with Rook.

## Configuration
Before deploying the yaml file label one node with 'mysql57' as key so that mysql pod gets scheduled only on this node. This is managed through nodeAffinity in PV properties.

Select the node with enough resources for mysql to run on to label it in the following manner. 

```bash
$ kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
10.128.233.18   Ready    master   7h11m   v1.17.6
10.128.233.25   Ready    master   7h11m   v1.17.6
10.128.233.26   Ready    worker   7h6m    v1.17.6
10.128.233.42   Ready    worker   7h6m    v1.17.6
10.128.233.47   Ready    master   7h11m   v1.17.6
10.128.233.71   Ready    worker   4h47m   v1.17.6
$ kubectl label nodes 10.128.233.42 mysql57=allow
```
Clone the KoolKubernetes db repository.

```bash
$ git clone https://github.com/KoolKubernetes/db.git
```
Apply deploy.yaml on your cluster

```bash
cd db/mysql57/stateful/simple ; kubectl apply -f deploy.yaml
```

Track deployment status
```bash
kubectl get pods -n mysql57 -w
```

Deployment is successful as soon as mysql57 pod progresses into 'Running' state. Deployment also creates a headless service to provide access to the db.
```bash
$ kubectl get all,svc,pvc -n mysql57
NAME                           READY   STATUS    RESTARTS   AGE
pod/mysql-client               1/1     Running   0          20m
pod/mysql57-7cdfc9f78f-xvvx7   1/1     Running   1          52m

NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/mysql57   ClusterIP   None         <none>        3306/TCP   45m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mysql57   1/1     1            1           53m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/mysql57-7cdfc9f78f   1         1         1       53m

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql57-pvc   Bound    mysql57-pv   10Gi       RWO            mysql57        53m
```

Allow access to the DB from external IP addresses. First login to the pod. Run mysql after landing into the pod.
```bash
$ kubectl get pods --field-selector=status.phase=Running,spec.restartPolicy=Always -n mysql57
NAME                       READY   STATUS    RESTARTS   AGE
mysql57-7cdfc9f78f-xvvx7   1/1     Running   1          62m

$ kubectl exec -it mysql57-7cdfc9f78f-xvvx7 -n mysql57 -- /bin/bash

root@mysql57-7cdfc9f78f-xvvx7:/# mysql
```

Run following commands on mysql prompt.

```bash
mysql> GRANT ALL ON *.* to root@'%' IDENTIFIED BY 'secretformysql';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)

mysql>  SELECT host FROM mysql.user WHERE User = 'root';
+-----------+
| host      |
+-----------+
| %         |
| localhost |
+-----------+
2 rows in set (0.00 sec)

mysql> exit;
Bye
```
Exit from the pod
```bash
root@mysql57-7cdfc9f78f-xvvx7:/# exit
```

Test connection through another pod created from same mysql image. Press enter if you do not get mysql prompt after running the command.

```bash
$ kubectl run -it --rm --image=mysql:5.7 --restart=Never mysql-client -n mysql57 -- mysql -h mysql57 -psecretformysql
mysql> use mysql
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              |
| db                        |
| engine_cost               |
| event                     |
| func                      |
| general_log               |
| gtid_executed             |
| help_category             |
| help_keyword              |
| help_relation             |
| help_topic                |
| innodb_index_stats        |
| innodb_table_stats        |
| ndb_binlog_index          |
| plugin                    |
| proc                      |
| procs_priv                |
| proxies_priv              |
| server_cost               |
| servers                   |
| slave_master_info         |
| slave_relay_log_info      |
| slave_worker_info         |
| slow_log                  |
| tables_priv               |
| time_zone                 |
| time_zone_leap_second     |
| time_zone_name            |
| time_zone_transition      |
| time_zone_transition_type |
| user                      |
+---------------------------+
31 rows in set (0.00 sec)

mysql> quit
Bye
pod "mysql-client" deleted
```

The client pod will get deleted as soon the user quits the mysql client.

## Cleanup
Simply run following command to delete everything.
```bash
$ kubectl delete -f deploy.yaml
```
## Refrences
https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/
https://github.com/KoolKubernetes/csi/tree/master/rook



