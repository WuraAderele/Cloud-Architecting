# 💻 Optimize Costs for Google Kubernetes Engine

## 📑 Introduction
The purpose of this case study is to optimize costs while maintaining performance for the OnlineBoutique application deployed on Google Kubernetes Engine (GKE). This involves creating a K8 cluster, deploying an application, configuring the cluster for cost-efficiency, updating the frontend, and setting up autoscaling for anticipated traffic surges.

This project involved using various skills and technologies, including:

* **Google Cloud Platform (GCP):** Utilized GKE for deploying and managing Kubernetes clusters.
* **Kubernetes:** Managed deployments, namespaces, and pod configurations.
* **Shell Scripting:** Automated cluster and resource management tasks using gcloud and kubectl commands.
* **Docker:** Updated application images and ensured smooth deployment without downtime.
* **Load Testing:** Simulated traffic surges to test the autoscaling configurations.


## 📃 Scenario
OnlineBoutique is an e-commerce platform managed by a team that aims to ensure high performance and cost efficiency. As the lead Google Kubernetes Engine admin, the responsibility is to deploy the OnlineBoutique application to GKE, optimize resource usage, and prepare for traffic surges.

This involves optimizing the node pool, ensuring the application is highly available, and preparing the infrastructure for anticipated traffic spikes due to upcoming marketing campaigns.

**🧩 Definitions of terms**

* **Kubernetes (K8):** automates the deployment, scaling, and management of containerized applications.
* **Namespaces:** A way to isolate groups of resources within a cluster - organizing clusters into virtual sub-clusters.
* **Nodes:** Individual machines that host and run containerized applications.
* **Pods:** Smallest deployable units in K8s that represent a single instance of a running process in the cluster. Pods are hosted by Nodes.
* **Clusters:** A set of nodes that run containerized applications 
* **Node Pools:** Groups of nodes within a K8s cluster that have the same configuration and are managed together.
* **Images:** A blueprint for what you want your containers to look like. They contain the application code and all its dependencies. Images are pulled from container registries like Docker Hub or Google Container Registry.
* **Workloads:** Apps or processes that run on the cluster. Workloads consists of pods.
* **Pod Disruption Budgets:** Defines the minimum number of pods of a particular type (identified by labels) that must remain available at any given time within a specified set of pods.
* **Horizontal Pod Autoscaling:** A Kubernetes feature that automatically scales the number of pods in a deployment, replication controller, or replica set based on observed CPU utilization or other custom metrics. It helps ensure that applications have enough resources to handle varying workloads efficiently without overprovisioning.
* **Cluster Autoscaling:** A feature in Kubernetes that automatically adjusts the number of nodes in a cluster based on the current resource demand. It ensures that there are enough nodes to run the scheduled pods while minimizing unnecessary resource usage during periods of low demand.

## 🎯 Solution

**Summary**

* **Cluster and Application Deployment:** Created a zonal GKE cluster and deployed the OnlineBoutique application to a development namespace.
* **Node Pool Optimization:** Migrated the application to a new node pool with custom machine type to optimize resource usage and reduce costs.
* **Frontend Update:** Applied a last-minute frontend update using a new Docker image without causing downtime.
* **Autoscaling:** Configured horizontal pod autoscaling for the frontend and recommendation service to handle anticipated traffic surges. Enabled cluster autoscaling to automatically adjust node count based on demand.

### Task 1 - Creating a cluster and deploying the app
We want to create a zonal cluster with 2 nodes to provide a stable environment for deploying  the OnlineBoutique application:

        gcloud container clusters create onlineboutique-cluster-663 \
        --zone=us-central1-c \
        --machine-type=e2-standard-2 \
        --release-channel=rapid \
        --num-nodes=2

Create two namespaces (dev and prod) to separate resources on the onlineboutique-cluster-663 cluster:

        kubectl create namespace dev && \
        kubectl create namespace prod

We will be working with resources in the dev namespace for now so we run the below command to set the environment context to dev:

        kubectl config set-context --current --namespace=dev
        
Clone the  OnlineBoutique repository and deploy the application to the dev namespace by running below command:

    git clone https://github.com/GoogleCloudPlatform/microservices-demo.git &&
    cd microservices-demo && kubectl apply -f ./release/kubernetes-manifests.yaml --namespace dev

Because we did not specify a node pool, the application is automatically deployed on the *default-pool*.

