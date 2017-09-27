# Multi-pod deployment of OrientDB on Bluemix Kubernetes cluster

## Impact of simply increasing the number of replicas in existing ReplicationController
In orientdb.yaml file, if we simply increase the number of replicas, to say 3, then Kubernetes will create 3 pods (all running OrientDB) but all connected to same persistent volume as illustrated in the diagram below.

```
$ kubectl get pods
NAME                               READY     STATUS    RESTARTS   AGE
orientdbservice-2043245721-5tdv3   1/1       Running   0          14s
orientdbservice-2043245721-p7b80   1/1       Running   0          14s
orientdbservice-2043245721-t4vm7   1/1       Running   0          14s

$ kubectl describe service orientdbservice
Name:			orientdbservice
Namespace:		default
Labels:			service=orientdb
			type=nodeport-service
Annotations:		kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"service":"orientdb","type":"nodeport-service"},"name":"orientdbservice","na...
Selector:		service=orientdb,type=container-deployment
Type:			NodePort
IP:			10.10.10.45
Port:			binary	2424/TCP
NodePort:		binary	31079/TCP
Endpoints:		172.30.199.27:2424,172.30.199.28:2424,172.30.199.29:2424
Port:			http	2480/TCP
NodePort:		http	31611/TCP
Endpoints:		172.30.199.27:2480,172.30.199.28:2480,172.30.199.29:2480
Session Affinity:	None
Events:			<none>

$ kubectl get pv
NAME          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                       STORAGECLASS   REASON    AGE
pv-volume   5Gi        RWO           Recycle         Bound     default/orientdb-pv-claim                            4m

$ kubectl get pvc
NAME                STATUS    VOLUME        CAPACITY   ACCESSMODES   STORAGECLASS   AGE
orientdb-pv-claim   Bound     pv-volume   5Gi        RWO                          4m

$  bx cs workers mycluster1
OK
ID                                                 Public IP       Private IP      Machine Type   State    Status 
kube-hou02-pa87e7526bacb74c79a92408445066e77e-w1   173.193.99.72   10.76.193.101   free           normal   Ready
```

<img src="doc/source/images/ThreePodsConnectingToSamePV.png" alt="Three pods connecting to same persistent volume" width="400" border="10" />

## Make sure each pod gets its own PersistentVolume independent of others by using Kubernetes StatefulSets
  * [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) allow each pod to make its own PersistentVolumeClaim as shown below.
```
kind: StatefulSet
apiVersion: "apps/v1beta1"
metadata:
  name: orientdbservice
spec:
  serviceName: orientdbservice
  replicas: 3
  template:
    metadata:
      labels:
        service: orientdb
        type: container-deployment
    spec:
      containers:
      - name: orientdbservice
        image: orientdb:2.2.26
        env:
        - name: ORIENTDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: orientdb-pass
              key: password.txt
        ports:
        - containerPort: 2424
          name: port-binary
        - containerPort: 2480
          name: port-http
        volumeMounts:
        - mountPath: /orientdb/databases
          name: orientdb-data
          subPath: databases
        - mountPath: /orientdb/backup
          name: orientdb-data
          subPath: backup
  volumeClaimTemplates:
  - metadata:
      name: orientdb-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

  * Kubernetes creates one PersistentVolume for each VolumeClaimTemplate. And since we are planning to have 3 OrientDB nodes, we will need to create 3 Persistent Volumes
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pv-volume-1"
  labels:
    type: local
spec:
  capacity:
    storage: "5Gi"
  accessModes:
    - "ReadWriteOnce"
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pv-volume-2"
  labels:
    type: local
spec:
  capacity:
    storage: "5Gi"
  accessModes:
    - "ReadWriteOnce"
  hostPath:
    path: /tmp/data02
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pv-volume-3"
  labels:
    type: local
spec:
  capacity:
    storage: "5Gi"
  accessModes:
    - "ReadWriteOnce"
  hostPath:
    path: /tmp/data03
  persistentVolumeReclaimPolicy: Recycle
```

  * Kubernetes StatefulSets currently require a [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) to be responsible for the network identity of the Pods. Hence we need to create this service as shown below.
```
kind: Service
apiVersion: v1
metadata:
  name: orientdbservice
  labels:
    service: orientdb
    type: headless-service
spec:
  clusterIP: None
  selector:
    service: orientdb
    type: container-deployment
  ports:
  - protocol: TCP
    port: 2424
    name: binary
  - protocol: TCP
    port: 2480
    name: http
```

