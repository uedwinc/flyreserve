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

Create a new Terraform module repository (_flyreserve-aws-db-module_) with _main.tf_ and _variables.tf_ (https://github.com/uedwinc/flyreserve-aws-db-module). This module provisions both types of databases as well as the network configuration and access policies that the database service needs for operation.

## The Staging Infrastructure Project

The staging environment we need for releasing our microservices will be very similar to the sandbox environment

We’ll use Terraform to define the environment in code using the modules we wrote for the network, Kubernetes cluster, and Argo CD. We’ll complement those modules with the new database and ingress controller modules we’ve just described. Finally, we’ll use a GitHub Actions pipeline to provision the environment, just like we did for our sandbox environment.

Create a new private repository named _flyreserve-infra-staging-env_

Start off with all the code from the _flyreserve-env-sandbox_ repository except the `global/s3` directory as s3 only need to be created once. There's also no need to have the live folder, so you can move the contents directly into the root directory. Adjust your workflow to reflect that.

### 1. Configuring the Staging Workflow

We’ll need to add AWS access management credentials and a MySQL password to the repository’s secrets

Add the following secrets:

|Key|Value|
|---|-----|
|AWS_ACCESS_KEY_ID|The access key ID for your AWS operator user|
|AWS_SECRET_ACCESS_KEY|The secret key for your operator user|
|MYSQL_PASSWORD|microservices|
|

Use the value "microservices" for the MYSQL_PASSWORD secret. This password will be used when we provision the AWS RDS database

The staging workflow will automatically generate a kubeconfig file as part of the provisioning process. This file contains connection information so you can connect to the Kubernetes cluster that we’ll create on EKS. If this code repository is public, that file will be available to anyone who visits your repository. In theory, this shouldn’t be a problem. Our EKS cluster requires AWS credentials to authenticate and connect. That means even with the kubeconfig file an attacker shouldn’t be able to connect to your cluster, unless they also have your AWS credentials.

### 2. Editing the Staging Infrastructure Code

We’ll be editing the _main.tf_ file that defines the staging environment

Next, we’ll need to give our AWS operator a few more permissions since we’ve added some new database modules, but the operator account we’re using isn’t allowed to create or work with those AWS resources. We’ll do this by creating a new IAM group for database work. When the group is set up, we’ll add our operator account to the group so it inherits those permissions

- Run the following AWS CLI command to create a new group called DB-Ops:

```sh
aws iam create-group --group-name DB-Ops
```

- Next, we can run the following command to attach access policies for RDS and ElastiCache to the group:

```sh
aws iam attach-group-policy --group-name DB-Ops \
--policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess && \
aws iam attach-group-policy --group-name DB-Ops \
--policy-arn arn:aws:iam::aws:policy/AmazonElastiCacheFullAccess
```

- Finally, use a CLI command to add our Ops account to the group we’ve just created:

```sh
aws iam add-user-to-group --user-name ops-account --group-name DB-Ops
```

- Let’s do a quick test to make sure our updated infrastructure code works. Run the following Terraform commands to format and validate our updated code:

```sh
terraform fmt
terraform init
terraform validate
terraform plan
```

- Now we’re ready to commit the infrastructure code and kick off the CI/CD pipeline. Let’s start by committing our updated Terraform code to your forked repository:

```sh
git add .
git commit -m "Staging environment with databases"
git push origin
```

- Our workflow gets triggered when we push a release tag that starts with a v. Use the following Git commands to create a new v1.0 tag and push it to your forked repository:

```sh
git tag -a v1.0 -m "Initial staging environment build"
git push origin v1.0
```

You can validate the status of your pipeline run in the browser-based GitHub console. If the pipeline job has succeeded, you now have a staging environment with a Kubernetes cluster and MySQL and Redis databases running and ready to use

We’ll need that Kubernetes cluster for our microservices deployment. So our next step will be to validate that it is up and running