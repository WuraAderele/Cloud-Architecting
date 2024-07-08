# 💻 Infrastructure Automation with Terraform

## 📑 Introduction
This project was undertaken as a hypothetical cloud engineer intern for a startup, with the goal of creating, deploying, and managing infrastructure efficiently using Terraform. The project involved setting up infrastructure on Google Cloud, importing existing resources, configuring remote backends, modifying infrastructure, and leveraging Terraform modules for reusable infrastructure code.

This project involved using various skills and technologies, including:

* **Google Cloud Platform (GCP):** Leveraged as the cloud service provider to host and manage infrastructure resources.
* **Terraform:** Utilized for creating, managing, and versioning infrastructure as code.
* **Shell Scripting:** Employed for automating repetitive tasks and facilitating infrastructure setup.
* **Terraform Modules:** Used for organizing and reusing infrastructure code efficiently.
* **Terraform Provider:** Specifically, the HashiCorp Google provider for interfacing with GCP services.
* **Firewall Configuration:** Ensured secure access and communication between infrastructure resources.
* **Networking:** Configured virtual private clouds (VPCs) and subnets to organize and secure resources.


## 📃 Scenario
For your first project, your new boss has tasked you with creating infrastructure in a quick and efficient manner and generating a mechanism to keep track of it for future reference and changes. You have been directed to use **Terraform** to complete the project.

You will use Terraform to create, deploy, and keep track of infrastructure on the startup's preferred provider, Google Cloud. You will also need to import some mismanaged instances into your configuration and fix them.

**♟️ Motivation Behind the Project**

The motivation behind this project was to establish an infrastructure management framework using Terraform in order to realise the following benefits of infrastructure as code (IaC):

* **Automation:** Reducing manual setup and configuration of cloud resources.
* **Consistency:** Ensuring all environments are configured identically, reducing errors caused by manual processes.
* **Version Control:** Keeping track of changes in infrastructure configuration over time.
* **Collaboration:** Enabling multiple team members to work on infrastructure code simultaneously, with a reliable state management system.


## 🎯 Solution

### Task 1 - Creating configuration files
We will write and run Linux gcloud commands to create the needed Terraform configuration files and a directory structure that resembles the below:

                  main.tf
                  variables.tf
                  modules/
                  └── instances
                      ├── instances.tf
                      ├── outputs.tf
                      └── variables.tf
                  └── storage
                      ├── storage.tf
                      ├── outputs.tf
                      └── variables.tf

See commands to achieve this below:

      # create below files in root directory
      touch main.tf
      touch variables .tf

      # create the main directory and its two sub directories
      mkdir modules/instances
      mkdir modules/storage

      # navigate into each sub directory and create the required files
      cd modules/instances
      touch instances.tf
      touch outputs.tf
      touch variables.tf

      cd ~      # navigate out of the instances sub directory
      cd modules/storage
      touch storage.tf
      touch outputs.tf
      touch variables.tf

Next, we will fill out all the variables.tf files with required variables that will be used during resource configurations.

      variable "region" {
          default = "us-east4"
      }
      
      variable "zone" {
          default = "us-east4-c"
      }
      
      variable "project_id" {
          default = "qwiklabs-gcp-01-4827ad987c04"
      }

Next, we add the Terraform block and Google Provider declaration to the main.tf file

      terraform {
      
          backend "gcs" {
              bucket = "tf-bucket-944459"
              prefix = "terraform/state"
          }
      
          required_providers {
              google = {
                  source = "hashicorp/google"
              }
          }
      }
      
      provider "google" {
          region = var.region
          zone = var.zone
          project = var.project_id
      }

Run below command to initializa terraform:

      terraform init

### Task 2 - Importing Infrastructure
There are already two existing instances named tf-instance-1 and tf-instance-2. We want to import these instances into Terraform so they can be managed by Terraform going. To do this, we follow the steps in Terraform import workflow:

* Identify the existing infrastructure to be imported - We can find the Network, Instance ID, boot disk image, and machine type of both instances by clicking on each of them in the Google Cloud Console.
* Write a Terraform configuration that matches that infrastructure.
* Import the infrastructure into your Terraform state.
* Review the Terraform plan to ensure that the configuration matches the expected state and infrastructure.
* Apply the configuration to update your Terraform state.

<img src="https://cdn.qwiklabs.com/feQ3c7%2Fby0X%2FS1RV9%2FMzQCVzbg30%2FYCHQairKWXjHF4%3D"/>

**Identify the existing infrastructure to be imported**

* Network - default
* Instance IDs - 4654656598094574949 (tf-instance-1), 1344797105777735013(tf-instance-2)
* Boot Disk Image - debian-11-bullseye-v20240611
* Machine Type - e2-micro

 first, we add the module reference into the main.tf file and then re-initialize Terraform.

      module "instances" {
          source = "./modules/instances"
      }

      terraform init
      