This creates 3 OrientDB pods each having its own hostPath PersistentVolume as shown below.
```
$ kubectl get pods
NAME                READY     STATUS    RESTARTS   AGE
orientdbservice-0   1/1       Running   0          4h
orientdbservice-1   1/1       Running   0          4h
orientdbservice-2   1/1       Running   0          4h

$ kubectl describe service orientdbservice
Name:			orientdbservice
Namespace:		default
Labels:			service=orientdb
			type=headless-service
Annotations:		<none>
Selector:		service=orientdb,type=container-deployment
Type:			ClusterIP
IP:			None
Port:			binary	2424/TCP
Endpoints:		172.30.199.31:2424,172.30.199.32:2424,172.30.199.33:2424
Port:			http	2480/TCP
Endpoints:		172.30.199.31:2480,172.30.199.32:2480,172.30.199.33:2480
Session Affinity:	None
Events:			<none>

$ kubectl get pv
NAME          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                                     STORAGECLASS   REASON    AGE
pv-volume-1   5Gi        RWO           Recycle         Bound       default/orientdb-data-orientdbservice-2                            4h
pv-volume-2   5Gi        RWO           Recycle         Bound       default/orientdb-data-orientdbservice-0                            4h
pv-volume-3   5Gi        RWO           Recycle         Bound       default/orientdb-data-orientdbservice-1                            4h

$ kubectl get pvc
NAME                              STATUS    VOLUME        CAPACITY   ACCESSMODES   STORAGECLASS   AGE
orientdb-data-orientdbservice-0   Bound     pv-volume-2   5Gi        RWO                          4h
orientdb-data-orientdbservice-1   Bound     pv-volume-3   5Gi        RWO                          4h
orientdb-data-orientdbservice-2   Bound     pv-volume-1   5Gi        RWO                          4h
```
<img src="doc/source/images/StatefulSets-EachPodsHasItsOwnPV.png" alt="StatefulSets make sure each Pod gets its own PersistentVolume" width="400" border="10" />

However, in this setup the three OrientDB nodes are independent of each other with no database replication setup among them. This can be confirmed as shown below.
 * Connect to first OrientDB pod and create a new database
```
$ kubectl exec -it orientdbservice-0 -- /orientdb/bin/console.sh

OrientDB console v.2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b) https://www.orientdb.com
Type 'help' to display all the supported commands.
Installing extensions for GREMLIN language v.2.6.0

orientdb> CREATE DATABASE remote:localhost/MovieRatings root <pwd> PLOCAL

Creating database [remote:localhost/MovieRatings] using the storage type [plocal]...
Connecting to database [remote:localhost/MovieRatings] with user 'root'...OK
Database created successfully.

Current database is: remote:localhost/MovieRatings
orientdb {db=MovieRatings}> exit;
```

  * Reconnect to first OrientDB pod and make sure database exists, and you can connect to it successfully.
```
$ kubectl exec -it orientdbservice-0 -- /orientdb/bin/console.sh

OrientDB console v.2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b) https://www.orientdb.com
Type 'help' to display all the supported commands.
Installing extensions for GREMLIN language v.2.6.0

orientdb> CONNECT remote:localhost/MovieRatings root <pwd>

Connecting to database [remote:localhost/MovieRatings] with user 'root'...OK
orientdb {db=MovieRatings}> exit;
```

  * Connect to the second OrientDB pod and try to connect to the database created in first node. The connection fails with a message saying database doesn't exist. This is due to the lack of database replication among first and second OrientDB pods.
```
$ kubectl exec -it orientdbservice-1 -- /orientdb/bin/console.sh

OrientDB console v.2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b) https://www.orientdb.com
Type 'help' to display all the supported commands.
Installing extensions for GREMLIN language v.2.6.0

orientdb> CONNECT remote:localhost/MovieRatings root <pwd>

Connecting to database [remote:localhost/MovieRatings] with user 'root'...
Error: com.orientechnologies.orient.core.exception.OConfigurationException: Database 'MovieRatings' is not configured on server (home=/orientdb/databases/)

orientdb> exit
```

  * Connect to the third OrientDB pod and try to connect to the database created in first node. The connection fails again with a message saying database doesn't exist. This again is due to the lack of database replication among different OrientDB pods.
```
$ kubectl exec -it orientdbservice-2 -- /orientdb/bin/console.sh

OrientDB console v.2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b) https://www.orientdb.com
Type 'help' to display all the supported commands.
Installing extensions for GREMLIN language v.2.6.0

orientdb> CONNECT remote:localhost/MovieRatings root <pwd>

Connecting to database [remote:localhost/MovieRatings] with user 'root'...
Error: com.orientechnologies.orient.core.exception.OConfigurationException: Database 'MovieRatings' is not configured on server (home=/orientdb/databases/)

orientdb> exit
```

To delete the StatefulSets, PersistentVolumeClaims and Headless Service, you can run the following command.
```
kubectl delete statefulset,pvc,svc -l service=orientdb
```

To delete the local volumes, you can run the following command.
```
kubectl delete pv -l type=local
```
