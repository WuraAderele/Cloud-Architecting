# 💻 Build and Deploy a Docker Image to a Kubernetes Cluster

## 📑 Introduction

## 📃 Challenge Scenario

Your development team is interested in adopting a containerized microservices approach to application architecture. 
You need to test a sample application they have provided for you to make sure that it can be 
deployed to a Google Kubernetes container. 

The development group provided a simple Go application called *echo-web* with a Dockerfile and the 
associated context that allows you to build a Docker image immediately.

To test the deployment, you need to download the sample application, then build the Docker container 
image using a tag that allows it to be stored on the Container Registry. 
Once the image has been built, you'll push it out to the Container Registry before you can deploy it.

With the image prepared you can then create a Kubernetes cluster, then deploy the sample application 
to the cluster.



## 🎯 Solution

### Task 1. Create a Kubernetes cluster
Your test environment is limited in capacity, so you should limit the test Kubernetes cluster you are creating to just two e2-standard-2 instances. You must call your cluster echo-cluster.
