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


