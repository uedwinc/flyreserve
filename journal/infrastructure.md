# Cloud platform setup

We will be using AWS for the microservice deployment

## Setup operations user and assign permissions

1. Create a new user (`ops-account`) on AWS and attach "IAMFullAccess" policy to it.

2. Configure aws cli for this user

```sh
aws configure
```

- Export access details as env variables (Gitpod):

```sh
export AWS_ACCESS_KEY_ID=""
gp env AWS_ACCESS_KEY_ID=""

export AWS_SECRET_ACCESS_KEY=""
gp env AWS_SECRET_ACCESS_KEY=""

export AWS_DEFAULT_REGION=us-east-2
gp env AWS_DEFAULT_REGION=us-east-2

export AWS_ACCOUNT_ID=""
gp env AWS_ACCOUNT_ID=""
```

3. Create a new group for the ops-account named Ops-Accounts

```sh
aws iam create-group --group-name Ops-Accounts
```

4. Add 'ops-account' user to the new group

```sh
aws iam add-user-to-group --user-name ops-account --group-name Ops-Accounts
```

5. Attach a set of permissions to the Ops-Account group

```sh
cd /aws-permissions-setup

chmod +x iam-permissions.sh

./iam-permissions.sh
```

6. Attach some special permissions to work with the AWS Elastic Kubernetes Service (EKS). We’ll need to create our own custom policy and attach it to the user group. The policy code is in [custom-eks-policy.json](/aws-permissions-setup/custom-eks-policy.json) file

- Run the following command to create a new policy named `EKS-Management` based on the custom-eks-policy.json file

```sh
cd /aws-permissions-setup

aws iam create-policy --policy-name EKS-Management \
--policy-document file://custom-eks-policy.json
```

- Next, attach the policy to the user group using the policy ARN (Amazon resource Number) generated from the previous step

```sh
aws iam attach-group-policy --group-name Ops-Accounts \
--policy-arn arn:aws:iam::"${AWS_ACCOUNT_ID}":policy/EKS-Management
```

## Setup infrastructure as code pipeline

We'll create a sandbox testing environment just as a test environment that will give us a chance to try out our IaC modules and pipelines.

Each environment’s code and pipeline will be managed independently in its own code repository.

We’ll build the following components:

  - A GitHub-hosted Git repository for a sandbox testing environment
  - A Terraform root module that defines the sandbox
  - A GitHub Actions CI/CD pipeline that can create a sandbox environment

### 1. Create the sandbox repository

- Create a new repository on GitHub. Give the new repository the name `flyreserve-env-sandbox` and select Private from the access options. You should also tick the `Add .gitignore` checkbox and choose `Terraform` from the drop-down

- You can either clone this new repository into your local development environment or use remote environemnts like Gitpod or GitHub Codespaces

### 2. Writing Terraform code for the sandbox environment

- We’ll be creating a simple starter file to test our Terraform-based tool chain

- We'll be creating isolated folder structures for related resources

- Create `global/s3` with terraform files to to define s3 bucket and dynamodb table

- Initialize and apply to create s3 and dynamodb table

- Next, configure terraform to store the state in s3 (with encryption and locking) by adding a `backend` configuration. 

- Again, initialize so that terraform can use the s3 backend provisioned
- `init` is idempotent (harmless to run multiple times)

- Terraform will automatically detect that you already have a state file locally and prompt you to copy it to the new S3 backend. Type 'yes'

- You can go to the console to see the terraform state in the s3 bucket

- Create `live/main.tf`, specify backend and define some local variables for testing

- `cd` into your working directory `live` and try running the `fmt` command to format your code

```sh
fmt main.tf
```

- Install provider

```sh
terraform init
```

- Validate syntax

```sh
terraform validate
```

- Plan to see what resources would be created (none for now as no resource is defined in _main.tf_)

```sh
terraform plan
```

- If you ever wanted to delete the S3 bucket and DynamoDB table
  - Go to the Terraform code, remove the backend configuration, and rerun terraform init to copy the Terraform state back to your local disk.
  - Run terraform destroy to delete the S3 bucket and DynamoDB table.

### 3. Building the pipeline (GitHub Actions)

Set up an automated CI/CD pipeline that will automatically apply the Terraform file that we’ve just created

**Set up secrets**

- Go to the `flyreserve-env-sandbox` repository on GitHub.

- Under _Settings_, select `Secrets and variables | Actions | New repository secret` and create a secret called AWS_ACCESS_KEY_ID. Enter your `ops-account` access key ID

