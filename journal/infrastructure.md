# Cloud platform setup

We will be using AWS for the microservice deployment

## Setup operations user and assign permissions

1. Create a new user (`ops-account`) on AWS and attach "IAMFullAccess" policy to it.

2. Configure aws cli for this user

```sh
aws configure
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

aws iam create-policy --policy-name EKS-Management\
--policy-document file://custom-eks-policy.json
```

- Next, attach the policy to the user group using the policy ARN (Amazon resource Number) generated from the previous step

```sh
aws iam attach-group-policy --group-name Ops-Accounts \
--policy-arn {YOUR_POLICY_ARN}
```