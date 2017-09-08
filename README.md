# deploy-graph-db-container

## Step 1. Setting up the Bluemix and Kubernetes CLI

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

## Step 2. Log in to the Bluemix CLI and initialize Bluemix Container Service plugin

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

## Step 3. Setting up your cluster environment

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

## Step 4. Deploying OrientDB service into Kubernetes clusters

1. Clone or download the OrientDB Kubernetes configuration scripts to your user home directory.
    ```
    $ git clone https://github.com/IBM/deploy-graph-db-container.git
    ```
    Navigate to the source directory
    ```
    $ cd deploy-graph-db-container
    $ ls
    ```

2. Setup OrientDB Secrets
    
    Create a new file called password.txt in the same directory and put your desired OrientDB password inside password.txt (Could be any string with ASCII characters).

    We need to make sure password.txt does not have any trailing newline. Use the following command to remove possible newlines.
    ```
    $ tr -d '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
    ```

    Put OrientDB password in Kubernetes [secret](https://kubernetes.io/docs/concepts/configuration/secret/)
    ```
    $ kubectl create secret generic orientdb-pass --from-file=password.txt
    ```

3. Run the OrientDB Kubernetes configuration script in the cluster. When the deployment and the service are created, OrientDB is available as a service for users.
    ```
    $ kubectl apply -f local-volumes.yaml
    $ kubectl apply -f orientdb.yaml
    ```

4. Open OrientDB dashboard
    Get information about the service to see which NodePort was assigned.
    ```
    $ kubectl describe service orientdbservice
    ```
    Get the public IP address for the worker node in the cluster.
    ```
    $ bx cs workers mycluster1
    ```
    Open a browser and check out the OrientDB dashboard with the following URL: http://&lt;IP_address&gt;:&lt;NodePort&gt;/studio/index.html

5. View a local version of the Kubernetes dashboard.
    Launch your Kubernetes dashboard with the default port 8001.
    ```
    $ kubectl proxy
    ```
    Open the following URL in a web browser to see the Kubernetes dashboard.
    http://localhost:8001/ui

    In the Workloads tab, you can see the resources that you created. When you are done exploring the Kubernetes dashboard, use CTRL+C to exit the proxy command.


## Step 5. Deleting the service when it is no more needed

* In case you want to delete the OrientDB service from your Bluemix Kubernetes cluster, then run the following command:
    ```
    $ kubectl delete -f orientdb.yaml
    ```

***References***
* Persistent data storage options in Bluemix Kubernetes Clusters
https://console.bluemix.net/docs/containers/cs_planning.html#cs_planning_apps_storage
* https://kubernetes.io/docs/concepts/storage/volumes/
* https://kubernetes.io/docs/concepts/storage/persistent-volumes/