- Repeat the process and create a secret named AWS_SECRET_ACCESS_KEY with the secret access key you generated earlier.

**Creating the workflow**

This is the set of steps that we want to run whenever a pipeline is triggered. For our microservices infrastructure pipeline, we’ll want a workflow that validates Terraform files and then applies them to our sandbox environment.

We'll use git tags for the trigger. This gives us a nice versioning history for the changes we are making to the environment. It also gives us a way of committing files to the repository without triggering a build.

- Go to the `flyreserve-env-sandbox` repository on GitHub.

- Under _Actions_, click `set up a workflow yourself`

- Edit the YAML file for the workflow. This includes steps to:
  - Configure the pipeline so that it runs whenever infrastructure is tagged with a label that starts with a `v`
  - Checkout our code from Git and create a copy in the ubuntu build env
  - Install an AWS authenticator tool
  - Apply Terraform
  - Publish assets. This action uploads a file called kubeconfig from the local working directory of the build environment to the GitHub Actions repository. It assumes that the file exists, so we’ll need to create that file.

- Lastly, commit the new file to the GitHub repository

**Testing the Pipeline**

- First pull the changes made into the clone on the local environment

```sh
git pull
```

- Create a tag to trigger Actions

```sh
git tag -a v0.1 -m "workflow test"
```

- Push the tag into remote GitHub repository

```sh
git push origin v0.1
```

- This should automatically trigger the workflow build in GitHub Actions.

- To see the status of the run, go back to the browser-based GitHub interface and navigate to Actions. Confirm that our workflow job has completed successfully.

- Click on the build of the workflow (Sandbox Environment Build) to see more details.

## Setup Cloud Infrastructure

We will be designing and writing Terraform code to create a working network, an AWS managed Kubernetes service, and a declarative GitOps server.

> Network (VPC, Subnets, Routing and Security)

> Kubernetes Service (Elastic Kubernetes Service (EKS))

> GitOps Deployment Server (Argo CD)

- Install kubectl

  - Follow documentation: https://kubernetes.io/docs/tasks/tools/

- Setting Up the Module Repositories

  - We’ll build modules for each of the architecturally significant parts of our system: networks, the API gateway, and the managed Amazon Kubernetes service (EKS)

  - Let’s create the following repositories for the modules we’ll be writing

  |Repository Name|Visibility|Description|
  |-----------------|------------|-------|
  |flyreserve-aws-network-module|Public|A Terraform module that creates the network|
  |flyreserve-aws-kubernetes-module|Public|A Terraform module that sets up EKS|
  |flyreserve-argo-cd-module|Public|A Terraform module that installs Argo CD into a cluster|

  - Add a .gitignore file for Terraform to these repositories

### 1. Network module

- Clone the _flyreserve-aws-network-module_ GitHub repository into your local environment and provision the following Terraform files in the root directory: `main.tf`, `variables.tf`, `outputs.tf`.

- Add a README to the repository for description

- In the module’s working directory, format the module’s code using:

```sh
terraform fmt
```

- Next, run `terraform init` so that Terraform can install the AWS provider libraries. Make sure AWS credentials are already configured

```sh
terraform init
```

- For debugging in case of error: https://developer.hashicorp.com/terraform/internals/debugging

- If you need to debug your Terraform code, you can set the environment variable `TF_LOG` to `INFO` or `DEBUG`. That will instruct Terraform to emit logging info to standard output.

- Finally, you can run the validate command to make sure that the module is syntactically correct:

```sh
terraform validate
```

- Add and commit your code to remote GitHub

```sh
git add .
git commit -m "network module created"
```

- We can use the module to create a real network in our sandbox environment.

### 2. Create a sandbox network

- Go to _main.tf_ for the _flyreserve-env-sandbox_ repository and enter a network configuration section

- Note that `YOUR_NETWORK_MODULE_REPO_PATH` equals The path to your module’s repository in GitHub. For example: github.com/***/module-aws-network

- Format and validate the code locally:

```sh
terraform fmt
terraform init
terraform validate
```

- If you need to debug the networking module and end up making code changes, you may need to run the following command in your sandbox environment directory:

```sh
terraform get -update
```

  - This will instruct Terraform to pull the latest version of the network module from GitHub

- If the code is valid, we can get a plan to validate the changes that Terraform will make when they are applied.

```sh
terraform plan
```

- Push the code to the GitHub repository and tag it for release:

```sh
git add .
git commit -m "initial network release"
git push origin
git tag -a v1.0 -m "network build"
git push origin v1.0
```

