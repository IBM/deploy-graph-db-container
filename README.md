# deploy-graph-db-container

## Prerequisite Step 1. Setting up the Bluemix and Kubernetes CLI

Setup Bluemix and Kubernetes CLI as per instructions in https://console.bluemix.net/docs/containers/cs_tutorials.html#cs_cluster_tutorial. The steps are repeated here for quick reference.

1. Download and Install Bluemix CLI as per instructions in https://clis.ng.bluemix.net/ui/home.html. Bluemix CLI provides the command line interface to manage applications, containers, infrastructures, services and other resources in Bluemix. The prefix for running commands by using the Bluemix CLI is `bx`.

2. Install the IBM Bluemix Container Service plug-in, which allows you to create Kubernetes clusters and manage worker nodes. The prefix for running commands by using the IBM Bluemix Container Service plug-in is `bx cs`.
    ```
    $ bx plugin install container-service -r Bluemix
    ```

3. Install Kubernetes CLI. This allows you to deploy apps into your Kubernetes clusters and to view a local version of the Kubernetes dashboard. The prefix for running commands by using the Kubernetes CLI is `kubectl`.

    Instructions for installing Kubernetes CLI on macOS are given below. Please see https://kubernetes.io/docs/tasks/tools/install-kubectl/ for other methods to install `kubectl` and for instructions to install Kubernetes CLI on other platforms.
    *  Download the Kubernetes CLI
        ```
        $ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl
        ```
    * Make the kubectl binary executable.
        ```
        $ chmod +x kubectl
        ```
    *  Move the binary in to your PATH.
        ```
        $ sudo mv ./kubectl /usr/local/bin/
        ```

## Prerequisite Step 2. Log in to the Bluemix CLI and initialize Bluemix Container Service plugin

1. Log in to the Bluemix CLI. Enter your Bluemix credentials when prompted.
    ```
    $ bx login -a api.ng.bluemix.net
    $ bx target --cf
    ```
    The API endpoint for various Bluemix regions is given below. If you have private Docker images that are stored in the container registry of a specific Bluemix region, or Bluemix services instances that you have already created, log in to this region to access your images and Bluemix services. The Bluemix region that you log in to also determines the region where you can create your Kubernetes clusters, including the available datacenters.

    - US South
        ```
        $ bx login -a api.ng.bluemix.net
        ```
    - United Kingdom
        ```
        $ bx login -a api.eu-gb.bluemix.net
        ```
    - Germany
        ```
        $ bx login -a api.eu-de.bluemix.net
        ```
    - Sydney
        ```
        $ bx login -a api.au-syd.bluemix.net
        ```

2. Initialize the IBM Bluemix Container Service plugin
    ```
    $ bx cs init
    ```
    If you want to create a Kubernetes cluster in a region other than the Bluemix region that you selected earlier, specify this region.

    - US South
      ```
      $ bx cs init --host https://us-south.containers.bluemix.net
      ```
    - UK-South
      ```
      $ bx cs init --host https://uk-south.containers.bluemix.net
      ```
    - EU-Central
      ```
      $ bx cs init --host https://eu-central.containers.bluemix.net
      ```
    - AP-South
      ```
      $ bx cs init --host https://ap-south.containers.bluemix.net
      ```

## Prerequisite Step 3. Setting up your cluster environment

Bluemix allows you to create a free cluster that comes with 2 CPUs, 4 GB memory, and 1 worker node. This is called _lite cluster_ and allows you to get familiar with and test Kubernetes capabilities. However they lack capabilities like persistent NFS file-based storage with volumes.

To setup your cluster for maximum availability and capacity, Bluemix allows you to create a fully customizable, production-ready cluster called _standard cluster_. _Standard clusters_ allow highly available cluster configurations such as a setup with two clusters that run in different regions, each with multiple worker nodes. Please see https://console.bluemix.net/docs/containers/cs_planning.html#cs_planning_cluster_config to review other options for highly available cluster configurations.

A detailed comparison of capabilities of _lite_ and _standard_ clusters is given in https://console.bluemix.net/docs/containers/cs_planning.html#cs_planning.

1. Create your lite Kubernetes cluster.
    ```
    $ bx cs cluster-create --name mycluster1
    ```
    Note: It can take up to 15 minutes for the worker node machine to be ordered and for the cluster to be set up and provisioned.

    In case you want to setup a standard cluster, then you can find the setup instructions in https://console.bluemix.net/docs/containers/cs_cluster.html#cs_cluster_cli.

