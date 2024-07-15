# 💻 Cloud-Enabled WordPress Deployment and Infrastructure Automation

## 📑 Introduction

The purpose of this case study is to demonstrate my expertise in cloud engineering and Kubernetes, specifically within the context of setting up a development environment for a new team at Jooli, Inc.

As a cloud engineer at Jooli, Inc., I was tasked with creating a robust and scalable development infrastructure using Google Cloud Platform (GCP) services and Kubernetes. This project required the utilization of various skills and technologies, including GCP, shell scripting, Kubernetes cluster management, and MySQL database setup.

## 📃 Detailed Scenario

Jooli, Inc. is a forward-thinking technology company that continuously seeks to innovate and streamline its operations. The company recently onboarded a new team, Griffin, to work on a  project involving the use of WordPress. The Griffin team has started setting up their environment but needed expert assistance to complete it efficiently and effectively.

There are several specific requirements and constraints provided by Jooli, Inc.:

* All resources have to be created in the us-east1 region and us-east1-c zone to maintain consistency and optimize network performance.
* The project was under strict budget monitoring, and resource allocation has to be cost-effective, with a preference for using e2-medium machine types unless otherwise directed.
* Standard naming conventions should be followed, typically using the format team-resource (e.g., griffin-webserver1).
* The infrastructure needs to include a development VPC with subnets, a production VPC with subnets, a bastion host connected to both VPCs, a Cloud SQL instance configured for WordPress, and a Kubernetes cluster to host the WordPress environment.
* Security and access management are crucial, requiring the creation of firewall rules and provision of access for an additional engineer.
  
This case study outlines the steps taken to meet these requirements and successfully set up the development environment for the Griffin team.

## 🎯 Solution

The end goal is for to set up an environment with resources that interact as shown below:

<p align = "center">
  <img src="https://cdn.qwiklabs.com/UE5MydlafU0QvN7zdaOLo%2BVxvETvmuPJh%2B9kZxQnOzE%3D" />

Before beginning the creation of resources on Google Cloud Platform, we will set up the environment variables. These variables ensure that the subsequent commands are executed in the correct region, zone, and project. We can also easily reference them in subsequent commands without repeatedly specifying their values. 

Run below commands to set the environment variables:

      REGION=us-east1
      
      ZONE=us-east1-c
      
      PROJECT=<project_id here>

### Task 1 - Create development VPC with 2 subnets

VPCs help to provide a structured way to deploy and isolate resources from each other. They also help to ensure controlled access and network segmentation using subnets. We will create two separate VPCs for team griffin's development and production environments. This isolation prevents potential issues in the development environment from affecting the production environment, thereby increasing overall system stability and security.

Subnets allow us to separate different types of resources within a VPC. For example, in the development VPC that we will create, we will create a griffin-dev-wp subnet for housing WordPress-related resources and a griffin-dev-mgmt for housing resources for management tasks. Subnets also help us control the flow of traffic between different parts of our resources.

First, we create the development VPC by running the below command:

      gcloud compute networks create griffin-dev-vpc --project=$PROJECT --subnet-mode=custom

To create the two subnets in the development VPC, we run the below commands:

        gcloud compute networks subnets create griffin-dev-wp --project=$PROJECT --range=192.168.16.0/20 --network=griffin-dev-vpc --region=$REGION

        gcloud compute networks subnets create griffin-dev-mgmt --project=$PROJECT --range=192.168.32.0/20 --network=griffin-dev-vpc --region=$REGION

### Task 2 - Create production VPC with 2 subnets
We will follow the same steps we did in **Task 1** to create the production environment:

First, we create the production VPC by running the below command:

      gcloud compute networks create griffin-prod-vpc --project=$PROJECT --subnet-mode=custom

To create the two subnets in the production VPC, we run the below commands:

      gcloud compute networks subnets create griffin-prod-wp --project=$PROJECT --range=192.168.48.0/20 --network=griffin-prod-vpc --region=$REGION
      
      gcloud compute networks subnets create griffin-prod-mgmt --project=$PROJECT --range=192.168.64.0/20 --network=griffin-prod-vpc --region=$REGION

### Task 3 - Create a bastion host
A bastion host is a server that acts as a security gateway to control access to a private network from an external network. 

In thisi case study, we want to create a bastion host to facilitate secure access to both the development and production environment. This host has two network interfaces, each connected to the management subnet of the respective VPCs (griffin-dev-mgmt and griffin-prod-mgmt). The bastion host serves as a gateway for administrators to access internal resources securely.

To do this, we use the Google Cloud Console and follow the below steps:
* Navigate to **Navigation menu > Compute Engine > VM instances**.
* Click **Create instance**.
* Fill in the instance name, machine type, region, zone, series, machine type
* From **Advanced options**, click **Networking, Disks, Security, Management, Sole-tenancy** dropdown.
* Click **Networking**.
* For **Network interfaces**, click the dropdown to edit.
* Add the griffin-dev-mgmt and griffin-prod-mgmt VPCs and click **Create**

Next, we create two firewall rules that will help manage the network traffic and ensure secure access to the resources.

griffin-firewall-rule1 will allow SSH access to instances within the development VPC from any IP address. This is essential for administrators or developers who need to remotely manage and access the virtual machines or other resources within the development environment.

      gcloud compute firewall-rules create griffin-firewall-rule1 --direction=INGRESS --priority=100 --network=griffin-dev-vpc --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0

griffin-firewall-rule2 will allow SSH access to instances within the production VPC from any IP address for the same reason as outlined above.

      gcloud compute firewall-rules create griffin-firewall-rule2 --direction=INGRESS --priority=100 --network=griffin-prod-vpc --action=ALLOW --rules=tcp:22 --source-ranges=0.0.0.0/0

