# 💻 Develop your Google Cloud Network

## 📑 Introduction

The purpose of this case study is to demonstrate my expertise in cloud engineering and Kubernetes, specifically within the context of setting up a development environment for a new team at Jooli, Inc.

As a cloud engineer at Jooli, Inc., I was tasked with creating a robust and scalable development infrastructure using Google Cloud Platform (GCP) services and Kubernetes. This project required the utilization of various skills and technologies, including GCP, shell scripting, Kubernetes cluster management, and MySQL database setup.

## 📃 Detailed Scenario

Jooli, Inc. is a forward-thinking technology company that continuously seeks to innovate and streamline its operations. The company recently onboarded a new team, Griffin, to work on a critical project involving the use of WordPress. The Griffin team has started setting up their environment but needed expert assistance to complete it efficiently and effectively.

There are several specific requirements and constraints provided by Jooli, Inc.:

* All resources have to be created in the us-east1 region and us-east1-c zone to maintain consistency and optimize network performance.
* The project was under strict budget monitoring, and resource allocation has to be cost-effective, with a preference for using e2-medium machine types unless otherwise directed.
* Standard naming conventions should be followed, typically using the format team-resource (e.g., griffin-webserver1).
* The infrastructure needs to include a development VPC with subnets, a production VPC with subnets, a bastion host connected to both VPCs, a Cloud SQL instance configured for WordPress, and a Kubernetes cluster to host the WordPress environment.
* Security and access management are crucial, requiring the creation of firewall rules and provision of access for an additional engineer.
  
This case study outlines the steps taken to meet these requirements and successfully set up the development environment for the Griffin team.

## 🎯 Solution

### Summary

<p align = "center">
  <img src="https://cdn.qwiklabs.com/UE5MydlafU0QvN7zdaOLo%2BVxvETvmuPJh%2B9kZxQnOzE%3D" />



Run below commands to set the enviroment variables:

REGION=us-east1
ZONE=us-east1-c
PROJECT=<project_id here>

### Task 1 - Create development VPC with 2 subnets
First, we create the development VPC by running the below command:

      gcloud compute networks create griffin-dev-vpc --project=$PROJECT --subnet-mode=custom

To create the two subnets in the development VPC, we run the below commands:

        gcloud compute networks subnets create griffin-dev-wp --project=$PROJECT --range=192.168.16.0/20 --network=griffin-dev-vpc --region=$REGION

        gcloud compute networks subnets create griffin-dev-mgmt --project=$PROJECT --range=192.168.32.0/20 --network=griffin-dev-vpc --region=$REGION

### Task 2 - Create production VPC with 2 subnets

First, we create the production VPC by running the below command:

      gcloud compute networks create griffin-prod-vpc --project=$PROJECT --subnet-mode=custom

To create the two subnets in the production VPC, we run the below commands:

      gcloud compute networks subnets create griffin-prod-wp --project=$PROJECT --range=192.168.48.0/20 --network=griffin-prod-vpc --region=$REGION
      
      gcloud compute networks subnets create griffin-prod-mgmt --project=$PROJECT --range=192.168.64.0/20 --network=griffin-prod-vpc --region=$REGION

### Task 3 - Create a bastion host
We want to  create a bastion host with two network interfaces, one connected to griffin-dev-mgmt and the other connected to griffin-prod-mgmt.

To do this, we use the Google Cloud Console and follow the below steps:
* Search for
* Click **create instance**
* Fill in the instance name, machine type, region, zion
* Click on
* Click the Network Interface dropdown

Next, we create two firewal rules for each of our VPCs y running the below commands:

      gcloud compute firewall-rules create griffin-firewall-rule1 --direction=INGRESS --priority=100 --network=griffin-dev-vpc --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0
      
      gcloud compute firewall-rules create griffin-firewall-rule2 --direction=INGRESS --priority=100 --network=griffin-prod-vpc --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0


### Task 4 - Create and configure Cloud SQL
We want to create a MySQL Cloud Instance called griffin-dev-db in the us-east1 region. To achieve this, we will use the interactive Google Console and take the following steps:
* CREATE INSTANCE > Choose MySQL
* Enter instance id as griffin-dev-db
* Enter a secure password in the **Password**
* Select the database version, Cloud SQL Edition, Preset
* Click CREATE INSTANCE

Next, we want to set up auth for the Cloud SQL instance. We will achieve that in the following steps:

First, run the below command:

      gcloud auth login --no-launch-browser

This command will output a link in Cloud Shell. Open the link in your browser and copy the verification code that is shown. Paste that code in the Cloud Shell.

Next, run the following command to connect to the SQL instance:

      gcloud sql connect griffin-dev-db --user=root --quiet

When prompted, enter the root password you set for the instance.

Next we run the below SQL commands to prepare the WordPress environment:

      CREATE DATABASE wordpress;
      CREATE USER "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
      GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%";
      FLUSH PRIVILEGES;

### Task 5 - Create Kubernetes cluster
We want to create a 2 node zonal cluster of machine type e2-standard-4 called griffin-dev in the griffin-dev-wp and in zone us-east1-c. To achieve this, we run the below command:

      gcloud container clusters create griffin-dev --zone=us-east1-c --node-locations=us-east1-c --network=griffin-dev-vpc --subnetwork=griffin-dev-wp --num-nodes=2 --machine-type=e2-standard-4

### Task 6 - Prepare the Kubernetes cluster
From Cloud Shell, we want to copy all files from gs://cloud-training/gsp321/wp-k8s. We do this by running the below command:

      gsutil cp -r gs://cloud-training/gsp321/wp-k8s .

The WordPress server needs to access the MySQL database using the username and password that was created in task 4. To achieve this, we follow the below steps:

First, configure the username and passowrd to wp_user and stormwind_rules respectively by editing the wp-env.yaml file. We do this by opening the file in a text editor and making the necessary changes:
       vi wp-k8s/wp-env.yaml

We run below command to create a key:

      gcloud iam service-accounts keys create key.json \
          --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
      
      kubectl create secret generic cloudsql-instance-credentials \
          --from-file key.json

We run below command to add the key to the kubernetes environment:

      kubectl apply -f wp-k8s/wp-env.yaml

### Task 7 - Create a WordPress deployment
Now that we have provisioned the MySQL database, and set up the secrets and volume, we can create the deployment using wp-deployment.yaml.

First, we edit the wp-deployment.yaml to include the instance connection name of the Cloud SQL instance.

       vi wp-k8s/wp-deployment.yaml

Save and verify the changes to the file before proceeding to deploy the word press by running the bellow commands:

      kubectl apply -f wp-env.yaml
      kubectl apply -f wp-deployment.yaml
      kubectl apply -f wp-service.yaml
