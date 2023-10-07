---
title: "Automate IaC with Terraform and Jenkins on AWS [On-going]"
date: 2023-10-04T18:51:08+07:00
draft: false
toc: false
ShowShareButtons: false
tags: ["AWS", "Terraform", "Jenkins"]
---

![Diagram](/jenkins-terraform.png#center)

### *Pre-requisite* :
1. AWS & Github Account
2. Configured AWS Credentials in local
3. AWS CLI Installed

&nbsp;

### First | Create and Setup Jenkins Server
Run EC2 Instance for Jenkins Server and fill the parameter what do you want.
```bash
aws ec2 run-instances \
--image-id ami-abcd1234 \
--instance-type t2.medium \
--key-name keypair-name \
--subnet-id subnet-abcd1234 \
--security-group-ids sg-abcd1234 \
--user-data file://setup.sh
```

This is a content of *setup.sh* :
```bash
#!/bin/bash

# Ubuntu/Debian

# Install Git & Java
sudo apt update && sudo apt install git fontconfig openjdk-17-jre -y

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update && sudo apt-get install jenkins -y
sudo systemctl enable jenkins

# Install Terraform
sudo apt install curl gnupg software-properties-common -y
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" sudo tee /etc/apt/sources.list.d/hashicorp.list 
sudo apt update && sudo apt install terraform -y
```
After instance running and Jenkins successfuly installed, ensure you have open port 8080 on *Security Group* because its Jenkins default port. And now setting up Jenkins with access **your-instance-ip-or-domain:8080** via browser. 
![Unlock-Jenkins](/Unlock-jenkins.png)  

To unlock Jenkins, open terminal and use command below to see the password.
```bash
> ssh -i keypair-name user@domain/public-ip-instance 'sudo cat /var/lib/jenkins/secrets/initialAdminPassword' 

0587d04623434423a1532785eaf9692a
```

&nbsp;

### Second | Generate AWS Credentials
You can skip this step, if you already have Access & Secret Key
```bash
> aws iam create-access-key --user-name your-user

{
    "AccessKey": {
        "UserName": "your-user",
        "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
        "Status": "Active",
        "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
        "CreateDate": "2023-10-054T17:34:16Z"
    }
}
```

&nbsp;

### Third | Install and Configure AWS Credential & Terraform Plugin               

&nbsp;

### Fourth | Setup Jenkins Pipeline
```bash
#Jenkinsfile
pipeline {
    agent any
    stages {
        
        stage("Checkout"){
            steps{
                git 'https://github.com/defloriooryn/jenkins-terraform.git'
            }
        }

        stage("TF Init"){
            steps{
                sh terraform init
            }
        }

        stage("TF Validate"){
            steps{
                sh terraform validate
            }
        }

        stage("TF Plan"){
            steps{
                sh terraform plan
            }
        }

        stage("TF Apply"){
            steps{
                sh export AWS_PROFILE=default
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "dev",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                sh 'terraform apply -auto-approve'
            }
        }
}
```
<!--  
### Fifth | 
### Sixth | 
### Seventh | 
### Eighth | 
### Ninth | 
### Tenth |  -->
