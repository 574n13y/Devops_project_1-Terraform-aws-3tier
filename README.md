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
