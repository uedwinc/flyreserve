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

### 3. Provisioning the Staging Infrastructure

Let’s do a quick test to make sure our updated infrastructure code works. Run the following Terraform commands to format and validate our updated code:

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

You can validate the status of your pipeline run in the browser-based GitHub console. If the pipeline job has succeeded, you now have a staging environment with a Kubernetes cluster, MySQL and Redis databases running and ready to use

We’ll need that Kubernetes cluster for our microservices deployment. So our next step will be to validate that it is up and running

### 4. Testing access to the Kubernetes cluster

In order to communicate with the staging Kubernetes cluster we’ll need the configuration details for the kubectl application

Set up your Kubernetes client environment by downloading the kubeconfig file that our GitHub Actions staging pipeline generated. Then set the KUBECONFIG environment variable to point to the configuration that you’ve just downloaded. For example:

```sh
export KUBECONFIG=~/Downloads/kubeconfig
```

To confirm that our staging cluster is running and the Kubernetes objects have been deployed, run:

```sh
kubectl get svc --all-namespaces
```

In the result you should see a list of all the Kubernetes services that we’ve deployed. That should include services for the Argo CD application and the Traefik ingress service. That means that our cluster is up and running and the services we need have been successfully provisioned

### 5. Create a Kubernetes secret

When our flight information microservices connects to MySQL, it will need a password. To avoid storing that password in plain text, we’re going to store it in a special Kubernetes object that keeps it hidden from unauthorized viewers

Run the following command to create and populate the Kubernetes secret for the MySQL password:

```sh
kubectl create secret generic \
mysql --from-literal=password=microservices -n microservices
```

