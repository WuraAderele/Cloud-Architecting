# 💻 Configuring a Scalable Nginx Web Server Infrastructure

## 📑 Introduction

In this case study, I implemented a scalable Nginx web server infrastructure for Jooli, Inc. 

The primary goal was to configure an environment that serves a website via Nginx web servers while ensuring fault tolerance and high availability. To achieve this, I created an HTTP load balancer with a managed instance group of two Nginx web servers in Google Cloud Platform (GCP).

This project involved using various skills and technologies, including:

* Google Cloud Platform (GCP): For provisioning and managing cloud resources.
* Nginx: As the web server to serve the website.
* Load Balancing: To distribute incoming traffic across multiple web servers.
* Shell Scripting: For automating the configuration and setup of the web servers.
* The case study demonstrates the ability to design, implement, and manage scalable cloud infrastructure, ensuring reliability and efficient load distribution.

## 📃 Detailed Scenario

Jooli, Inc. is a rapidly growing technology company specializing in providing innovative web solutions to a diverse clientele. 

As the company scales its operations, it faces the challenge of ensuring that its web services remain highly available and resilient to failures. 

The company’s current infrastructure was not designed to handle the increasing load and required improvements to meet the demands of its growing user base.

**♟️ Motivation Behind the Project**

The primary motivation behind this project was to enhance the company’s web server infrastructure by implementing a fault-tolerant and scalable solution. The key objectives were:

* High Availability: Ensure that the website remains accessible even if one of the web servers fails.
* Load Distribution: Distribute incoming traffic evenly across multiple web servers to prevent any single server from being overwhelmed.
* Scalability: Easily scale the number of web servers based on demand.

**🧩 Specific Requirements and Constraints**

To address these objectives, Jooli, Inc. outlined the following requirements and constraints for the project:

* **Region and Zone:** All resources must be hosted in the US-West4 region and US-West4b zone to optimize latency and comply with data residency requirements.
* **Nginx Web Servers:** Use Nginx as the web server to serve the company’s website.
* **Managed Instance Group:** Create a managed instance group consisting of two Nginx web servers to ensure redundancy and load distribution.
* **Automated Configuration:** Utilize the below startup script to automate the installation and configuration of Nginx on the web servers.
  
          cat << EOF > startup.sh
        #! /bin/bash
        apt-get update
        apt-get install -y nginx
        service nginx start
        sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
        EOF
  
* **Health Checks:** Implement health checks to monitor the status of the web servers and ensure they are responding correctly to traffic.
* **Firewall Rules:** Configure firewall rules to allow HTTP traffic on port 80.
* **Load Balancer:** Set up an external HTTP load balancer to route incoming traffic to the web servers.
By adhering to these requirements, the project aims to provide a robust infrastructure solution that meets Jooli, Inc.'s operational needs and supports its growth trajectory.

## 🎯 Solution
**How an external HTTP(S) Load Balancer works**

An external HTTP(S) Load Balancer distributes incoming HTTP(S) traffic across multiple backend instances to ensure high availability and reliability of web applications. Here is a detailed explanation of how it operates:

* **Client Request:** A client sends an HTTP(S) request to the IP address and port specified in the load balancer's forwarding rule.
* **Forwarding Rule:** The forwarding rule defines the front-end configuration, including the IP address, protocol, and port, that clients will use to connect to the load balancer. It routes the client request to a specified target proxy.
* **Target Proxy:** The forwarding rule directs the traffic to a target proxy. The target proxy is responsible for managing the incoming requests. It receives the client request and compares the request's destination IP address & port to what is configured in the forwarding rule. If a match is found, the target proxy terminates the client's network connection and establishes a new one  to the appropriate backend as determined by the Load Balancer's URL Map.
* **URL Map:** The target proxy uses a URL map to determine how to route requests. The URL map matches the request's URL to backend services or buckets.
* **Backend Service:** The URL map forwards the request to the appropriate backend service. The backend service manages backend instances, performing health checks, and distributing traffic.
* **Instance Group:** The backend service routes the request to instances within an instance group based on the load balancing algorithm. The instance group contains the web servers that serve the actual content.
* **Target Pool:** Target pools are used by the load balancer to manage the backend instances. When a forwarding rule directs traffic to a target pool, the load balancer chooses an instance from the target pool based on a hash of the source and destination IP and port.
* **Health Checks:** Health checks are performed to ensure that instances are available and healthy. If an instance fails a health check, it is removed from the pool of available servers until it recovers.

