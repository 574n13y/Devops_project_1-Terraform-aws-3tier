# Devops project with Terraform

- Use Terraform to set up infrastructure on AWS.
- The infrastructure includes VPC, NAT gateway, security group, Auto Scaling group, EC2, ALB, Elastic IP, and Certificate Manager (SSL/TLS certificates).
- Note that this is a 3-Tier VPC network architecture. We rely on the Terraform module “terraform-aws-modules/vpc/aws” to create most of the network components here.

## Architecture diagram
  ![image](https://github.com/574n13y/Devops/assets/35293085/142d04ed-e1d2-49f2-9ccb-e78e88088970)

## Tools 
 - [Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
   ``
   $ brew tap hashicorp/tap
   $ brew install hashicorp/tap/terraform
   $ terraform -version
   ``
 - [Install awscli](https://formulae.brew.sh/formula/awscli) OR [awscli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
   `
   $ brew install awscli
   $ aws --version
   `
 - [Install aws-vault](https://github.com/99designs/aws-vault)
   ``
   $ brew install --cask aws-vault
   ``
 - [Install Visual Studio Code](https://code.visualstudio.com/download)

## Prepare AWS account and user
 - You need an AWS account and an IAM user (console and programmatic access). If you don’t have an AWS account, sign up for it, but make sure you have an international payment card.
 - For lab environments, an IAM user with the managed policy “AdministratorAccess” is fine. For better security, you should also apply the “ForceMFA” policy. You can skip this extra setting for now.
 - You can get the content of the ForceMFA policy here:
   ```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowViewAccountInfo",
            "Effect": "Allow",
            "Action": [
                "iam:GetAccountPasswordPolicy",
                "iam:GetAccountSummary",
                "iam:ListVirtualMFADevices",
                "iam:ListUsers"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowManageOwnPasswords",
            "Effect": "Allow",
            "Action": [
                "iam:ChangePassword",
                "iam:GetUser"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnAccessKeys",
            "Effect": "Allow",
            "Action": [
                "iam:CreateAccessKey",
                "iam:DeleteAccessKey",
                "iam:ListAccessKeys",
                "iam:UpdateAccessKey"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnSigningCertificates",
            "Effect": "Allow",
            "Action": [
                "iam:DeleteSigningCertificate",
                "iam:ListSigningCertificates",
                "iam:UpdateSigningCertificate",
                "iam:UploadSigningCertificate"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnSSHPublicKeys",
            "Effect": "Allow",
            "Action": [
                "iam:DeleteSSHPublicKey",
                "iam:GetSSHPublicKey",
                "iam:ListSSHPublicKeys",
                "iam:UpdateSSHPublicKey",
                "iam:UploadSSHPublicKey"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnGitCredentials",
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceSpecificCredential",
                "iam:DeleteServiceSpecificCredential",
                "iam:ListServiceSpecificCredentials",
                "iam:ResetServiceSpecificCredential",
                "iam:UpdateServiceSpecificCredential"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnVirtualMFADevice",
            "Effect": "Allow",
            "Action": [
                "iam:CreateVirtualMFADevice",
                "iam:DeleteVirtualMFADevice"
            ],
            "Resource": "arn:aws:iam::*:mfa/${aws:username}"
        },
        {
            "Sid": "AllowManageOwnUserMFA",
            "Effect": "Allow",
            "Action": [
                "iam:DeactivateMFADevice",
                "iam:EnableMFADevice",
                "iam:ListMFADevices",
                "iam:ResyncMFADevice"
            ],
            "Resource": "arn:aws:iam::*:user/${aws:username}"
        },
        {
            "Sid": "DenyAllExceptListedIfNoMFA",
            "Effect": "Deny",
            "NotAction": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:GetUser",
                "iam:ListMFADevices",
                "iam:ListVirtualMFADevices",
                "iam:ResyncMFADevice",
                "sts:GetSessionToken",
                "iam:ListUsers"
            ],
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
   }
   ```
 - Use aws-vault to add (store) your IAM user’s access key. You may name the profile “terraform.test”.
   ```
   $ aws-vault add terraform.test
   $ aws-vault list
   $ aws-vault exec terraform.test -d 24h
   ```
 - If you enable MFA
   ```
   # https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html
   # .aws/config

   [default]
   region=us-west-2
   # output=json

   [profile terraform.test]
    region=us-east-1
   mfa_serial=arn:aws:iam::xxxxxxxxxxxx:mfa/terraform.test
   ```
 ## Register a domain
  - The domain can be registered from any domain registrar or right in the same AWS account depending on your needs. Owning a domain name will typically cost between $3 and $20 per year.
  - You also need to create a Route53 hosted zone for the above domain in the same AWS account and get the NS record information from there.
  - Then go to your domain settings and update its nameservers to route traffic to your AWS account’s Route53.

## Remote state storage with S3 and DynamoDB
  - If you choose not to use remote state for a simpler approach, you can skip this step, but be sure to comment out the contents of the “t1–02-backend.tf” file later.
  - Terraform state (“terraform.tfstate" file) can also be stored remotely. In this lab, you need to create an AWS S3 bucket in advance to store the state file as follows:
    1. Bucket name: devops-projects-infra-test
    2. Region: US-East (N.Virginia) us-east-1
    3. Bucket Versioning: Enable
    4. Create folder: dev/devops-01
       
 - Then, create a Dynamo DB Table that is used for state locking:
   1. Table Name: dev-devops-01
   2. Partition key (Primary Key): LockID (Type as String)

## Let's run the terraform now 
  ```
   terraform init
   terraform plan
   terraform apply
 ```

## Explore and confirm AWS resources
  - Check your email for the SNS Subscription
  - EC2 Instances (1 bastion and 2 web servers)
  - Auto Scaling Groups (and Launch template)
  - Load Balancers
  - Target Groups (and Health checks)
  - CloudWatch -> ALB Alarm, ASG Alarm, CIS Alarms, Synthetics
  - Access the service domain
  - Download the remote state file if needed:
    ```
    $ terraform state pull
    ```
    
  - Log in to the Bastion instance:
    ```
    $ ssh -i private-key/terraform-key.pem ec2-user@BastionHost-Public IPv4
    ```
    
## Cleanup 
  - terraform destroy
  - Some AWS resources you have to delete manually:
    ```
    CloudWatch > Log groups
    CloudWatch > Alarms
    Lambda > Functions
    S3 bucket, DynamoDB
    ```
    
  - and also the following local files:
    ```
    # Delete Files
    rm -rf .terraform*
    rm -rf terraform.tfstate*
    ```
