# TFE v5 Clustered Deployment on AWS using TF CLI

The goal of this guide is to walkthrough, step by step, a deployment of TFE v5 clustered on AWS using TF OSS and eventually TFC-hosted workspaces.

**NOTE** when referring to published guides or docs, these will be labeled as "official" versus docs that were created by HashiCorp field SEs that may not be officially supported by HashiCorp Support.

## References

### Official Pre-Install Checklist

https://www.terraform.io/docs/enterprise/before-installing/index.html

1. Choose a deployment method: **clustered, AKA v5** versus individual v4
2. Choose an operational mode: **production, external services** versus demo mode, etc.
3. Credentials Required: **TFE license file + TLS cert for TFE to use**
4. Data Storage: data storage service **S3**
5. Linux Instance: **Linux machine image** (for clustering v5) or versus a running Linux instance (if deploying individual v4)

### Deploying a TFE Cluster

https://www.terraform.io/docs/enterprise/install/cluster-aws.html

1. Follow the pre-install checklist.
2. Prepare the machine that will run Terraform.
3. Prepare some required AWS infrastructure.
4. Write a Terraform configuration that calls the deployment module.
5. Apply the configuration to deploy the cluster.

### Bootstrap Code

Forked TF code used to bootstrap AWS with VPC, network, bucket for license file, and more

https://github.com/raygj/terraform-content/tree/master/enterprise/deploy-TFE-v5/stage1