### Task 2 - Migrating to an optimized node pool

After successfully deploying the app to the dev namespace, we take a look at the node details via Google Cloud Console:

<p align = "center">
  <img src="https://cdn.qwiklabs.com/uYinv2VTYdTkOsSsWGkvXvD%2FmTd4IsJXz4vkrNyl9no%3D" width = "350" height = "100"/>

We come to the conclusion that we should make changes to the cluster's node pool:

* There's plenty of left over RAM from the current deployments so we should be able to use a node pool with machines that offer less RAM.
* Most of the deployments that you might consider increasing the replica count of will require only 100mcpu per additional pod. We could potentially use a node pool with less total CPU if you configure it to use smaller machines. However, we also need to consider how many deployments will need to scale, and how much they need to scale by.

We will create a new node pool called optimized-pool-6437 with custom-2-3584 as the machine type to better match resource requirements, optimizing costs and performance based on observed application needs.

        gcloud container node-pools create optimized-pool-6437 \
        --cluster=onlineboutique-cluster-663 \
        --machine-type=custom-2-3584 \
        --num-nodes=2 \
        --zone=us-central1-c

Once the new node pool is set up, we migrate our application's deployments to the new nodepool by cordoning off and draining default-pool. This means we safely evacuate the existing deployment to the new optimized node pool without disrupting the app's availability.

        for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
          kubectl cordon "$node";
        done

        for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
          kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
        done

Once the deployments have been safely migrated, we delete the old default-pool to release unused resources.

        gcloud container node-pools delete default-pool --cluster=onlineboutique-cluster-663 --zone=us=central1-c

### Task 3 - Applying a frontend update
We have made all necessary deployments, and now the dev team wants us to push a last-minute update before an upcoming release. The team changed the file used for the application's home page banner and have shared an updated docker image. This update has to be achieved without causing down time.

The frontend deployment is one of the workloads in the Onlineboutique application. It is the workload that services the app's home page. Hence, it is the deployment that needs to be modified.

First, we set a pod disruption budget for our frontend deployment to ensure that at least one instance remains available during updates, maintaining high availability.

        kubectl create poddisruptionbudget onlineboutique-frontend-pdb \
        --selector app=frontend \
        --min-available=1

Next, we will edit the configuration file for the frontend deployment to update its image by running the below command:

        kubectl edit deployment frontend

This will open up the configuration yaml file in a linux text editor and we will edit the **Image** property to the updated image provided (gcr.io/qwiklabs-resources/onlineboutique-frontend:v2.1) and edit the **ImagePullPolicy** property to Always.

### Task 4 - Autoscaling from estimated traffic

A marketing campaign is coming up that will cause a traffic surge on the OnlineBoutique shop. Normally, we would spin up extra resources in advance to handle the estimated traffic spike. However, if the traffic spike is larger than anticipated, we may get woken up in the middle of the night to spin up more resources to handle the load.

We also want to avoid running extra resources for any longer than necessary. To both lower costs and save ourselves a potential headache, we can configure the Kubernetes deployments to scale automatically when the load begins to spike.

First we apply horizontal pod autoscaling to the frontend deployment in order to handle the traffic surge. This allows K8s to automatically adjust the number of pod replicas based on CPU utilization:

        kubectl autoscale deployment frontend \
        --cpu-percent=50 \
        --min=1 \
        --max=13

We also need to ensure that the cluster is able to automatically spin up additional compute nodes if necessary. This way, the cluster can add nodes when traffic is high, and reduce the number of nodes when traffic is low.

        gcloud container clusters update onlineboutique-cluster-663 \
        --enable-autoscaling \
        --min-nodes=1 \
        --max-nodes=6

Lastly, we will run a load test to simulate traffic surge to ensure that the cluster can now dynamically scale pods and nodes as needed:

        kubectl exec $(kubectl get pod --namespace=dev | grep 'loadgenerator' | cut -f1 -d ' ') 
        -it --namespace=dev -- bash -c 'export USERS=8000; locust --host="http://34.44.26.32:80" 
        --headless -u "8000" 2>&1'

After running this command, we observed our workloads on Google Cloud Console and monitored how the cluster handles the traffic spikes. We can see the recommendationservice (another workload) struggling from increased demand so we apply horizontal pod autoscaling to the recommendationservice deployment.

        kubectl autoscale deployment recommendationservice \
        --cpu-percent=50 \
        --min=1 \
        --max=5

