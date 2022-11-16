# k8s 2 tier App with a front-end Deployment and a MySQL StatefulSet as back-end

In this replo we will deploy a 2 tier app. As a front-end we will use PhpMyAdmin with a k8s Deployment  with 3 Repliacas , the PhpMyAdmin will connect to a MySQL StatefulSet as back-end with 3 replicas.

The Mysql Stateful will record data persistently in a persistent volume and can replicate data to the other pods in the same StatefullSet.

A StatefulSet uses a Headless service to provide the domain control of its pods.   

Secrets are used to supply sensitive information to containers.  

## Objetives

- Deploy a simple StatefulSet MySql db wit 3 replicas as a backend. 
- Add a Headless service to provide a stable domain controller of the pods.
- Use  secrets to mount the passwords in the mysql pods.
- Add a database to Mysql throw configmaps 
- Deploy a PhpMyAdmin as a front end with 3 replicas.
- Add a Load balancer to the deployment
- Connect the front end Deployment with the back-end StatefulSet.  
- Test the app.

## Prerequisites 📋

- A k8s cluster v1.9 or later, for testing purposes can be minikube (https://minikube.sigs.k8s.io/docs/). You can check the k8s server with the command kubectl version
- kubectl configured to communicate with the cluster

## Starting 🚀

## Repo description

- Mysql folder
  - mysql-secret.yaml this YAML is used to mount the password as a secret
  - mysql-statefulset.yaml this YAML file describes the Stateful set and associated the pods to the persistent volume
  - mysql-service.yaml this YAML provides the a load balancer service, this could be used to connect to the Mysql StatefulSet only for reading
  - mysql-headless-service.yaml this YAML provides a Headless service to provide domain control of the pods.
  - mysql-secret.yaml this YAML provides a pod to test the connection to the MySql StatefulSet

- PhpMyadmin folder
  - php-deplyment.yaml this YAML provides the phpmyadmin deployment and connect to the Mysql deployment throw the  envs of the PhpMyAdmin docker image
  - php-service.yaml this YAML explores the deploymet

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

### Create the StaefulSet

cd to the mysql directory

#### Create the secret

```
kubectl apply -f mysql-secret.yaml -n k8s-app   
secret/mysql-secret created
```
Check the secret 
```
kubectl describe secret mysql-secret -n k8s-app
```
You will get an output like that:
```
Name:         mysql-secret
Namespace:    k8s-app
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

#### Create the ConfigMap

```
kubectl apply -f .\mysql-configmap.yaml
configmap/mysql-configmap created
```
Check de ConfigMap 

```
kubectl describe configmap mysql-configmap -n k8s-app
Name:         mysql-configmap
Namespace:    k8s-app
Labels:       <none>
Annotations:  <none>

Data
====
MYSQL_DATABASE:
----
todos
Events:  <none>
```

#### Create the Headless Service

```
kubectl apply -f mysql-headless-service.yaml
service/headless-mysql-svc created
```
Check the service
```
kubectl describe svc headless-mysql-svc -n k8s-app
```
You will get an output like that:
```
Name:              headless-mysql-svc
Namespace:         k8s-app
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
#### Create the StatefulSet
```
kubectl apply -f mysql-statefulset.yaml
statefulset.apps/mysql created
```
Check the StatefulSet

```
kubectl describe statefulset mysql -n k8s-app
```
You will get an output like that:

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
kubectl get pv -n k8s-app
```
The output is similar to:
```
pvc-0bb2592e-9fd4-4532-be3d-fa433a85f3d0   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-2        hostpath                36m
pvc-874103c1-3cb8-4130-9871-1d16b87726a4   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-0        hostpath                37m
pvc-c21b36b5-5d2d-4740-99b7-6120a1fa1896   1Gi        RWO            Delete           Bound         k8s-mysql/data-mysql-1        hostpath                36m
```

Check the PV Claim

```
kubectl get pvc -n k8s-app
```
The output is similar to:
```
data-mysql-0   Bound    pvc-874103c1-3cb8-4130-9871-1d16b87726a4   1Gi        RWO            hostpath       41m
data-mysql-1   Bound    pvc-c21b36b5-5d2d-4740-99b7-6120a1fa1896   1Gi        RWO            hostpath       41m
data-mysql-2   Bound    pvc-0bb2592e-9fd4-4532-be3d-fa433a85f3d0   1Gi        RWO            hostpath       41m
```

#### Test the MySql SatefulSet

##### Access the database 

We can access directly to the pod with the pod name in a StatefulSet the pod name is

```
 kubectl exec -it mysql-0 -n k8s-app -- mysql -u root -p
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
| sys                |
| todos              |
+--------------------+
5 rows in set (0.02 sec)
```
As you can see the todos database was created throw the mysql image env in the stateful set introduced by the ConfigMap
 
In the StatefulSet
```
configMapKeyRef:
  name: mysql-configmap
  key: MYSQL_DATABASE     
```
In the ConfigMap
```
data:
  MYSQL_DATABASE: todos
```

##### Test the DNS suport

First we will create a Msql client pod
```
kubectl apply -f mysql-client.yaml -n k8s-app

```
we connect to the pod and install the mysql client

```
kubectl exec --stdin --tty mysql-client -n k8s-app -- sh
apk add mysql-client
exit
```
As the StatefulSet  with the Headless svc provides DNS support we can connect throw the pod dns or the service DNS.
The DNS structure of the pod dns is:

(statefulset)-{0..N-1}.$(service).$(namespace).svc.cluster.local

The DNS of the pod 0 is: 

mysql-0.headless-mysql-svc.k8s-app.svc.cluster.local

we can connect to the database with the pod dns
```
kubectl exec --stdin --tty mysql-client -n k8s-app -- sh
mysql -h mysql-0.headless-mysql-svc.k8s-app.svc.cluster.local -p
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.002 sec)
```
The DNS structure of the service is:

$(service).$(namespace).svc.cluster.local

The NDS of the svc is:

headless-mysql-svc.k8s-app.svc.cluster.local


we can connect to the database with the svc dns
```
/ # mysql -h headless-mysql-svc.k8s-app.svc.cluster.local -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.010 sec)