- Log in to GitHub and check the Actions tab in your sandbox environment repository to confirm pipeline ran successfully

- You can test that the VPC has been successfully created by running the following command to list the VPCs with a CIDR block that matches the one that we’ve defined:

```sh
aws ec2 describe-vpcs --filters Name=cidr,Values=10.10.0.0/16
```

5. Kubernetes module

- Clone the _module-aws-kubernetes_ GitHub repository into your local environment and provision the following in the root directory

  + [**outputs.tf**]()
  + [**main.tf**]()
  + [**variables.tf**]()

- Add a README to the repository for description

- In the module’s working directory, format and validate the module’s code using:

```sh
terraform fmt
terraform init
terraform validate
```

- Add and commit your code to remote GitHub

```sh
git add .
git commit -m "kubernetes module complete"
git push origin
```

- We can use the module to create a kubernetes cluster in our sandbox environment.

6. Create a sandbox Kubernetes cluster

- Update the _main.tf_ file of the sandbox environment at the #EKS Configuration placeholder so that it uses the Kubernetes module

- Commit, push and tag this file into your CI/CD infrastructure pipeline and create a working EKS cluster

```sh
git add .
git commit -m "initial k8s release"
git push origin
git tag -a v1.1 -m "k8s build"
git push origin v1.1
```

- Provisioning an EKS cluster takes time

- You can test that the cluster has been provisioned by running the following AWS CLI command:

```sh
aws eks list-clusters
```

- Destroy custer environment when not in use

7. Setting Up Argo CD

-  Install a GitOps deployment tool that will be used to release our services into our environment’s Kubernetes cluster

- We’ll be installing Argo CD on the Kubernetes system that we’ve just instantiated. We’ll use a Kubernetes provider;
this enables Terraform to issue Kubernetes commands and install the application to our new cluster. We’ll also use a package-management system called Helm to do the installation

- Create a _main.tf_ file in the root directory of the module-argo-cd Git repository

- Create a _variables.tf_ file for Argo CD

- Format and validate the code

```sh
terraform fmt
terraform init
terraform validate
```

- Commit your code changes and push them to the GitHub repository so that we can use the module in our sandbox environment

```sh
git add .
git commit -m "ArgoCD module init"
git push origin
```

8. Installing Argo CD in the sandbox

- Modify the sandbox module’s _main.tf_ file to install Argo CD for #GitOps Configuration

- Wait for the EKS build to complete 

- Tag, commit, and push the Terraform file into your CI/CD pipeline

```sh
git add .
git commit -m "initial ArgoCD release"
git push origin
git tag -a v1.2 -m "ArgoCD build"
git push origin v1.2
```

**Testing the Environment**

- In the Terraform code for our Kubernetes module, we added a local file resource to create a kubeconfig file. Now, we need to download that file so that we can connect to the EKS cluster using the kubectl application

- To retrieve this file, navigate to your sandbox GitHub repository in your browser and click on the Actions tab. You should see a list of builds with your latest run. When you select the build that you just performed, you should see an artifact called “kubeconfig” that you can click and download

Documentation: https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts#downloading-or-deleting-artifacts

- Extract the zip file to get a _kubeconfig_ file and store in any choiced location on your machine

- Set an environment variable called KUBECONFIG that points to it

```sh
export KUBECONFIG=~/Downloads/kubeconfig
```

- If you like, you can copy the kubeconfig file to ~/.kube/config and avoid having to set an environment variable. Just make sure you aren’t overwriting a Kubernetes configuration you’re already using.

- Now, run `kubectl` command to confirm that everything went well

```sh
kubectl get svc
```

- Result shows us that our network and EKS services were provisioned and we were able to successfully connect to the cluster

- Check to make sure that Argo CD has been installed in the cluster

```sh
kubectl get pods -n "argo"
```

**Cleaning Up the Infrastructure**

1. To destroy the sandbox environment:

- Navigate to the working directory of your sandbox environment code on your machine

- Pull the latest version of the code from the repository

```sh
git pull
```

- Install the Terraform providers that our environment code uses (we’ll need these so we can destroy the resources)

```sh
terraform init
```

- Enter the following command to destroy the sandbox environment

```sh
terraform destroy
```

- To verify that the EKS resources have been removed, you can run the following AWS CLI command to list EKS clusters

```sh
aws eks list-clusters
```

- You can also run the following commands to double-check that the other billable resources have been removed

```sh
aws ec2 describe-vpcs --filters Name=cidr,Values=10.10.0.0/16
aws elbv2 describe-load-balancers
```