- Reference ([original source](https://github.com/hashicorp/private-terraform-enterprise/tree/master/examples/bootstrap-aws)) for VPC code and bucket configuration (Roger Berlind's original [v4 deployment](https://github.com/rberlind/private-terraform-enterprise/tree/automated-aws-pes-installation))

## Terraform CLI Dry Run

_before migrating code to TFC, execute a dry run to ensure code and variables are functional_

### Stage 1

Goal of Stage 1 code is to provide these perquisites required in Stage 2 (the TFE module):

* VPC
* Subnets (both public and private) spread across multiple AZs
* A DNS Zone
* A Certificate available in ACM for the hostname provided (or a wildcard certificate for the domain)
* A license file provided by your Technical Account Manager

[code is stored here](https://github.com/raygj/terraform-content/tree/master/enterprise/deploy-TFE-v5/stage1)

#### Steps

1. clone the [repo](https://github.com/raygj/terraform-content/), navigated to `.../terraform-content/enterprise/deploy-TFE-v5/stage1`
2. set values of `variables.tf` to reflect your environment and decisions
3. set value of module source in `main.tf` to point to your PMR, e.g., `source = "app.terraform.io/jray-hashi/vpc/aws"`
3. run `terraform plan` and validate names and other values are correct
4. move on to Stage 2 or run `terraform apply` now

### Stage 2

[code is stored here](https://github.com/raygj/terraform-content/tree/master/enterprise/deploy-TFE-v5/stage2)

#### Steps

1. clone the [repo](https://github.com/raygj/terraform-content/), navigated to `.../terraform-content/enterprise/deploy-TFE-v5/stage2`
2. set values of `variables.tf` to reflect your environment and decisions
3. set value of module source in `main.tf` to point to your PMR, e.g., `source = "app.terraform.io/jray-hashi/terraform-enterprise/aws"`
3. run `terraform plan` and validate names and other values are correct

# general notes

1. doc say's tf0.11 must be used, but TFE module has been upgraded to tf0.12 - but not completely clean
2. example bootstrap code is not tf0.12 compliant
3. sample TFE code is not tf0.12 compliant
4. a complete example of bootstrap and tfe module code (and supporting variable files, etc.) would be very helpful

# Appendix 1: TFE v5 Clustered Deployment on AWS using TFC

Building off the main walkthrough, the goal of this appendix is to step through the process of deploying TFE v5 clustered architecture from a workspaces running on TFC SaaS. Why? TFC offers security (Sentinel policy, secure variables) and collaboration (PMR, notifications) tooling to support this type of automated installation and it will be convenient to deploy a TFE instance at will.

## Prepare TFC

### Publish the TFE and VPC Module in your PMR

- fork the HashiCorp Official TFE Module to your private GitHub account

https://github.com/hashicorp/terraform-aws-terraform-enterprise

- for the AWS VPC module  to your private GitHub account

https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws

- publish the modules in your PMR

## Deploy TFE

Two stages will be used to deploy TFE because there are dependencies that must be fed into the official module and separating into two steps allows the TFE instance to be deployed into existing infrastructure and/or repaved with ease.

### Prerequisites: Stage 1

- these resources (and a valid license file) are required as input

* VPC
* Subnets (both public and private) spread across multiple AZs
* A DNS Zone
* A Certificate available in ACM for the hostname provided (or a wildcard certificate for the domain)
* A license file provided by your Technical Account Manager

- using Roger Berlind's code for a "public network" deployment forked into this repo to create the VPC, subnets, other network resources, security group, KMS key, and TFE source bucket (where TFE license file will reside)

1. create a TFC workspace called `tfe_deploy-stage1-demo-us-east`

- update VCS connection settings to the location of this forked repo, for example, my workspace is mounted at the repo path `https://github.com/raygj/terraform-content` and the working directory of the workspace is `/enterprise/deploy-TFE-v5/stage1`

![screenshot](/images/tfe-v5-deploy-stage1-workspace2.png)

**notes**

- make sure your AMI value in Stage 2 matches an available AMI in your default region
- WIP, TF CLI (OSS) code to configure the workspace in TFC started [HERE](https://github.com/raygj/terraform-content/tree/master/enterprise/tfe-provider-workspace-code)

2. using `...deploy-TFE-v5/stage1/network.tfvars` as a reference, configure Terraform Variables for the workspace as follows:

- set `namespace` to "<name>-tfe-v5" where "<name>" is some suitable prefix for your TFE deployment
- set `bucket_name` to the name of the TFE source bucket you wish to create
- set `cidr_block` to a valid CIDR block
- set `subnet_count` to the number of subnets you want in your VPC

3. set secure variables for `AWS_ACCESS_KEY_ID=<your_aws_key>` and `AWS_SECRET_ACCESS_KEY=<your_aws_secret_key>`

- example:

![screenshot](/images/tfe-v5-terraform-vars2.png)


4. run `queue plan` to initialize the Stage 1 Terraform configuration and download providers.
5. troubleshoot any TF errors, if successful you will receive feedback from TF `Plan: 11 to add, 0 to change, 0 to destory.`
6. hit `confirm & apply` to provision the Stage 1 resources.
7. note the `kms_id`, `security_group_id`, `subnet_ids`, and `vpc_id` outputs which you will need in Stage 2.
8. add your PTFE license file to your PTFE source bucket that was created. You can do this in the AWS Console.

**note** all of subnets will be public unless you modify the CIDR blocks for the security group configuration of `resource "aws_security_group"` in the `...deploy-TFE-v5/stage1/main.tf` config

### Terraform Code: Stage 2

- goal is to use the data source "terraform_remote_state" to fetch the VPC ID of the VPC provisioned in the Stage 1 workspace

- references:

https://www.terraform.io/docs/providers/terraform/index.html

https://www.terraform.io/docs/providers/terraform/d/remote_state.html

- remote-state data sources in TFC/TFE use the security context/creds of the user executing the run to access remote workspace outputs

**note** ^^ this is not documented, need to verify and test :-)

https://www.terraform.io/docs/cloud/run/index.html#cross-workspace-state-access

- cross-workspace-state-access using TF CLI, a CLI [credential file](https://www.terraform.io/docs/commands/cli-config.html) must be present

- the `variables.tf` or TFC Terraform variables value will include the HCL to configure the "terraform_remote_state"

```

# gather infra values from Stage 1 state stored in TFC

data "terraform_remote_state" "vpc" {
  backend = "remote"
  config {
    organization = "jray-hashi"
    workspaces = {
      name = "tfe_deploy-stage1-demo-us-east"
    }
  }
}

```

- the `main.tf` code will include a reference to the variable

```

vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id


```

**WIP note** need to make sure all this code is TF 0.12 compliant