```
### Create the Deploymet & LoadBalencer

cd to the PhpMyAdmin directory

### Create the svc

```
kubectl apply -f php-service.yaml
kubectl describe svc myadmin-svc -n k8s-ap

Name:                     myadmin-svc
Namespace:                k8s-app
Labels:                   <none>
Annotations:              <none>
Selector:                 app=phpmyadmin
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.97.222.180
IPs:                      10.97.222.180
LoadBalancer Ingress:     localhost
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32574/TCP
Endpoints:                10.1.0.17:80,10.1.0.18:80,10.1.0.19:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### Create the Deplyment


```
kubectl apply -f .\php-deploymet.yaml
deployment.apps/myadmin created

kubectl describe deployment  myadmin -n k8s-app
Name:                   myadmin
Namespace:              k8s-app
CreationTimestamp:      Wed, 16 Nov 2022 11:44:32 +0100
Labels:                 app=phpmyadmin
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=phpmyadmin
Replicas:               3 desired | 3 updated | 3 total | 0 available | 3 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=phpmyadmin
  Containers:
   phpmyadmin:
    Image:      phpmyadmin/phpmyadmin
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:
      PMA_HOST:             headless-mysql-svc.k8s-app.svc.cluster.local
      PMA_PORT:             3306
      MYSQL_ROOT_PASSWORD:  <set to the key 'ROOT_PASSWORD' in secret 'mysql-secret'>  Optional: false
    Mounts:                 <none>
  Volumes:                  <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  <none>
NewReplicaSet:   myadmin-6654fd9b5 (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  33s   deployment-controller  Scaled up replica set myadmin-6654fd9b5 to 3

```

### Access to the deplyment throw the service

```
kubectl get svc -n k8s-app
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
headless-mysql-svc   ClusterIP      None            <none>        3306/TCP       20h
myadmin-svc          LoadBalancer   10.97.222.180   localhost     80:32574/TCP   14m
```

Port forward the service

```
kubectl port-forward service/myadmin-svc 3000:80 -n k8s-app
Forwarding from 127.0.0.1:3000 -> 80
Forwarding from [::1]:3000 -> 80
Handling connection for 3000
Handling connection for 3000
Handling connection for 3000
Handling connection for 3000
```

You can check the PhpMyAdmin in your browser with the url localhost:3000

![front](https://user-images.githubusercontent.com/67485607/202235907-c65e8ae7-48ed-4242-a817-bb153ae2bf2a.PNG)

Login with root /password add check the databases

https://github.com/ezequiellladoce/k8s-2-tier-app/issues/2#issue-1451902123









