# 💻 Build and Deploy a Docker Image to a Kubernetes Cluster

## 📑 Introduction
In this case study, we deploy a simple web server containerized application to a Google Kubernetes Engine (GKE) cluster.

## 📃 Challenge Scenario

Your development team is interested in adopting a containerized microservices approach to application architecture. 
You need to test a sample application they have provided for you to make sure that it can be 
deployed to a Google Kubernetes container. 

The development group provided a simple Go application called *echo-web* with a Dockerfile and the associated context that allows you to build a Docker image immediately.

To test the deployment, you need to download the sample application, then build the Docker container image using a tag that allows it to be stored on the Container Registry. 
Once the image has been built, you'll push it out to the Container Registry before you can deploy it.

With the image prepared you can then create a Kubernetes cluster, then deploy the sample application to the cluster.

## 🎯 Solution
First things first, we want to define the specific project and zone where subsequent resources will be created in. To do that, we run the following commands:

          gcloud config set project $DEVSHELL_PROJECT_ID
          gcloud config set compute/zone us-east1-b

In Cloud Shell, the $DEVSHELL_PROJECT_ID is a default environment variable that contains your project ID.

After the above commands executes, we no longer have to specify project id and zones in subsequent commands.

### Task 1. Create a Kubernetes cluster
Your test environment is limited in capacity, so you should limit the test Kubernetes cluster you are creating to just two e2-standard-2 instances. You must call your cluster echo-cluster.

In Kubernetes (K8s), a cluster consists of at least one cluster control plane machine and multiple worker machines called nodes. Nodes are Compute Engine virtual machine (VM) instances that run the K8s processes necessary to make them part of the cluster. You deploy applications to clusters, and the applications run on the nodes.

        gcloud containers clusters create echo-cluster \
        --num-nodes=2
        --machine-type=e2-standard-2

### Task 2. Build a tagged Docker image
The sample application, including the Dockerfile and the application context files, are contained in an archive called echo-web.tar.gz. The archive has been copied to a Cloud Storage bucket belonging to your lab project called gs://PROJECT_ID. You must deploy this with a tag called v1.

First we need to download/copy the application we want to deploy and its associated file from where they are currently located. To do that, we will run the below command:

        gsutil cp gs://$DEVSHELL_PROJECT_ID/echo-web.tar.gz .

The files are all in a zipped folder so we need to extract them by running below command:

        tar -xzvf echo-web.tar.gz

After this, you can run the common **ls** command to see a lot of all the files you will be working with and also to confirm that the Dockerfile you need to build an image is also available.

A docker image is a starting point for a Docker container. It is a blueprint for what you want your applications to look like. The dockerfile is how you build a docker image. It is a text file with a series of commands that tells the Operating Sytem what to do when building a container.

Now that we have that down, and we have confirmed that the Dockerfile is one on the files we have extracted into the current directory, we will run the below command to build a docker image called **echo-app:v1**:

        docker build -t gcr.io/$DEVSHELL_PROJECT_ID/echo-app:v1 .

### Task 3. Push the image to the Google Container Registry
Your organization has decided that it will always use the gcr.io Container Registry hostname for all projects. The sample application is a simple web application that reports some data describing the configuration of the system where the application is running. It is configured to use TCP port 8000 by default.

A container registry is a tool that is used to store and distribute container images. In this case, we want to store the echo-app:v1 docker image in the gcr.iogcr.io/$DEVSHELL_PROJECT_ID container registry.

The docker push command uploads Docker images to a registry. This process is similar to sharing a pre-assembled software package with others. By pushing an image, you are sharing the complete setup of an application, including its required components. This allows others to access and deploy the application as a container in their environments. 

        docker push gcr.io/$DEVSHELL_PROJECT_ID/echo-app:v1

### Task 4. Deploy the application to the Kubernetes cluster
Even though the application is configured to respond to HTTP requests on port 8000, you must configure the service to respond to normal web requests on port 80. When configuring the cluster for your sample application, call your deployment echo-web.

Run below command to confirm that the docker image is now available in the Google Container Registry:

        gcloud container images list

This will output a list of the available images and you can copy the name of the image you intend to use as the blueprint for your application and use it in the next command.

        kubectl create deployment echo-web \
            --image=gcr.io/$DEVSHELL_PROJECT_ID/echo-app:v1

After deploying the application, you need to expose it to the internet so that users can access it. You can expose your application by creating a Service, a Kubernetes resource that exposes your application to external traffic.

To expose your application, run the following kubectl expose command:

        kubectl expose deployment echo-web \
            --type LoadBalancer \
            --port 80 \
            --target-port 8000

Passing in the **--type LoadBalancer** flag creates a Compute Engine load balancer for your container. The **--port flag** initializes public port 80 to the internet and the **--target-port** flag routes the traffic to port 8080 of the application.

Inspect the echo-web Service by using kubectl get service command:
      
        kubectl get service echo-web

This command will output the services external IP alongside some other information. You can visit the IP in your web browser to confirm the service is now running or run below command in cloud shell:

        curl http://[EXTERNAL_IP_ADDRESS]
