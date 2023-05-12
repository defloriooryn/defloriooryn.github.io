---
title: "Copy Container Image Between ECR Private Repository Within Separate Account"
date: 2023-05-12T18:51:08+07:00
draft: false
toc: false
images:
---

![Logo ECR](https://github.com/defloriooryn/Assets/blob/main/ecr.png?raw=true#center)

### *Prerequisite* : 
1. Two AWS accounts
2. ECR repositories in each account
3. Version 2.11.3 or later of the AWS CLI installed 

&nbsp;

### First | Create Access & Secret Key
1. Open AWS Console at https://console.aws.amazon.com/. Then click on your profile name and then click on **Security Credentials**. (you can skip to the next second-step, if you're already have access and secret key).
![Logo 1](https://github.com/defloriooryn/Assets/blob/main/1.jpeg?raw=true)

2. Scroll down to **Access Keys** and select **Create New Access Key**.
![Logo 1](https://github.com/defloriooryn/Assets/blob/main/2.jpeg?raw=true)

3. Click on Show **Secret Key** and save the Access key and Secret key. 
![Logo 1](https://github.com/defloriooryn/Assets/blob/main/3.jpeg?raw=true)

4. Repeat the steps 1-3 for the other account.

Fyi, you can have maximum of two Access & Secret key and you can create Access & Secret Key using AWS CLI use following this command.
  > aws iam create-access-key

&nbsp;

### Second | Configure the AWS CLI
Run the command below, then fill in the Access & Key according to previous step and fill Region and Output contents according to what you need. (- -profile can use any name)
> aws configure --profile *source-account*

> aws configure --profile *destination-account*

&nbsp;

### Third | Pull the container image from the source accounts
1. Open terminal and you can switch profile to Source Account using command below.
   > export AWS_PROFILE=*source-account*
2. You can login to the registry source account after replacing *region-code* with the AWS Region that you created your Amazon ECR private repository in, and replace the *123456789* with Source Account ID.
   > aws ecr get-login-password - -region ***region-code*** | docker login  - -username AWS - -password-stdin ***123456789*** .dkr.ecr.region-code.amazonaws.com
3. Pull the image and replace ***123456789*** and ***region-code*** with the information that you provided in the previous step.
   > docker pull ***123456789***.dkr.ecr.***region-code***.amazonaws.com/name-of-image:***v1.0*** s

&nbsp;

### Fourth | Tag the container image that you pulled with your destination registry
Replace the *987654321* with Source Account ID, and *region-code* with the AWS Region that you created your Amazon ECR private repository in.
> docker tag name-of-image:v1.0 ***987654321***.dkr.ecr.***region-code***.amazonaws.com/name-of-image:v1.0

&nbsp;

### Fifth | Push the container image to the destination account
1. Switch profile to Source Account
   > export AWS_PROFILE=*destination-account*
2. Login to the registry destination account.
   > aws ecr get-login-password --region ***region-code*** | docker login --username AWS --password-stdin ***987654321***.dkr.ecr.region-code.amazonaws.com
3. Push the image to the destination account.
   > docker push ***987654321***.dkr.ecr.***region-code***.amazonaws.com/name-of-image:v1.0

&nbsp;

### Additional

Just like that for notes today, hopefully I can still exist to update this website
