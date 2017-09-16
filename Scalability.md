# Multi-pod deployment of OrientDB on Bluemix Kubernetes cluster

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

![alt text](https://github.com/IBM/deploy-graph-db-container/blob/master/images/ThreePodsConnectingToSamePV.png "Three pods connecting to same persistent volume")