2. Verify that the deployment of your worker node is complete.
    ```
    $ bx cs clusters
    $ bx cs workers mycluster1
    ```

3. Download the Kubernetes configuration files and get the command to set the environment variable
    ```
    $ bx cs cluster-config mycluster1
    ```
    Set the KUBECONFIG environment variable as per output from above command
    ```
    $ export KUBECONFIG=~/.bluemix/plugins/container-service/clusters/mycluster1/kube-config-hou02-mycluster1.yml
    $ echo $KUBECONFIG
    ```
    Verify that the kubectl commands run properly with your cluster by checking the Kubernetes CLI server version.
    ```
    $ kubectl version  --short
    ```

## Step 1. Deploying OrientDB service into Kubernetes clusters

### 1.1 Copy OrientDB Kubernetes configuration scripts
Clone or download the OrientDB Kubernetes configuration scripts to your user home directory.
```
$ git clone https://github.com/IBM/deploy-graph-db-container.git
```

Navigate to the source directory
```
$ cd deploy-graph-db-container
$ ls
```

### 1.2 Save desired OrientDB password in Kubernetes secret
Create a new file called password.txt in the same directory and put your desired OrientDB password inside password.txt (Could be any string with ASCII characters).
 
We need to make sure password.txt does not have any trailing newline. Use the following command to remove possible newlines.
```
$ tr -d '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
```

