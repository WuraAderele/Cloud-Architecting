# Infrastructure Automation with Terraform

## Scenario
You are a cloud engineer intern for a new startup. For your first project, your new boss has tasked you with creating infrastructure in a quick and efficient manner and generating a mechanism to keep track of it for future reference and changes. You have been directed to use **Terraform** to complete the project.

## Solution

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
There are already two existing instances named tf-instance-1 and tf-instance-2. We can find its Instance ID, boot disk image, and machine type by clicking on each of them in the Googlr Cloud Console. These information are necessary for writing the necessary configurations in order to import them into Terraform.

We want to import these instances into an instances module. To do this, first, we add the module reference into the main.tf file and then re-initialize Terraform.

      module "instances" {
          source = "./modules/instances"
      }

      terraform init
      
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

Next, use the terraform import command to import thr two instances into our instances module. The format for the command is:

        terraform import module.instances.google_compute_instance.tf-instance-1 project_id/zone/instance_id

Run below commands:

        terraform import module.instances.google_compute_instance.tf-instance-1 qwiklabs-gcp-01-4827ad987c04/us-east4-c/4654656598094574949

        terraform import module.instances.google_compute_instance.tf-instance-1 qwiklabs-gcp-01-4827ad987c04/us-east4-c/1344797105777735013


Apply the changes to import the resources:

        terraform apply

### Task 3 - Configure a remote backend
Next, we want to create a Cloud Storage bucket resource inside the storage module. First, we will specify the resource configuration of the bucket in the storage.tf file.

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

      Initialize terraform:

              terraform init

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