The built-in secrets functions of Kubernetes are useful, but we recommend that you use something more feature rich for a proper implementation. There are lots of options available in this area, including [HashiCorp Vault](https://www.vaultproject.io/)

We now have a staging environment with an infrastructure that fits the needs of the microservices we’ve developed 

The next step will be to publish those microservices as containers so that we can deploy them into the environment

## Releasing Microservices

### Shipping the Flight Information Container

We’ll build a continuous integration and continuous delivery (CI/CD) pipeline to build and publish the flights microservice to a DockerHub container registry

**1. Configuring Docker Hub**

- Log in to [Docker Hub](https://hub.docker.com/) and create a repository for our flight application example by following these steps:

  - Go to the Docker Hub home page in your browser.
  - Log in to Docker Hub.
  - Click the "Create Repository" button.
  - Give the repository the name “flights”.
  - Click the "Create" button.

- Documentation: https://docs.docker.com/docker-hub/repos/

- With a Docker account and a container repository, we’re ready to build and push containers into it with a CI/CD pipeline.

**2. Configuring the Pipeline**

We’ll create our GitHub Actions workflow inside the flights service repository so that our CI pipeline can live right alongside the code.

Our workflow will need to communicate with Docker Hub in order to publish a container. So we’ll need to add our Docker Hub access information as secrets in the flights GitHub repository. Specifically, we’ll need to create and populate two secret keys.

|Key|Description|
|---|-----------|
|DOCKER_USERNAME|Your Docker account identity|
|DOCKER_PASSWORD|Your Docker account password|

- Using your browser, navigate to the Settings page of the *flyreserve-ms-flights* repository in GitHub.
- Under Secrets and variables dropdown, select Actions from the side navigation.
- Add the secret you want to define by clicking New repository secret.

> We aren’t adding any AWS account secrets to this repository. That’s because we won’t be deploying into an AWS instance in this pipeline. This workflow will only focus on pushing containers into the Docker Hub registry—not the deployment of the containers into our staging environment.

**3. Shipping the flight service container**

The GitHub Actions workflow we’ve defined is triggered by a release tag, and has the following steps:

  - Runs unit tests on the code
  - Builds a containerized version of the microservices
  - Pushes the container to the Docker Hub registry

- All we’ll need to do is push a tag called v1.0 into the release to trigger the CI/CD workflow

Let's just trigger this build using the Git‐Hub browser-based UI

- In your browser, navigate to the Code tab of the *flyreserve-ms-flight*s GitHub repository. On the righthand side of the screen, click the “Create a new release” link in the "Releases" section

- Next, enter the value v1.0 in the tag version field. Then click the Publish Release button at the bottom of the screen.

- Publishing a GitHub release with a tag of v1.0 is the same as pushing the tag with the Git CLI. The end result should be that our GitHub Actions workflow will have kicked off. You can navigate to the Actions tab in your repository and verify that a workflow named CICD has started. It will take a few minutes to run the makefile and package up the container. When it’s done, the flights service container will be pushed and ready to use in the Docker Hub registry.

- To validate that the container has been shipped, access your Docker Hub account in the browser and take a look at your repositories. You should see an entry for the flights container that was just pushed.

- We now have a containerized ms-flights microservice ready to be deployed into our staging environment. With our microservices pushed into the container registry, we can move on to the work of deploying the container into our staging environment.

### Deploying the Flights Service Container

We will use the Argo CD GitOps deployment tool we installed in our infrastructure stack

We’ll be creating a new deployment repository that will contain Helm packages. The Helm packages we build will describe how a microservice should be deployed. When they’re ready and pushed into the deployment repository, Argo CD will use them to deploy containers into the staging environment.

- To get started, create a new GitHub repository called *flyreserve-ms-deploy*. Once it’s ready, create a local clone of the repository in your development environment (I use GitPod).

- The easiest way to start working with Helm files is to use the Helm CLI application. Helm’s CLI allows you to create, install, and inspect Helm charts by using the Kubernetes API. Install helm from [documentation](https://helm.sh/docs/intro/install/) on your local machine. When you’ve done that, you’ll be ready to create the _ms-flights_ Helm chart.

- To create our skeleton chart, make sure you are in the root directory of your ms-deploy repository and run the following command:

```sh
helm create ms-flights
```

- When it’s done, Helm will have created a basic package that contains a *chart.yaml* file which describes our chart, a *values.yaml* file we can use to customize chart values, and a *templates* directory that contains a whole set of Kubernetes YAML templates for a basic deployment.

+ Update the flights deployment template

- The _/ms-flights/templates/deployment.yaml_ file is a Kubernetes object description file that declares the target deployment state for a Pod.

- We’ll need to make a few small changes for this deployment to work for our flights microservice.

- For our simple deployment, we’re going to use the default values that Helm has generated for most of the Deployment object’s properties. But we’ll need to update _spec.template.containers_ so that it works for the ms-flights container that we’ve built.

- Update the YAML for the containers property so that it contains the _env_, _ports_, _livenessProbe_, and _readinessProbe_ values

- See updated file: https://github.com/implementing-microservices/ms-deploy/blob/master/ms-flights/templates/deployment.yaml
- Change readinessProbe path from /ping to /health

+ Set package values

- We’ll need to update the details for the Docker image. Open the _values.yaml_ file in your favorite text editor and find the image key at the beginning of the YAML file. Update image with the details:

```yml
replicaCount: 1

image:
  repository: uedwinc/flights #use your own repository and tag names
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.0"
```

- Next, we’ll add MySQL connection values so the microservice can connect to the staging environment’s database services. Add the following YAML to your values file (you can add it immediately after the tag property):

```yml
MYSQL_HOST: rds.staging.msur-vpc.com
MYSQL_USER: microservices
MYSQL_DATABASE: microservices_db
MYSQLSecretName: mysql
MYSQLSecretKey: password
```

- Finally, find the ingress property near the end of the YAML file and update it with the following text:

```yml
ingress:
  enabled: true
  annotations: 
    kubernetes.io/ingress.class: traefik
  hosts:
    - host: flightsvc.com
      paths: ["/flights"]
  tls: []
```

- This definition lets our Ingress service know that it should route any messages sent to the host _flightsvc.com_ with a URI of _/flights_ to the flight information microservice. We won’t need to actually host the service at the flightsvc.com domain, we’ll just need to make sure that HTTP requests have those values if we want them to reach our service.

- Find full code here: https://github.com/implementing-microservices/ms-deploy/blob/master/ms-flights/values.yaml

- For a production environment, we’d probably have more values and template changes we’d want to make. But to get up and running, this is more than enough.

+ Test and commit the package

- The last thing we’ll need to do is a quick dry-run test to ensure that we haven’t made any syntax errors. You’ll need to have connectivity to your Kubernetes cluster, so make sure you still have that environment accessible.

- Run the following command to make sure that Helm will be able to build a package:

```sh
helm install --debug --dry-run flight-info .
```

- If it works, Helm will return a lot of YAML that shows the objects that it would generate. It should end with something that looks like this:

```
[... lots of YAML...]
backend:
    serviceName: flight-info-ms-flights
    servicePort: 80
NOTES:
1. Get the application URL by running these commands:
http://flightsvc.com/flights
```

- If everything looks good, commit the finished Helm files to the GitHub repository:

```sh
git add .
git commit -m "initial commit"
git push origin
```

- Now that the package files are available in the deployment monorepo, we’re ready to use them with the Argo CD GitOps deployment tool.

**Argo CD for GitOps Deployment**

- Argo CD is a GitOps deployment tool, designed to use a Git repository as the source for the desired deployment state for our workloads and services. When it checks a repository that we’ve specified, it determines whether the target state we defined matches the running state in the environment. If it doesn’t, Argo CD can “synchronize” the deployment to match what we declared in our Helm charts.

- We need to log in to the Argo CD instance that we’ve installed in staging, point to our ms-deploy repository, and set up a synchronized deployment.

- Make sure you’ve added the MySQL password Kubernetes Secret as described in “Create a Kubernetes secret”. Otherwise, the flight information service won’t be able to start up.

1. Log in to Argo CD

- Before we can log in to Argo CD, we’ll need to get the password for the Argo administrative user. Argo CD does something clever and makes the default password the same as the name of the Kubernetes object that it runs on. Run the following kubectl command to find the Argo CD Pod:

```sh
kubectl get pods -n "argocd" | grep argocd-server
```

- Copy the name of the Pod somewhere as that will be the password we’ll use to log in.

- In order to access the login screen and use our credentials, we’ll need to set up a port-forwarding rule. That’s because we haven’t properly defined a way to access our Kubernetes cluster from the internet. But thankfully kubectl provides a handy built-in tool for forwarding requests from your local machine into the cluster. 

Use the following to get it running:

```sh
kubectl port-forward svc/flyreserve-argocd-server 8443:443 -n "argocd"
```

- Now you should be able to navigate to _localhost:8443_ in your browser. You’ll almost definitely get a warning indicating that the site can’t be trusted. That’s OK and is expected at this point. Let your browser know that it is OK to proceed and you should then see a login screen

- Enter "admin" as your user ID and use the password you noted earlier and log in. If you can log in successfully, you’ll see a dashboard screen. Now we can move on to creating a reference to our flight information service deployment.

2. Sync and deploy a microservice

- In Argo CD, a microservice or workload that needs to be deployed is called an _application_. To deploy the flight-information microservice we’ll need to create a new “application” and configure it with values that reference the Helm package in the Git repository that we created earlier.

- Start by clicking the Create Application or New App button on the dashboard screen. When you click it, a web form will slide in from the righthand side of the screen that you’ll need to populate. This is where you define the metadata for the application and the location of the Helm package. In our case, we’ll want Argo CD to pull that from the deployments monorepo and the ms-flights directory within it.

- Use the values in Table 10-4 to set up your flight-information microservice deployment. Make sure you replace the value YOUR_DEPLOYMENTS_REPOSITORY_URL with the URL of the deployment repository from “Creating the Microservices Deployment
Repository” so that Argo CD can access your Helm packages.

|Section|Key|Value|
|-------|---|-----|
|GENERAL|Application name|flight-info|
|GENERAL|Project|default|
|GENERAL|Sync policy|manual|
|SOURCE|Repository URL|YOUR_DEPLOYMENTS_REPOSITORY_URL|
|SOURCE|Path|ms-flights|
|DESTINATION|Cluster|in-cluster (https://kubernetes.default.svc)|
|DESTINATION|Namespace|microservices|

- When you are done filling in the form, click the Create button.

- Documentation: https://argo-cd.readthedocs.io/en/stable/getting_started/#creating-apps-via-ui

- If you’ve created the application successfully, Argo CD will list the flight-info application in the dashboard

- However, while the application has been created, it’s not yet synchronized with the Deployment declaration, and the flight-info application in our cluster doesn’t match the description in our package. That’s because Argo CD hasn’t actually done the Deployment yet. To make that happen, click the flight-info application that we’ve just created, click the Sync button, and then click the Synchronize button in the window that slides in

- When you click Synchronize, Argo CD will do the work it needs to do to make your application match the state you’ve described in the Helm package. If everything goes smoothly, you’ll have a healthy, synchronized, and deployed microservice

- If your deployment status isn’t “healthy,” try clicking the pod (the last node on the far right of the tree). You’ll be able to view events and log messages that can help you troubleshoot the problem.

- Our container has been deployed in the Kubernetes cluster, its health checks and liveness checks have passed, and it is ready to receive requests.

3. Test the flights service

- Our flights microservice is now up and running in our AWS-hosted staging environment. In order to test the service with a request message, we’ll need to access Traefik’s load balancer, which will route our request to the containerized service. The first thing we’ll need is the load balancer’s network address. Since we didn’t set up a DNS entry, AWS will have given us a random address automatically. To get that address, run the following kubectl command:

```sh
kubectl get svc ms-traefik-ingress
```

- The EXTERNAL-IP is the address of the Traefik load balancer. Make a note of it for our test request.

- We’ll be using curl to send a request message to the flights microservice.

- We’re using `curl` because it has a lot of useful options, including the ability to set a host header in the HTTP request. That’s helpful for us because we need to set a host of _flightsvc.com_ for our ingress routing rule to work.

- Run the following curl command to send a test request message to the flights service (replacing {TRAEFIK-EXTERNAL-IP} with the address for your load balancer):

```sh
curl --header "Host: flightsvc.com" \
  {TRAEFIK-EXTERNAL-IP}/flights?flight_no=AA2532
```

- If all has gone well, you’ll get the details of that flight as a JSON-formatted response.

- You can use a dedicated API testing tool such as Postman or SoapUI to get a more user-friendly formatted version of the
response message.

- The HTTP request we’ve just made calls the ingress service, which in turn routes the message to the flights microservice based on the ingress rule we defined earlier. The flights microservice retrieves data from the database service we provisioned and returns a result to us through the load balancer. With that request, we’ve been able to bring together all the parts of our architecture deployment and test an end-to-end microservices architecture!

**Clean up**

- We’ll use a local Terraform client to bring down the infrastructure. Make sure you’re in the directory where your staging Terraform files are and run the following command:

```sh
terraform destroy
```

- You can check to make sure that the resources have been destroyed by using the AWS CLI or the AWS browser-based console.