Put OrientDB password in Kubernetes [secret](https://kubernetes.io/docs/concepts/configuration/secret/)
```
$ kubectl create secret generic orientdb-pass --from-file=password.txt
```

### 1.3 Configure persistent storage for OrientDB volumes    
[OrientDB docker image](https://hub.docker.com/_/orientdb/) requires following directories to be volume mounted so as to persist data across container delete/relaunch.
```
/orientdb/databases
/orientdb/backup
```

If you are using Bluemix *standard* Kubernetes cluster, then you can leverage [dynamic volume provisioning](http://blog.kubernetes.io/2016/10/dynamic-provisioning-and-storage-in-kubernetes.html) which allows storage volumes to be created on-demand. To use this feature, update the value of `volume.beta.kubernetes.io/storage-class` annotation in `orientdb.yaml` to one of the [NFS file-based storage classes supported in Bluemix](https://console.bluemix.net/docs/containers/cs_apps.html#cs_apps_volume_claim): `ibmc-file-bronze` or `ibmc-file-silver` or `ibmc-file-gold`. Also change `accessModes` to `ReadWriteMany` and increase storage request to say 20GB.
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: orientdb-pv-claim
  labels:
    service: orientdb
    type: pv-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "ibmc-file-gold"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  annotations:
```

In case you are using Bluemix *lite* Kubernetes cluster, where NFS file storage is not supported, you can instead use [hostPath PersistentVolume](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume). A hostPath PersistentVolume uses a file or directory on the Node to emulate network-attached storage. To create a hostPath PersistentVolume, review `local-volumes.yaml` and run `kubectl apply` command.
```
$ cat local-volumes.yaml 
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "pv-volume"
spec:
  capacity:
    storage: "5Gi"
  accessModes:
    - "ReadWriteOnce"
  hostPath:
    path: /tmp/data01

$ kubectl apply -f local-volumes.yaml
```

### 1.4 Deploy OrientDB into Kubernetes cluster
Run the OrientDB Kubernetes configuration script in the cluster. When the deployment and the service are created, OrientDB is available as a service for users.
```
$ kubectl apply -f orientdb.yaml
```

The `orientdb.yaml` script creates a Kubernetes deployment for [OrientDB container](https://hub.docker.com/_/orientdb/). The OrientDB password is fetched from the Kubernetes secret created in Step 1.2 above. Similarly the persistent volumes configured in Step 1.3 above are used as the persistent storage for OrientDB volumes. The corresponding snippet from `orientdb.yaml` script is shown below.
```
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: orientdbservice
  labels:
    service: orientdb
spec:
  replicas: 1
  template:
    metadata:
      name: orientdbservice
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
      volumes:
      - name: orientdb-data
        persistentVolumeClaim:
          claimName: orientdb-pv-claim
```

If you are using Bluemix *standard* Kubernetes cluster, then it is recommended that you increase the number of replicas to 3 or more. You can also spread deployment pods across multiple nodes (anti-affinity) as per instructions in https://console.bluemix.net/docs/containers/cs_planning.html#highly_available_apps
    
The `orientdb.yaml` script also exposes OrientDB ports (HTTP: 2480 and binary: 2424) to the internet by creating a Kubernetes service of type NodePort as shown in the snippet below.
```
kind: Service
apiVersion: v1
metadata:
  name: orientdbservice
  labels:
    service: orientdb
    type: nodeport-service
spec:
  type: NodePort
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

### 1.5 Open OrientDB dashboard
Get information about the deployed OrientDB service to see which NodePort was assigned for OrientDB's HTTP port 2480.
```
$ kubectl describe service orientdbservice
```

Get the public IP address for the worker node in the cluster.
```
$ bx cs workers mycluster1
```

Open a browser and check out the OrientDB dashboard with the following URL.
```
http://<IP_address>:<HTTP_NodePort>/studio/index.html
```    
![alt text](https://github.com/IBM/deploy-graph-db-container/raw/master/images/OrientDB-dashboard.png "OrientDB Dashboard")

### 1.6 View a local version of the Kubernetes dashboard.  
Launch your Kubernetes dashboard with the default port 8001.
```
$ kubectl proxy
```

Open the following URL in a web browser to see the Kubernetes dashboard.
http://localhost:8001/ui
![alt text](https://github.com/IBM/deploy-graph-db-container/raw/master/images/KubernetesDashboard.png "Kubernetes Dashboard")

In the Workloads tab, you can see the resources that you created. When you are done exploring the Kubernetes dashboard, use CTRL+C to exit the proxy command.

## Step 2. Import a public database and explore it using OrientDB Dashboard and Gremlin console

### 2.1 Import a public database

  * In the OrientDB dashboard, click on the cloud import button (next to *New DB*).
  * Specify username (root) and password (same as the value specified in password.txt).
  * Scroll down to *MovieRatings* database and click on import button.
    
    This will import a database containing Movies classified by Genre and Ratings by Users, created by MovieLens (movielens.org).
    
    Once import is successful, you will be taken back to login screen.

### 2.2 Explore schema and data (vertices/edges) using OrientDB dashboard

  * Log in to *MovieRatings* database
    * In the login screen of OrientDB dashboard, select *MovieRatings* under *Database* and specify username (root) and password.
    * Click *Connect*.
  * Click on Schema.
    * Under Vertex Classes, you can see following classes: `Movies, Users, Genres, Occupation`
    * Under Edge Classes, you can see following classes: `rated, hasGenera, hasOccupation`
    * Click on any of the Vertext/Edge classes, like Movies, to see its properties.
  * Click on Browse
    * Run following query:
      ```
      select from Movies
      ```
      The first 10 vertices of *Movies* class (ordered by *id*) will be shown along with its properties and, incoming and outgoing edges.
  * Click on Graph
    * Run following query:
      ```
      select from users where id = 1
      ```
    * Click on the *User* vertex at the center. In the ring that pops up, select outgoing edges, and click on *rated*.
      
      All the movies rated by this user will be shown.
    
    * Click on any of the movie vertices. Under *Settings*, next to *Display*, select *title*.
      
      This will show the movie title below each of the *Movie* vertices as shown in the snapshot below.
      ![alt text](https://github.com/IBM/deploy-graph-db-container/raw/master/images/OrientDB-GraphEditor.png "OrientDB Graph Editor")

### 2.3 Open Gremlin/OrientDB console and run queries

  * Kubernetes allows us to [get a shell to a running container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/). We can use this feature to open [OrientDB's Gremlin console](https://orientdb.com/docs/2.2/Gremlin.html) as shown below.
    ```
    $ kubectl get pods
    NAME                               READY     STATUS    RESTARTS   AGE
    orientdbservice-2043245721-81524   1/1       Running   0          2d
    $ kubectl exec -it orientdbservice-2043245721-81524  -- /orientdb/bin/gremlin.sh

             \,,,/
             (o o)
    -----oOOo-(_)-oOOo-----
    gremlin>
    ```
    Note: Replace the name after `kubectl exec -it` with the name of the pod on which OrientDB is running as obtained by `kubectl get pods` command.

  * As an illustration, from within Gremlin console, we will connect to *MovieRatings* database and display the movies rated by a particular user (say with record id `#16:0`)
    ```
    gremlin> g = new OrientGraph("remote:localhost/MovieRatings");
    ==>orientgraph[remote:localhost/MovieRatings]
    gremlin> g.v('#16:0').outE('rated').inV().title
    ==>One Flew Over the Cuckoo's Nest (1975)
    ==>James and the Giant Peach (1996)
    ==>My Fair Lady (1964)
    ==>Erin Brockovich (2000)
    ==>Bug's Life, A (1998)
    ==>Princess Bride, The (1987)
    ==>Ben-Hur (1959)
    ==>Christmas Story, A (1983)
    ...
    gremlin> exit
    ```

  * We can similarly open OrientDB console and run [OrientDB commands](http://orientdb.com/docs/2.2.x/Console-Commands.html) as shown below.
    ```
    $ kubectl exec -it orientdbservice-2043245721-81524  -- /orientdb/bin/console.sh

    OrientDB console v.2.2.26 (build ae9fcb9c075e1d74560a336a96b57d3661234c7b) https://www.orientdb.com
    Type 'help' to display all the supported commands.
    Installing extensions for GREMLIN language v.2.6.0

    orientdb> CONNECT remote:localhost/MovieRatings root
    Enter password: 

    Connecting to database [remote:localhost/MovieRatings] with user 'root'...OK
    orientdb {db=MovieRatings}> select OUT("rated").title as MovieRated from Users where id = 1 UNWIND MovieRated;

    +----+--------------------------------------+
    |#   |MovieRated                            |
    +----+--------------------------------------+
    |0   |One Flew Over the Cuckoo's Nest (1975)|
    |1   |James and the Giant Peach (1996)      |
    |2   |My Fair Lady (1964)                   |
    |3   |Erin Brockovich (2000)                |
    |4   |Bug's Life, A (1998)                  |
    |5   |Princess Bride, The (1987)            |
    |6   |Ben-Hur (1959)                        |
    |7   |Christmas Story, A (1983)             |
    |8   |Snow White and the Seven Dwarfs (1937)|
    |9   |Wizard of Oz, The (1939)              |
    |10  |Beauty and the Beast (1991)           |
    |11  |Gigi (1958)                           |
    |12  |Miracle on 34th Street (1947)         |
    |13  |Ferris Bueller's Day Off (1986)       |
    |14  |Sound of Music, The (1965)            |
    |15  |Airplane! (1980)                      |
    |16  |Tarzan (1999)                         |
    |17  |Bambi (1942)                          |
    |18  |Awakenings (1990)                     |
    |19  |Big (1988)                            |
    +----+--------------------------------------+
    LIMIT EXCEEDED: resultset contains more items not displayed (limit=20)

    20 item(s) found. Query executed in 0.013 sec(s).
    orientdb {db=MovieRatings}> quit
    $ 
    ```
    Note: Replace the name after `kubectl exec -it` with the name of the pod on which OrientDB is running as obtained by `kubectl get pods` command.
    
    The [OrientDB select query](http://orientdb.com/docs/2.2.x/SQL-Query.html) that was run above displays the movies rated by a specified user (with id = 1).

# Troubleshooting

* If you want to delete the OrientDB service from your Bluemix Kubernetes cluster, you can run the following command.
    ```
    $ kubectl delete -f orientdb.yaml
    ```
* If you want to delete your local persistent volume, you can run the following command.
    ```
    $ kubectl delete -f local-volumes.yaml
    ```
* If you want to delete the Kubernetes sceret containing OrientDB password, you can run the following command.
    ```
    $ kubectl delete secret orientdb-pass
    ```
* For debugging purposes, if you want to inspect the logs of OrientDB service, you can run the following command.
    ```
    $ kubectl get pods # Get the name of the OrientDB pod
    $ kubectl logs [OrientDB pod name]
    ```

# References
* Persistent data storage options in Bluemix Kubernetes Clusters
https://console.bluemix.net/docs/containers/cs_planning.html#cs_planning_apps_storage
* https://kubernetes.io/docs/concepts/storage/volumes/
* https://kubernetes.io/docs/concepts/storage/persistent-volumes/

# License
[Apache 2.0](LICENSE)
