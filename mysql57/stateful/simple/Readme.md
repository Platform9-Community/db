# Simple and stateful Mysql DB deployment on Platform9 Managed Kubernetes Freedom Plan


Here we are going to deploy Mysql 5.7 database server on top of Platform9 Managed kubernetes 4.4 

## Configuration
Before deploying the yaml file label one node with 'mysql57' as key so that mysql pod gets scheduled on this node. This is to ensure the pod always gets scheduled to same node. Underlying persistent volume is also present on the same node. Both are managed through nodeAffiinity. 

Select the node with enough resources for mysql to run on. Label it in the following manner. 

```bash
$ kubectl label nodes <node-name> mysql57=allow
```
Now clone the Kool Kubernetes repository on any machine from where you can deploy json manifests to your kubernetes cluster.

```bash
$ git clone https://github.com/KoolKubernetes/db.git
```



