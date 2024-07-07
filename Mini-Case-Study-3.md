# Optimize Costs for Google Kubernetes Engine
## Scenario
You are the lead Google Kubernetes Engine admin on a team that manages the online shop for 
OnlineBoutique.

You are ready to deploy your team's site to Google Kubernetes Engine but you are still looking for 
ways to make sure that you're able to keep costs down and performance up.

You will be responsible for deploying the OnlineBoutique app to GKE and making some configuration 
changes that have been recommended for cost optimization.

### Task 1 - Creating a cluster and deploy your app
We want to create a zonal cluster with 2 nodes:

        gcloud container clusters create onlineboutique-cluster-663 \
        --zone=us-central1-c \
        --machine-type=e2-standard-2 \
        --release-channel=rapid \
        --num-nodes=2

Create two namespaces to separate resources on the onlineboutique-cluster-663 cluster:

        kubectl create namespace dev && \
        kubectl create namespace prod

We will be working with resources in the dev namesoace for now so we run the below command to set the encironment context to dev:

        kubectl config set-context --current --namespace=dev
        
Deploy the applicaation to the dev namespace by running below command:

    git clone https://github.com/GoogleCloudPlatform/microservices-demo.git &&
    cd microservices-demo && kubectl apply -f ./release/kubernetes-manifests.yaml --namespace dev

### Task 2 - Migrating to an optimized node pool

After successfully deploying the app to the dev namespace, we take a look at the node details via Google Cloud Console:

<p align = "center">
  <img src="https://cdn.qwiklabs.com/uYinv2VTYdTkOsSsWGkvXvD%2FmTd4IsJXz4vkrNyl9no%3D" width = "350" height = "100"/>

We come to the conclusion that we should make changes to the cluster's node pool:

* There's plenty of left over RAM from the current deployments so we should be able to use a node pool with machines that offer less RAM.
* Most of the deployments that you might consider increasing the replica count of will require only 100mcpu per additional pod. We could potentially use a node pool with less total CPU if you configure it to use smaller machines. However, we also need to consider how many deployments will need to scale, and how much they need to scale by.

We will create a new node pool called optimized-pool-6437 with custom-2-3584 as the machine type.

        gcloud container node-pools create optimized-pool-6437 \
        --cluster=onlineboutique-cluster-663 \
        --machine-type=custom-2-3584 \
        --num-nodes=2 \
        --zone=us-central1-c

Once the new node pool is set up, we migrate our application's deployments to the new nodepool by cordoning off and draining default-pool.

        for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
          kubectl cordon "$node";
        done

        for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
          kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
        done

Now we can delate the default-pool since the deployments have been safely migrated

gcloud container node-pools delete default-pool --cluster=onlineboutique-cluster-663 --zone=us=central1-c

### Task 3 - Applying a frontend update
We have made all necessary deployments, and now the dev team wants us to push a last-minute update before an upcoming release. The teak changed the file used for the application's home page's banner and have provided us with an updated docker image. This update has to be achieved without causing down time.

First, we set a pod disruption budget for our frontend deployment.

        kubectl create poddisruptionbudget onlineboutique-frontend-pdb \
        --selector app=frontend \
        --min-available=1

Next, we will edit the configuration file for the frontend deployment by running the below command:

        kubectl edit deployment frontend

This will open up the configuration yaml file in a linux text editor and we will edit the **Image** property to the updated image provided (gcr.io/qwiklabs-resources/onlineboutique-frontend:v2.1) and edit the **ImagePullPolicy** property to Always.

### Task 4 - Autoscaling from estimated traffic

A marketing campaign is coming up that will cause a traffic surge on the OnlineBoutique shop. Normally, we would spin up extra resources in advance to handle the estimated traffic spike. However, if the traffic spike is larger than anticipated, we may get woken up in the middle of the night to spin up more resources to handle the load.

We also want to avoid running extra resources for any longer than necessary. To both lower costs and save ourselves a potential headache, we can configure the Kubernetes deployments to scale automatically when the load begins to spike.

First we apply horizontal pod autoscaling to the frontend deployment in order to handle the traffic surge:

        kubectl autoscale deployment frontend \
        --cpu-percent=50 \
        --min=1 \
        --max=13

We also need to ensure that the cluster is able to automatically spin up additional compute nodes if necessary. This way, the cluster can add nodes when traffic is high, and reduce the number of nodes when traffic is low.

        gcloud container clusters update onlineboutique-cluster-663 \
        --enable-autoscaling \
        --min-nodes=1 \
        --max-nodes=6

Lastly, we will run a load test to simulate traffic surge:

        kubectl exec $(kubectl get pod --namespace=dev | grep 'loadgenerator' | cut -f1 -d ' ') 
        -it --namespace=dev -- bash -c 'export USERS=8000; locust --host="http://34.44.26.32:80" 
        --headless -u "8000" 2>&1'

After running this command, we observed our workloads on Google Cloud Console and monitored how the cluster handles the traffic spikes. We can see the recommendationservice struggling from increased demand so we apply horizontal pod autoscaling to the recommendationservice deployment.

        kubectl autoscale deployment recommendationservice \
        --cpu-percent=50 \
        --min=1 \
        --max=5

