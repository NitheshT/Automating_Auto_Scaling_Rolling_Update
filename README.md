# Description

I've created an Ansible playbook for ASG with dynamic inventory to update Git on active instances. Use it manually or via Jenkins, no extra instances will be created.

# Features

- ASG Rolling updates are made with ansible-playbook (Primary)
- We do not need hosts (Inventory file) for ASG under client servers as this work with Dynamic Inventory setup.
- The values of the project variables can be changed from var files as needed.

# Prerequisites

- Knoweldge in AWS infrastructure.
- Creating a dedicated directory for the ansible project.
- Installing Ansible, Jenkins and Git tools on the Master Machine.
- Installing pip, boto, boto3 and botocore modules.
- Creating an IAM user role under your AWS account.

# Architacture

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/6047334c-400f-4d4d-93cf-eb7d699d197e)

# Ansible

```
---
- name: "Bulding Dynamic Inventory"
  hosts: localhost
  vars_files:
    - key.vars
  tasks:

    - name: "Fetching EC2 instance details from ASG"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "tag:aws:autoscaling:groupName": "{{ asg_tag }}" 
          instance-state-name: [ "running"]
      register: ec2

    - name: "Print instance details"
      debug:
        var: ec2

    - name: "Creating Dymaic Inventory"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "instances"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "{{ user }}"
        ansible_ssh_private_key_file: "{{ key }}"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}" 
  
    - name: "Print ec2 IP's"
      debug:
        msg: "public ip's -> {{ item.public_ip_address }}"
      with_items:
        - "{{ ec2.instances }}"

- name: "Deloying website from Github accoung"
  hosts: instances                       #Dynmic_Inventory
  become: true
  serial: 1       
  vars_files:
    - app.vars
  tasks:
    
    - name: "Installing required pacages for application"
      yum:
        name:
          - httpd
          - php
          - git

    - name: "Deploymnet of the application from the Git repository"
      git: 
        repo: "{{ git_url }}" 
        dest: "{{ clone_dir }}"
      register: git_repo_status

    - name: "Disabling ELB health check"
      when: git_repo_status.changed == true
      file:
        path: /var/www/html/{{ health_page }}
        state: touch
        mode: 0000

    - name: "Loading Ec2 instance from ELB"
      when: git_repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"
  
    - name: "Copying New Content To App Root"
      when: git_repo_status.changed == true
      copy:
        src: "{{ clone_dir }}"
        dest: /var/www/html/
        remote_src: true

    - name: "ReStarting/enabling Application"
      when: git_repo_status.changed == true
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Re-enabling ELB health check"
      when: git_repo_status.changed == true
      file:
        path: "/var/www/html/{{ health_page }}"
        state: touch
        mode: 0644

    - name: "Loading Ec2 instance to ELB"
      when: git_repo_status.changed == true
      pause:
        seconds: "{{ health_time }}"  
```

# Variables

- key.vars

```
git_url: "https://git_repo_url.git"                       #Repository_URL
clone_dir: "/var/temp"                       #Clone_Directoy
health_page: health.html                       #Health_Page
health_time: 25                       #Health_Time
```

- app.vars

```
access_key: "<access-key>"                       #IAM_Access_Key
secret_key: "<secret-key>"                       #IAM_Secret_Key
region: "us-east-1"
key: "*****"                       #key_Pairname 
user: "*****"                       #User
asg_tag: "*****"                       #ASG_Name
```

# Setting up JENKINS

```
yum update –y
wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum upgrade
amazon-linux-extras install java-openjdk11 -y
dnf install java-11-amazon-corretto -y
yum install jenkins -y
systemctl enable jenkins
sudo systemctl start jenkins
```

# Inital Setup

http://< serverIP >:8080

# Find temoprary password to access the portal from the below mentioned path:

cat /var/lib/jenkins/secrets/initialAdminPassword

# Installing Ansible On The Jenkin Server

```
amazon-linux-extras install ansible2 -y
yum install git -y
```

# Configuring Jenkins

- Manage Jenkins >> Manage Plugins >> Ansible Plugin (searching) >> Install Ansible plugin in Jenkins >> Select restart jenkins when installation complete:

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/7034f628-4728-4528-97c9-f94843ea5849)

- Manage Jenkins >> Global Tool Configuration >> Navigate to Ansible section >> Add Ansible (Add the name and Ansible instalaltion path):

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/52cacfea-3ae6-49c8-81c7-c00cd94dc085)

- Providing a project name and selecting Freestyle Project:

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/98d74414-80ab-4077-8763-48d30716240e)

- Select the Source code management >> Adding Repository URL:

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/80770cb2-3f2c-497e-88c8-a291176aac01)

- Build Steps >> Add Build Step >> Invoke Ansible Playbook

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/29b990f5-2f53-4adf-8eb5-af706c31f80f)

- Add Ansible Playbook path:

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/3f811ca6-49a2-484a-9195-ff8924f584d0)

- Build Trigger >> Check-in GitHub hook trigger for GITScm polling

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/fbd03529-3eaa-4318-82fe-218d2aab2cb6)

# Automation using Webhook Trigger from GitHub

- Navigate to GitHub, adding payload URL: http://< jenkins-server-IP >:8080/github-webhook/

![image](https://github.com/NitheshT/Automating_Auto_Scaling_Rolling_Update/assets/122042254/0e45db47-204e-41c2-ac78-947e2d5ff137)
