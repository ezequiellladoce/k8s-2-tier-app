# k8s 2 tier App with a fronted Deployment and a MySQL StatefulSet as backend

In this replo we will deploy a 2 tier app. As a frontend we will use phpmyadmin with a k8s Deployment  with 3 Repliacas , the phpmyadmin wil connect to a MySQL StatefulSet as backend with 3 replicas.

The Mysql Stateful will record data persistently in a persistent volume and can replicate data to the other pods in the same StatefullSet.

A StatefulSet uses Headless service to provide the domain control of its pods.   

Secrets are used to supply sensitive information to containers.  

## Objetives

- Deploy a simple StatefulSet MySql db wit 3 replicas as a backend. 
- Add a Headless service to provide a stable domain controller of the pods.
- Use  secrets to mount the passwords in the mysql pods.
- Add a database to Mysql throw configmaps 
- Deploy a Phpmyadmin as a front end with 3 replicas.
- Add a Load balancer to the deployment
- Connect the front end Deployment with the backend StatefulSet.  
- Test the app.

## Prerequisites 📋

- A k8s cluster v1.9 or later, for testing purposes can be minikube (https://minikube.sigs.k8s.io/docs/). You can check the k8s server with the command kubectl version
- kubectl configured to communicate with the cluster

## Starting 🚀

## Repo description

- Mysql folder
  - mysql-secret.yaml this YAML is used to mount the password as a secret
  - mysql-statefulset.yaml this YAML file describes the Stateful set and associated the pods to the persisten volume
  - mysql-service.yaml this YAML provides the a load balancer service, this could be use to connect to the Mysql StatefulSet ony for reading
  - mysql-headless-service.yaml this YAML provides a Headless service to provide domain control of the pods.
  - mysql-secret.yaml this YAML provides a pod to test the connection to the MySql StatefulSet

- PhpMyadmin folder
  - php-deplyment.yaml this YAML provides the phpmyadmin Deplyment adnd connect to the Mysql deployment throu the  envs of phpmyadmin docker image

## Clone the repository 

```
git clone https://github.com/ezequiellladoce/k8s-2-tier-app.git
```

## Deploy 📦  

Enter to the repo folder

### Create namespace

```
kubectl create namespace k8s-app

kubectl get namespace
```
### Create the secret
```
kubectl apply -f mysql-secret.yaml -n k8s-app   
secret/mysql-secret created
```
Check the secret 
```
kubectl describe secret mysql-secret -n k8s-mysql
```
You will get an output like that:
```
Name:         mysql-secret
Namespace:    k8s-mysql
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
ROOT_PASSWORD:  8 bytes
```
In this file The password were encoded in base64
```
echo -n "password" | base64
cGFzc3dvcmQ=
```
And added to data in the secret.yaml file in data.
```
apiVersion: v1
kind: Secret
metadata: 
    name: mysql-secret
type: Opaque
data:
   ROOT_PASSWORD: cGFzc3dvcmQ=

```

### Create the Headless Service

```
kubectl -f apply service.yaml -n k8s-mysql
```
Check the service
```
kubectl describe scv svc.mysql -n k8s-mysql
```
You will get an output like that:
```
Name:              svc-mysql
Namespace:         k8s-mysql
Labels:            app=k8s-mysql
Annotations:       <none>
Selector:          app=k8s-mysql
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                None
IPs:               None
Port:              tcp  3306/TCP
TargetPort:        3306/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```
A Headless service is specified with That does not allocate an IP address or forward traffic. This is specified in  the service.yaml explicitly setting ClusterIP to “None”.
```
apiVersion: v1
kind: Service
metadata:
  name: svc-mysql
  labels:
    app: k8s-mysql
spec:
  clusterIP: None
  selector:
    app: k8s-mysql
  ports:
    - name: tcp
      protocol: TCP
      port: 3306
```

### Create the StatefulSet
```
kubectl apply -f statefulset.yaml -n k8s-mysql
```
Check the StatefulSet
```
Name:               mysql
Namespace:          k8s-mysql
CreationTimestamp:  Sun, 06 Nov 2022 18:40:52 +0100
Selector:           app=k8s-mysql
Labels:             <none>
Annotations:        <none>
Replicas:           3 desired | 3 total
Update Strategy:    RollingUpdate
  Partition:        0
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=k8s-mysql
  Containers:
   k8s-mysql:
    Image:      mysql:5.6
    Port:       3306/TCP
    Host Port:  0/TCP
    Environment:
      MYSQL_ROOT_PASSWORD:  <set to the key 'ROOT_PASSWORD' in secret 'mysql-secret'>  Optional: false
    Mounts:
      /var/lib/mysql from data (rw)
  Volumes:  <none>
Volume Claims:
  Name:          data
  StorageClass:
  Labels:        <none>
  Annotations:   <none>
  Capacity:      1Gi
  Access Modes:  [ReadWriteOnce]
Events:
  Type    Reason            Age    From                    Message
  ----    ------            ----   ----                    -------
  Normal  SuccessfulCreate  2m46s  statefulset-controller  create Claim data-mysql-0 Pod mysql-0 in StatefulSet mysql success
  Normal  SuccessfulCreate  2m46s  statefulset-controller  create Pod mysql-0 in StatefulSet mysql successful
  Normal  SuccessfulCreate  2m42s  statefulset-controller  create Claim data-mysql-1 Pod mysql-1 in StatefulSet mysql success
  Normal  SuccessfulCreate  2m42s  statefulset-controller  create Pod mysql-1 in StatefulSet mysql successful
  Normal  SuccessfulCreate  2m38s  statefulset-controller  create Claim data-mysql-2 Pod mysql-2 in StatefulSet mysql success
  Normal  SuccessfulCreate  2m38s  statefulset-controller  create Pod mysql-2 in StatefulSet mysql successful
```
Chek the PV

```
kubectl get pv -n k8s-mysql
```
The output is similar to:
```
pvc-0bb2592e-9fd4-4532-be3d-fa433a85f3d0   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-2        hostpath                36m
pvc-874103c1-3cb8-4130-9871-1d16b87726a4   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-0        hostpath                37m
pvc-c21b36b5-5d2d-4740-99b7-6120a1fa1896   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-1        hostpath                36m
```

Check the PV Claim

```
kubectl get pvc -n k8s-mysql
```
The output is similar to:
```
data-mysql-0   Bound    pvc-874103c1-3cb8-4130-9871-1d16b87726a4   1Gi        RWO            hostpath       41m
data-mysql-1   Bound    pvc-c21b36b5-5d2d-4740-99b7-6120a1fa1896   1Gi        RWO            hostpath       41m
data-mysql-2   Bound    pvc-0bb2592e-9fd4-4532-be3d-fa433a85f3d0   1Gi        RWO            hostpath       41m
```

## Access the replica set

```
 kubectl exec -it mysql-0 -n k8s-mysql -- mysql -u root -p
```
The output is similar to:
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 5.6.51 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
And you can run a mysql command:
```
mysql> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)
```
















