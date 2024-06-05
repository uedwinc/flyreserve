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