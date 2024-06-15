# Release and change

- We’ll build a new infrastructure environment called _staging_.
- We’ll augment our code repository with a container delivery process. 
- We’ll implement a deployment process using the Argo CD GitOps tool.

- For this project, we’ll only deploy the flight information microservice

- We’ll be using three different GitHub repositories with their own pipelines and assets

## Provisioning the AWS-based staging environment

Update the infrastructure code of the sandbox env to reflect the needs of the flight information and flight reservation microservices

We’ll be adding three new components to our Terraform code to support our microservices:

  - An ingress controller and edge router that sends requests to microservices in Kubernetes
  - An AWS-based MySQL database instance for the flights microservice’s data
  - An AWS-based Redis database instance for the reservations microservice’s data

### The Ingress Module

In the development stage, we will be using an edge router called Traefik to route messages to our container-based microservices. We’ll implement it as our controller in the AWS environment as well. We’ll use Traefik to route messages from the load balancer to microservices deployed in Kubernetes. There are plenty of tools available to perform ingress routing. For example, [Nginx ingress controller](https://kubernetes.github.io/ingress-nginx/)

Create a new Terraform module repository (_flyreserve-aws-traefik-module_) with _main.tf_ and _variables.tf_ (https://github.com/uedwinc/flyreserve-aws-traefik-module). This module will install the Traefik ingress controller into the environment.

We’ve implemented Traefik using an AWS Network Load Balancer, which makes that kind of connectivity (provisioning an [AWS API Gateway](https://aws.amazon.com/api-gateway/) in front of the ingress controller to compose services into a single API) easier to implement.

### The Database Module

We need to provision two different databases in the infrastructure environment. We’ll need both a MySQL and a Redis database.

For our build, we’ve decided to use AWS managed versions of these database products. We will create Terraform-based modules to provision AWS hosted and managed database services for each environment. We will use the AWS ElastiCache service to provision a Redis data store and the AWS Relational Database (RDS) to provision a MySQL instance.

Create a new Terraform module repository (_flyreserve-aws-db-module_) with _main.tf_ and _variables.tf_ (https://github.com/implementing-microservices/module-aws-db). This module provisions both types of databases as well as the network configuration and access policies that the database service needs for operation.

## The Staging Infrastructure Project

The staging environment we need for releasing our microservices will be very similar to the sandbox environment

We’ll use Terraform to define the environment in code using the modules we wrote for the network, Kubernetes cluster, and Argo CD. We’ll complement those modules with the new database and ingress controller modules we’ve just described. Finally, we’ll use a GitHub Actions pipeline to provision the environment, just like we did for our sandbox environment.

Create a new private repository named _flyreserve-infra-staging-env_

Start off with all the code from the _flyreserve-env-sandbox_ repository

### 1. Configuring the Staging Workflow