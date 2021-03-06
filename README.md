# christmas-project-2017

## Overview

An educational project:
 
Purpose: 
* Provision 5 Centos 7 VM's on AWS using Terrarform
* Set these up as a kubernetes cluster using Ansible
* Deploy and test app accross multiple nodes 
* Create a Kubernetes Service to front the test app

## How to use

* Install git and terraform on your local workstation
* Clone this repo
* Create an AWS ssh keypair (or choose an exiting one to use)
* Change the value of private_key_path in variables.tf to point at your private ssh key
* Create a terraform.tfvars file with the following contents
```
  secret_key = "your_aws_secret_key"
  access_key = "your_aws_access_key"
  key_name = "your_aws_ssh_key_name"  
```
* Run terraform init to download provider plugins
* Run terraform plan to check your config
* If all is well run terraform apply
* This will provision a VPC with the neccesary internet gateway, subnets, security groups etc, plus the following servers:
  * Ansible server
    * IP address: 10.0.1.100  
    * hostname : 10-0-1-101
    * hostname alias : ansible
  * Kubernetes master 
    * IP address: 10.0.1.101  
    * hostname : 10-0-1-101
    * hostname alias : kube_controller
  * Kubernetes minion 
    * IP address: 10.0.1.111  
    * hostname : 10-0-1-111
    * hostname alias : kube_minion_1
  * Kubernetes minion 
    * IP address: 10.0.1.112  
    * hostname : 10-0-1-112
    * hostname alias : kube_minion_2
  * Kubernetes minion 
    * IP address: 10.0.1.113  
    * hostname : 10-0-1-113
    * hostname alias : kube_minion_3

* When complete, log on to the Ansible server, su to ansible (password is ansible) and clone this repository
* cd to the ```ansible``` directory and run ***```ansible-playbook -i hosts site.yml --ask-pass```*** (the password is ansible)
* This will install, configure and start Kubernetes and Etcd on the above servers
* When the playbook is complete logon to the Kubernetes master (the server labelled kube_controller in the AWS console) and sudo to root (no password required) 
* Check whether our minions have joined the cluster by running the command ```kubectl get nodes```. The output should look like this 
```
NAME            STATUS    AGE
ip-10-0-1-111   Ready     5m
ip-10-0-1-112   Ready     5m
ip-10-0-1-113   Ready     5m
```
* Whilst still logged into the kubernetes master as root, clone this repo. 
* cd to the kubernetes directory
* Run the command ***```kubectl create -f colour-deployments.yaml```***  The output should look similar to this: 
```
  deployment "red-deployment" created
  deployment "green-deployment" created
  deployment "blue-deployment" created
```
* This has created 3 deployments each consisting of 3 replicas (i.e 3 pod for each of red, green amd blue) across the three minions. You can check this by running the commands ***```kubectl get deployments```*** and ***```kubectl describe deployments```***
* To make these deployments available to the outside world, create a service for each by running ***```kubectl create -f colour-service.yaml```*** 
* Check the status of each service by running ***```kubectl get services```*** The output should look something like this 
```
NAME            CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
blue-service    10.0.226.140   <nodes>       8888:31003/TCP   3m
green-service   10.0.207.137   <nodes>       8888:31002/TCP   3m
kubernetes      10.0.0.1       <none>        443/TCP          34m
red-service     10.0.213.33    <nodes>       8888:31001/TCP   3m
```
* You can now test each service in a browser by hitting any minion's public IP on either port 31001, 31002 or 31003
* You'll notice that the service on port 31001 returns ```{"colour": "blue"}```, the one of port 31002 returns ```{"colour": "green"}``` and the one on port 31003 returns ```{"colour": "red"}```
* Nothing amazing right now, i.e you've got three minions each running an instance of the red, green and blue app. However, the cool stuff happens when you reduce the instances of a app but are still able to hit it on any node. This is because we're fronting our app with a service.
