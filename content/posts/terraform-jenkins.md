---
title: "Automate IaC with Terraform and Jenkins on AWS"
date: 2023-10-04T18:51:08+07:00
draft: false
toc: false
ShowShareButtons: false
tags: ["AWS", "Terraform", "Jenkins"]
---

![Diagram](/jenkins-terraform.png#center)

### Objective
- Setup Jenkins for automate IaC 
- Provision highly available 3-tier architecture with Terraform

&nbsp;


### *Pre-requisite* :
- AWS & Github Account
- Configured AWS Credentials on local 
- AWS CLI Installed

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

### Third | Install & Configure AWS Credentials Plugin

Go to Manage Jenkins > Plugin > Availabe Plugin > search AWS Credentials, and click Install.
![1](/1-terraform-jenkins.png)

Next configure AWS Credential, go to Manage Jenkins > Credentials > System > Global credentials > and click Add Credentials.
![2](/2-terraform-jenkins.png)

&nbsp;

### Fourth | Setup Jenkins Pipeline

Click New Item > Enter Pipeline Name
![3](/3_terraform-jenkins.png)

Enter you pipeline script : 
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
                WithCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "nizamzam",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                sh 'terraform apply -auto-approve'
                }
            }
        }
    }
}
```
&nbsp;

### Fifth | Deploy 3-Tier Architecture 
![4](/4-terraform-jenkins.png#center)

Below is the terraform code for the 3 tier architecture topology above or you can find the complete code in my GitHub account.

version.tf
```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

backend.tf
```bash
terraform {
  backend "s3" {
    region  = "us-east-1"
    bucket  = "nizamzam"
    key     = "terraform.tfstate"
    encrypt = true
  }
}
```

networking.tf
```bash
resource "aws_vpc" "vpc_example" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc_example.id
}

resource "aws_subnet" "public" {
  for_each                = var.subnet_public
  vpc_id                  = aws_vpc.vpc_example.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true
}

resource "aws_subnet" "private" {
  for_each          = var.subnet_private
  vpc_id            = aws_vpc.vpc_example.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
}

resource "aws_subnet" "database" {
  for_each          = var.subnet_database
  vpc_id            = aws_vpc.vpc_example.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
}

resource "aws_eip" "eip_natgw" {
  domain = "vpc"

  # lifecycle {
  #   prevent_destroy = true
  # }
}

resource "aws_nat_gateway" "nat_gw" {
  allocation_id = aws_eip.eip_natgw.id
  subnet_id     = values(aws_subnet.public)[0].id
}

resource "aws_route_table" "route_public" {
  vpc_id = aws_vpc.vpc_example.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table" "route_private" {
  vpc_id = aws_vpc.vpc_example.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat_gw.id
  }
}

resource "aws_route_table_association" "public" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.route_public.id
}

resource "aws_route_table_association" "private" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.route_private.id

}
```