**Key Considerations**
While these firewall rules are necessary for enabling SSH access, they also expose the SSH ports to the entire internet, which can be a security risk. To mitigate this risk, it is recommended to implement additional security measures such as:

* Using strong, unique passwords and key-based authentication for SSH.
* Restricting the source IP ranges to only those addresses that are known and trusted (e.g., the office IP address or VPN range).
* Regularly monitoring and logging SSH access to detect and respond to any unauthorized attempts.

### Task 4 - Create and configure Cloud SQL
We want to create a MySQL Cloud Instance called griffin-dev-db in the us-east1 region. This database will be configured to serve the WordPress environment. The setup includes creating a WordPress database, a dedicated user (wp_user), and granting appropriate privileges to ensure secure and efficient database operations.

To achieve this, we will use the interactive Google Console and take the following steps:
* Navigate to the **Cloud SQL** option
* Click **CREATE INSTANCE** > Choose **MySQL**
* Enter instance id as griffin-dev-db
* Enter a secure password in the **Password**
* Select the database version, Cloud SQL Edition, Preset
* Click **CREATE INSTANCE**

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
We want to create a 2 node zonal cluster of machine type e2-standard-4 called griffin-dev in the griffin-dev-wp and in zone us-east1-c. This cluster will host the WordPress environment. To achieve this, we run the below command:

      gcloud container clusters create griffin-dev --zone=us-east1-c --node-locations=us-east1-c --network=griffin-dev-vpc --subnetwork=griffin-dev-wp --num-nodes=2 --machine-type=e2-standard-4

### Task 6 - Prepare the Kubernetes cluster
First, we need to copy the necessary configuration files for the WordPress deployment from a Google Cloud Storage bucket to the Cloud Shell environment. This is achieved with the following command:

      gsutil cp -r gs://cloud-training/gsp321/wp-k8s .

The WordPress server needs to access the MySQL database using the username and password that was created in task 4. To do this, we need to update the configuration file with the correct username and password by following the below steps:

Open the wp-env.yaml file in a text editor and make the necessary changes:
       vi wp-k8s/wp-env.yaml

To securely connect to the Cloud SQL instance from Kubernetes, we need to create a key for the service account that will be used by the Cloud SQL Proxy. The below command will generate a JSON key file for the Cloud SQL Proxy service account, which is necessary for authentication when connecting to the Cloud SQL instance.

      gcloud iam service-accounts keys create key.json \
          --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com

Next, we need to store the service account key as a secret in Kubernetes. This ensures that the key can be securely accessed by the pods that need it:

      kubectl create secret generic cloudsql-instance-credentials \
          --from-file key.json

Finally, we need to apply the configuration file to the Kubernetes cluster. This ensures that the environment variables and secrets are correctly set up for the WordPress deployment:

      kubectl apply -f wp-k8s/wp-env.yaml

### Task 7 - Create a WordPress deployment
Now that we have provisioned the MySQL database, and set up the secrets and volume, we can create the deployment using wp-deployment.yaml.

First, we need to edit the wp-deployment.yaml file to include the instance connection name of the Cloud SQL instance. This ensures that the WordPress application can connect to the MySQL database. 

The below command opens the wp-deployment.yaml file in the vi editor, allowing you to configure the Cloud SQL instance connection name:

       vi wp-k8s/wp-deployment.yaml

The environment configuration file, wp-env.yaml, contains environment variables and other settings required by the WordPress application. We need to apply this configuration to the Kubernetes cluster:

      kubectl apply -f wp-env.yaml

Next, we need to apply the wp-deployment.yaml file to create the WordPress deployment in the Kubernetes cluster:

      kubectl apply -f wp-deployment.yaml

Finally, we need to apply the wp-service.yaml file to expose the WordPress application to external traffic:

      kubectl apply -f wp-service.yaml

### Task 8 - Enable monitoring

We want to create an uptime check for the WordPress development site. We can do this using the interactive Cloud Console:

* Navigate to **Monitoring**
* Click **Create an Uptime Check**
* Configure it by selecting: Check type - HTTP, Resource type - URL, Hostname - external IP address of the Load Balancer associated with the service, Path - /
* Click **Create**

### Task 9 - Provide access for an additional engineer

We want to grant another engineer editor role to the project where all the resources we have created so far reside in. We can do this using the interactive Cloud Console:

* Navigate to **IAM**
* Scroll through the list of users to find the engineer's username
* Click **Edit**, make the necessary changes, and then **Save**

## ✍️ Learnings and Reflections

* **Infrastructure as Code (IaC):** The project emphasized the importance of using IaC for managing cloud resources. By using tools like gcloud and kubectl, I gained practical experience in automating the deployment and management of infrastructure components.

* **Networking and Security:** Setting up VPCs, subnets, and firewall rules improved my understanding of network segmentation and security best practices. The project highlighted the need for secure access and proper configuration to protect cloud resources.

* **Database Management:** Configuring and connecting a Cloud SQL instance for the WordPress application provided insights into database management and secure connections between services in a cloud environment.

* **Kubernetes Orchestration:** Deploying and managing a Kubernetes cluster, along with setting up secrets and environment configurations, reinforced my skills in container orchestration and management. This experience is crucial for deploying scalable and resilient applications.

* **Monitoring and Observability:** Enabling monitoring for the Kubernetes cluster highlighted the importance of observability in cloud environments. Understanding how to monitor and troubleshoot applications is essential for maintaining uptime and performance.

Overall, this project has prepared me for future projects/roles by enhancing my technical skills and improving my understanding of cloud architecture.
