# Description

I've created an Ansible playbook for ASG with dynamic inventory to update Git on active instances. Use it manually or via Jenkins, no extra instances will be created.

# Features

ASG Rolling updates are made with ansible-playbook (Primary)
We do not need hosts (Inventory file) for ASG under client servers as this work with Dynamic Inventory setup.
The values of the project variables can be changed from var files as needed.

# Prerequisites

Knoweldge in AWS infrastructure.
Creating a dedicated directory for the ansible project.
Installing Ansible, Jenkins and Git tools on the Master Machine.
Installing pip, boto, boto3 and botocore modules.
Creating an IAM user role under your AWS account.