resources.tf
```bash
# Security Group Application Load Balancer
resource "aws_security_group" "sg-alb" {
  name        = "sg_alb"
  vpc_id      = aws_vpc.vpc_example.id
  description = "allow http and https"
  dynamic "ingress" {
    for_each = var.sg_alb_port
    iterator = port
    content {
      from_port   = port.value
      to_port     = port.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Security Group Instance
resource "aws_security_group" "sg-instance" {
  name        = "sg_ec2"
  vpc_id      = aws_vpc.vpc_example.id
  description = "allow http from sg-alb"
  dynamic "ingress" {
    for_each = var.sg_ec2_port
    iterator = port
    content {
      from_port       = port.value
      to_port         = port.value
      protocol        = "tcp"
      security_groups = [aws_security_group.sg-alb.id]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

#Security Group RDS
resource "aws_security_group" "sg-rds" {
  name        = "sg_rds"
  vpc_id      = aws_vpc.vpc_example.id
  description = "allow port 3306 from sg-instance"

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.sg-instance.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Auto Scalling Group
resource "aws_launch_template" "launch_template" {
  name          = "template"
  image_id      = var.instance_template.ami
  instance_type = var.instance_template.type

  network_interfaces {
    device_index    = 0
    security_groups = [aws_security_group.sg-instance.id]
  }
}

resource "aws_autoscaling_group" "asg" {
  desired_capacity    = var.asg.desired
  min_size            = var.asg.min
  max_size            = var.asg.max
  target_group_arns   = [aws_lb_target_group.tg_instance.arn]
  vpc_zone_identifier = values(aws_subnet.private)[*].id

  launch_template {
    id      = aws_launch_template.launch_template.id
    version = "$Latest"
  }
}


# Application Load Balancer
resource "aws_lb" "alb" {
  name               = "external-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.sg-alb.id]

  dynamic "subnet_mapping" {
    for_each = aws_subnet.public
    content {
      subnet_id = subnet_mapping.value.id
    }
  }

}
resource "aws_lb_target_group" "tg_instance" {
  name     = "nginx"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.vpc_example.id

  health_check {
    path    = "/"
    matcher = 200
  }
}

resource "aws_lb_listener" "lb_listener_http" {
  load_balancer_arn = aws_lb.alb.arn
  port              = "80"
  protocol          = "HTTP"
  default_action {
    target_group_arn = aws_lb_target_group.tg_instance.id
    type             = "forward"
  }
}


# Database
resource "aws_db_subnet_group" "sg_db" {
  name       = "db-subnet"
  subnet_ids = values(aws_subnet.database)[*].id
}

resource "aws_db_instance" "db" {
  allocated_storage    = var.db.allocated_storage
  db_name              = var.db.db_name
  db_subnet_group_name = aws_db_subnet_group.sg_db.name
  engine               = var.db.engine
  engine_version       = var.db.engine_version
  instance_class       = var.db.instance_class
  username             = var.db.username
  password             = var.db.password
  parameter_group_name = var.db.parameter_group_name
  skip_final_snapshot  = var.db.skip_final_snapshot
  availability_zone    = var.db.availability_zone
  multi_az             = var.db.multi_az
}
```

output.tf
```bash
output "alb_dns" {
  value = aws_lb.alb.dns_name
}
```

variables.tf
```bash
variable "region" {
  type    = string
  default = "us-east-1"
}

# VPC & Subnet 
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_public" {
  type = any
  default = {
    "public-1" = { cidr = "10.0.0.0/24", az = "us-east-1a" }
    "public-2" = { cidr = "10.0.1.0/24", az = "us-east-1b" }
  }
}

variable "subnet_private" {
  type = map(object({
    cidr = string
    az   = string
  }))
  default = {
    "private-1" = { cidr = "10.0.2.0/24", az = "us-east-1a" }
    "private-2" = { cidr = "10.0.3.0/24", az = "us-east-1b" }
  }
}

variable "subnet_database" {
  type = map(object({
    cidr = string
    az   = string
  }))
  default = {
    "private-3" = { cidr = "10.0.4.0/24", az = "us-east-1a" }
    "private-4" = { cidr = "10.0.5.0/24", az = "us-east-1b" }
  }
}

# Security Group
variable "sg_alb_port" {
  type    = list(any)
  default = [80, 443]
}

variable "sg_ec2_port" {
  type    = list(any)
  default = [80, 22]
}

variable "sg_rds_port" {
  type    = number
  default = 3306
}

# Auto Scalling Group
variable "instance_template" {
  type = map(string)
  default = {
    type = "t2.micro"
    ami  = "ami-067d1e60475437da2" #Amazon Linux 2023
  }
}

# ASG
variable "asg" {
  type = any
  default = {
    az      = ["us-east-1a", "us-east-1b"]
    desired = 2
    min     = 2
    max     = 4
  }
}

# Database
variable "db" {
  type = map(string)
  default = {
    allocated_storage    = 10
    db_name              = "mydb"
    engine               = "mysql"
    engine_version       = "5.7"
    instance_class       = "db.t3.micro"
    username             = "admin"
    password             = "admin"
    parameter_group_name = "default.mysql5.7"
    skip_final_snapshot  = true
    availability_zone    = "us-east-1a"
    multi_az             = false
  }
}
```