**Write a Terraform configuration that matches that infrastructure**      
Next, we write the resource configurations of both instances in the instances.tf file to match pre-existing instances.

        resource "google_compute_instance" "tf-instance-1"{
            name = "tf-instance-1"
            machine_type = "e2-micro"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

        
        resource "google_compute_instance" "tf-instance-2"{
            name = "tf-instance-2"
            machine_type = "e2-micro"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

**Import the infrastructure into your Terraform state**

Next, use the terraform import command to import the two instances into our instances module. The format for the command is:

        terraform import module.instances.google_compute_instance.tf-instance-1 project_id/zone/instance_id

Run below commands:

        terraform import module.instances.google_compute_instance.tf-instance-1 qwiklabs-gcp-01-4827ad987c04/us-east4-c/4654656598094574949

        terraform import module.instances.google_compute_instance.tf-instance-1 qwiklabs-gcp-01-4827ad987c04/us-east4-c/1344797105777735013

**Review the Terraform plan to ensure that the configuration matches the expected state and infrastructure**

The *terraform plan* command determines what actions are necessary to achieve the desired state specified in the configuration files. It provides a convenient way to check whether the execution plan for a set of changes matches your expectations without making any changes to real resources or to the state.

        terraform plan

**Apply the configuration to update your Terraform state**

Run the below command to apply the changes to import the resources:

        terraform apply

### Task 3 - Configure a remote backend
We want to create the Cloud Storage bucket resource inside the storage module. First, we will specify the resource configuration of the bucket in the storage.tf file. 

        resource "google_storage_bucket" "storage-bucket" {
            name = "tf-bucket-960831"
            location = "US"
            force_destroy = true
        }

Next we add the module reference to the main.tf file.

        module "storage" {
            source = "./modules/storage"
        }

Initialize terraform and apply the changes:

        terraform init
        terraform apply

We will configure this storage bucket as the remote backend inside the main.tf file:

      terraform {
      
          backend "gcs" {
              bucket = "tf-bucket-944459"
              prefix = "terraform/state"
          }
      
      
          required_providers {
              google = {
                  source = "hashicorp/google"
              }
          }
      }
      
      provider "google" {
          region = var.region
          zone = var.zone
          project = var.project_id
      }

Initialize terraform and apply the changes:

              terraform init -migrate-state

Upon running the above command, Terraform will ask whether we want to copy the existing state data to the new backend. We type yes at the prompt.

### Task 4 - Modify and update infrastructure
We want to change the machine type of the existing two instances and create an additional one.

        resource "google_compute_instance" "tf-instance-1"{
            name = "tf-instance-1"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }
        
        resource "google_compute_instance" "tf-instance-2"{
            name = "tf-instance-2"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

        resource "google_compute_instance" "tf-instance-3"{
            name = "tf-instance-3"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

Initialize terraform and apply the changes:

        terraform init
        
        terraform apply

### Task 5 - Destroy resources
To destroy tf-instance-3, we simply delete the below block of code from the instances.tf file.

        resource "google_compute_instance" "tf-instance-3"{
            name = "tf-instance-3"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "default"
                access_config {
                
                }
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

### Task 6 - Use a module from the Registry
First, we add a network module to the main.tf 

      module "vpc" {
        source  = "terraform-google-modules/network/google"
      
        project_id = var.project_id
        network_name = "tf-vpc-548028"
        routing_mode = "GLOBAL"
      
        subnets = [
          {
              subnet_name = "subnet-01"
              subnet_ip = "10.10.10.0/24"
              subnet_region = var.region
          },
      
          {
              subnet_name = "subnet-02"
              subnet_ip = "10.10.20.0/24"
              subnet_region = var.region
          }
        ]
      }

Initialize terraform and apply the changes to create the networks

        terraform init
        
        terraform apply

Next, we update the configuration of the resources in the instances.tf file to connect to the created subnets.


        resource "google_compute_instance" "tf-instance-1"{
            name = "tf-instance-1"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "tf-vpc-548028"
                subnetwork = "subnet-01"
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }
        
        resource "google_compute_instance" "tf-instance-2"{
            name = "tf-instance-2"
            machine_type = "e2-standard-2"
        
            boot_disk {
                initialize_params {
                    image = "debian-11-bullseye-v20240611"
                }
            }
        
            network_interface {
                network = "tf-vpc-548028"
                subnetwork = "subnet-02"
            }
        
        metadata_startup_script = <<-EOT
        #!/bin/bash
        EOT
        
        allow_stopping_for_update = true
        }

### Task 7 - Configure a firewall
We create a firewall rule resource in the main.tf file.

        resource "google_compute_firewall" "tf-firewall" {
          name    = "tf-firewall"
          network = "projects/qwiklabs-gcp-04-78636d1c655b/global/networks/tf-vpc-548028"
          allow {
            protocol = "tcp"
            ports = [80]
          }
        
          source_tags = ["web"]
          source_ranges = ["0.0.0.0/0"]
        }

Initialize terraform and apply the changes to create the networks

        terraform init
        
        terraform apply  
