# Configuring a Scalable Nginx Web Server Infrastructure

## Scenario
As a Junior Cloud Engineer for Jooli, Inc., the organization has put forward the below request:

The company wants to serve a website via nginx web servers, but also wants to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of 2 nginx web servers. Use the following code to configure the web servers; the team will replace this with their own configuration later:

    cat << EOF > startup.sh
    #! /bin/bash
    apt-get update
    apt-get install -y nginx
    service nginx start
    sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
    EOF

## Solution
Joolo, Inc.leverages Google Cloud Platform for their infrastructure needs. An optimal solution for these requirements would be to implement load balancing on GCP compute engine.

### Summary of how an external HTTP(S) Load Balancer works
A client makes a connection to the IP address and port of the Load Balancer's forwarding rule. A forwarding rule and its corresponding IP address represent the front-end configuration of a Load balancer. The forwarding rule routes the request to a specified target proxy. 
The target proxy receives the client request and compares the request's destination IP address & port to what is configured in the forwarding rule. If a match is found, the target proxy terminates the client's network connection and establishes a new one  to the appropriate backend as determined by the Load Balancer's URL Map.

### Creating Rescources in GCP and Setting up Infrastructure
To get started, log into Google Cloud Console and Activate Cloud Shell.

Jooli, Inc. hosts all their resources in US-West-4 Region and US-West-4b so we run the below command to set the default zone and region in Cloud Shell:

      gcloud config set compute/region us-west4
      
      gcloud config set compute/zone us-west4-b

Run below command in Cloud Shell to save the configuration code to a startup.sh file

      cat << EOF > startup.sh
      #! /bin/bash
      apt-get update
      apt-get install -y nginx
      service nginx start
      sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
      EOF

A gcloud instance template saves a VM's configuration and makes it easy to create a group of instances  with the same identical properties. Our goal is to have 2 web servers that the load balancer will distribute traffic between. Instead of creating them one after the other, we can use an instance template to create both VMs at once.
Run below command and parameters to create an instance template. Specify the startup script in the metadata parameter so that the script runs during the startup process of created VMs and installs nginx.

      gcloud compute instance-templates create nginx-template \
      --metadata-from-file startup-script=startup.sh

Run below command to create a target pool in the same region as the instances

      gcloud compute target-pools create nginx-pool

Instance groups help with running scalable and highly available workloads.
Run below command to create an instance group of at least 2 VMs that will host the web server. These VMs are based on the nginx-template created earlier

      gcloud compute instance-groups managed create nginx-group \
      --base-instance-name nginx-instance \
      --size 2 \
      --template nginx-template \
      --target-pool nginx-pool

Run below command to create a firewall rule that allows ingress traffic from TCP port 80
      
      gcloud compute firewall-rules create permit-tcp-rule-685 \
      --allow tcp:80

Run below command to create a health check resource that will help determine whether backend instances are responding properly to traffic

      gcloud compute http-health-checks create basic-check

Run below command to create a backend service that will handle all client requests and specify the created health check resource

      gcloud compute backend-services create nginx-backend \
      --protocol HTTP  \
      --port-name http \
      --http-health-checks basic-check 
      --global

Run below command to add the nginx-group instances as the backend service

      gcloud compute backend-services add-backend nginx-backend \
      --instance-group nginx-group \
      --instance-group-zone us-west4-b\
      --global

Run below command to create a URL map to route the incoming requests to the created backend service
      
      gcloud compute url-maps create web-map \
      --default-service nginx-backend

Run below command to create a target HTTP proxy to route requests to the created URL map

      gcloud compute target-http-proxies create web-proxy \
      --url-map web-map

Run the below command to create a global forwarding rule to route incoming requests to the created proxy

      gcloud compute forwarding-rules create web-forwarding-rule \
      --global \
      --target-http-proxy web-proxy \
      --ports=80

Run below command to specify the ports to be used by the instances in the instance group

      gcloud compute instance-groups managed set-named-ports nginx-group --named-ports http:80
 