This entire setup ensures that traffic is evenly distributed across multiple servers, providing redundancy and improving the application's overall reliability. 

**📷 Infrastructure Diagram**


**☁️ Creating Resources in GCP and Setting up Infrastructure**

To get started:
* Log into Google Cloud Console in a web browser
* Activate Cloud Shell by clicking the "Activate Cloud Shell button" at the top right of the console. This opens a terminal window at the bottom of the console.

Jooli, Inc. hosts all their resources in US-West-4 Region and US-West-4b so we run the below command to set the default zone and region in Cloud Shell:

      gcloud config set compute/region us-west4
      
      gcloud config set compute/zone us-west4-b

Next, create a startup script that will be used to configure the Nginx web servers. Run the following command to save the configuration code to a startup.sh file:

      cat << EOF > startup.sh
      #! /bin/bash
      apt-get update
      apt-get install -y nginx
      service nginx start
      sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
      EOF

A gcloud instance template saves a VM's configuration and makes it easy to create a group of instances  with the same identical properties. 

Our goal is to have 2 web servers that the load balancer will distribute traffic between. Instead of creating them one after the other, we can use an instance template to create both VMs at once.

Run below command and parameters to create an instance template. Specify the startup script in the metadata parameter so that the script runs during the startup process of created VMs and installs nginx.

      gcloud compute instance-templates create nginx-template \
      --metadata-from-file startup-script=startup.sh

Create the target pool to group instances that will receive incoming traffic from the load balancer:

      gcloud compute target-pools create nginx-pool

Instance groups help with running scalable and highly available workloads.

Run below command to create an instance group of at least 2 VMs that will host the web server. These VMs are based on the nginx-template created earlier.

      gcloud compute instance-groups managed create nginx-group \
      --base-instance-name nginx-instance \
      --size 2 \
      --template nginx-template \
      --target-pool nginx-pool

Run below command to specify the ports to be used by the instances in the instance group:

      gcloud compute instance-groups managed set-named-ports nginx-group --named-ports http:80

Run below command to create a firewall rule that allows ingress traffic from TCP port 80:
      
      gcloud compute firewall-rules create permit-tcp-rule-685 \
      --allow tcp:80

Run below command to create a health check resource that will help determine whether backend instances are responding properly to traffic:

      gcloud compute http-health-checks create basic-check

Run below command to create a backend service that will handle all client requests and specify the created health check resource:

      gcloud compute backend-services create nginx-backend \
      --protocol HTTP  \
      --port-name http \
      --http-health-checks basic-check 
      --global

Run below command to add the nginx-group instances as the backend service:

      gcloud compute backend-services add-backend nginx-backend \
      --instance-group nginx-group \
      --instance-group-zone us-west4-b\
      --global

Run below command to create a URL map to route the incoming requests to the created backend service:
      
      gcloud compute url-maps create web-map \
      --default-service nginx-backend

Run below command to create a target HTTP proxy to route requests to the created URL map:

      gcloud compute target-http-proxies create web-proxy \
      --url-map web-map

Run the below command to create a global forwarding rule to route incoming requests to the created proxy:

      gcloud compute forwarding-rules create web-forwarding-rule \
      --global \
      --target-http-proxy web-proxy \
      --ports=80

Each of these steps ensures that the infrastructure is set up correctly, providing a fault-tolerant and scalable Nginx web server environment. You can verify the setup by accessing the load balancer's IP address in your browser to see if the Nginx welcome page is displayed, indicating successful configuration.

### ✍️ Conclusion
This project has prepared me for managing cloud infrastructure projects, either as a Technical Program Manager or a Cloud Architect, by providing practical experience with essential cloud services and infrastructure design principles. 

It has enhanced my problem-solving skills and ability to design scalable, reliable, and secure cloud-based solutions.











