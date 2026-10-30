# Terraform workflow: init, validate, plan, apply, and destroy.

**Step 1: terraform init**

Initializes the working directory.

It downloads the required providers, configures the backend, and initializes modules.
```bash
terraform init
```
When do you use it?
- When starting a new Terraform project.
- After adding a new provider or module.
- After changing backend configuration.

**Step 2: terraform validate**

Checks whether the Terraform configuration is syntactically valid and internally consistent.
```bash
terraform validate
```
It can identify issues such as:
- Invalid argument names.
- Incorrect block structures.
- Invalid references.
It does not verify that the cloud resources can actually be created.

**Step 3: terraform plan**

Previews the infrastructure changes Terraform intends to make.
```bash
terraform plan
```
Terraform compares your configuration with its state and the available information about real infrastructure.

You can save the plan for later application:
```bash
terraform plan -out=tfplan
```
Then inspect it:
```bash
terraform show tfplan
```
**Step 4: terraform apply**

Executes the planned infrastructure changes.
```bash
terraform apply
```
Terraform displays a proposed plan and normally asks for confirmation.

To apply a previously saved plan:
```bash
terraform apply tfplan
```
This executes the saved plan without the usual interactive approval prompt. Protect saved plan files because they can contain sensitive information.

**Step 5: terraform destroy**

Destroys infrastructure managed by the current Terraform configuration and state.
```bash
terraform destroy
```
Terraform displays the proposed destruction and normally asks for confirmation.

Important: Never run this casually against production infrastructure. Review the proposed changes and confirm that you are operating in the correct environment.


# Providers, Resources, Data Sources, Variables, Outputs, and Locals
**2.1 Providers**

A provider allows Terraform to interact with an external platform or service.

Examples include:
- AWS
- Azure
- OCI
- Kubernetes
- Docker

Example using AWS:
```code
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}
```
The provider configuration specifies the AWS region.

Best practice: Pin provider versions to an appropriate range and commit .terraform.lock.hcl to version control.

Avoid hardcoding credentials in Terraform files. Prefer environment variables, workload identity, or the cloud provider's supported credential chain.

**2.2 Resources**

A resource represents infrastructure that Terraform manages.

Examples include:
- Virtual machines.
- Networks.
- Security groups.
- Storage buckets.
- Kubernetes resources.

Example:
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name = "app-server"
    Environment = "dev"
  }
}
```
Here:
- aws_instance is the resource type.
- app_server is the Terraform-local resource name.
- ami specifies the machine image.
- instance_type specifies the instance size.

Terraform uses the resource type and local name to identify the resource in its configuration and state.

**2.3 Data Sources**

A data source retrieves information about existing infrastructure or external data without declaring that Terraform should create the queried object.

Example:
```code
data "aws_vpc" "existing" {
  id = var.vpc_id
}

You can then reference the VPC:
output "vpc_cidr" {
  value = data.aws_vpc.existing.cidr_block
}
```
Resource vs. data source

| Resource  |	Data source |
| --------- | --------------|
| Manages an infrastructure object	| Reads information about an object|
| Can create, update, or delete it | Queries existing or externally available information |
| Declared using resource	| Declared using data |
| Example: create a VPC	| Example: retrieve an existing VPC


<mark>Interview question:</mark> How would you use Terraform if your company already has a network but needs you to provision application servers inside it?

<mark>Answer:</mark> I would retrieve the existing network using a data source and reference its ID when creating the application servers or associated networking resources.

**2.4 Variables**

Variables make Terraform configurations reusable.

Instead of hardcoding values, you can pass them from environment-specific configuration files or pipeline variables.

Declare variables in variables.tf:
```code
variable "instance_type" {
  description = "EC2 instance size"
  type        = string
  default     = "t3.medium"
}

variable "ami_id" {
  description = "AMI ID for the application server"
  type        = string
}
```
Provide values through terraform.tfvars:
```code
ami_id        = "ami-0123456789abcdef0"
instance_type = "t3.medium"
```
The AMI ID above is illustrative; use a valid image ID for your selected region.

Reference the variables:
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = var.instance_type
}
```
For different environments, you can maintain separate variable files:
```
environments/
├── dev.tfvars
├── test.tfvars
└── prod.tfvars
```
Apply a specific environment's values:
```bash
terraform plan -var-file="environments/dev.tfvars"
```
For production, use protected credentials and pipeline controls rather than relying solely on variable files.

**2.5 Outputs**

Outputs expose information generated by Terraform.

For example, you may want to retrieve a server's IP address after provisioning.
```code
output "server_public_ip" {
  description = "Public IP of the application server"
  value       = aws_instance.app_server.public_ip
}
```
After applying:
```bash
terraform output
```
To retrieve one output:
```bash
terraform output server_public_ip
```
Outputs are also useful for passing information between root modules and for automation.

Security note: Outputs may expose sensitive information. Marking an output as <mark>sensitive = true</mark> hides it in certain CLI displays, but does not automatically remove the value from state.

**2.6 Locals**

Locals define reusable expressions inside a Terraform module.

Example:
```code
locals {
  project_name = "payment-platform"

  common_tags = {
    Project     = local.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```
Use the local values in a resource:
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = local.common_tags
}
```
**Variables vs. locals**
|Variables	| Locals |
| ----------| -------|
| Receive values from outside the module	|Define expressions inside the module|
| Can have defaults	| Do not use variable-style defaults |
| Referenced as var.name	|Referenced as local.name |
| Useful for environment-specific configuration	| Useful for computed values and reducing duplication |



Remember: Variables are inputs; locals are internal reusable values; outputs expose results.

# State Files and Why State Matters

Terraform state is one of the most important concepts for a DevOps Engineer.

By default, Terraform stores state in a file named:
```
terraform.tfstate
```
The state file maps Terraform resource addresses to the real infrastructure objects Terraform manages.

For example:
```
aws_instance.app_server
        |
        v
EC2 instance ID: i-0123456789abcdef0
```
Terraform uses this mapping to understand which real-world object corresponds to a resource declared in your configuration.

<mark> Why does Terraform need state? </mark>

Suppose you configure:
```code
resource "aws_instance" "app_server" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.medium"
}
```

After provisioning, Terraform needs to remember which EC2 instance it created.

Without the correct state mapping, Terraform may be unable to identify the intended instance reliably and could propose creating another one.

State also records resource attributes and relationships used to calculate infrastructure changes.
Important state commands
```bash
List managed resources:
terraform state list
```
Show details about a resource:
```bash
terraform state show aws_instance.app_server
```
Display the state file:
```bash
terraform show
```
Refresh state information as part of a planning operation:
```bash
terraform plan -refresh-only
```
If the refresh-only plan is appropriate, apply it to update the state with observed remote changes:
```bash
terraform apply -refresh-only
```
This updates Terraform's recorded information; it does not intentionally change the infrastructure to match the configuration.
What happens if someone changes infrastructure manually?
Suppose Terraform created an EC2 instance with:
```
Instance type: t3.medium
```
An administrator changes it manually to:
```
Instance type: t3.large
```
Terraform can detect this drift when it refreshes the relevant resource information.

If your configuration still specifies t3.medium, a subsequent plan may propose changing the instance back.

The exact result depends on the resource, provider behavior, and whether the changed attribute is managed by the configuration.
State security
State can contain:
- Resource IDs.
- Network details.
- Connection information.
- Sensitive values returned by providers.
- Secrets, depending on the resources and configuration.
Therefore:
- Never commit terraform.tfstate to a public repository.
- Restrict access to state files.
- Encrypt state at rest and in transit.
- Use a remote backend with appropriate access controls for team projects.
- Never assume that marking an output as sensitive removes its value from state.

# Remote State and State Locking
When multiple engineers or CI/CD pipelines work with the same infrastructure, local state files become difficult to manage safely.

Remote state allows the team to store and access state through a shared backend.

**4.1 Remote state**

For example, an AWS S3 backend can store Terraform state remotely.
```code
A simplified configuration:
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "dev/application/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true
  }
}
```
This example assumes the bucket already exists and that the installed Terraform version supports the configuration shown.

The bucket should have appropriate access controls, encryption, and versioning. Configure credentials through a supported authentication mechanism.

Initialize the backend:
```bash
terraform init
```
If you are migrating existing local state, Terraform may prompt you to migrate it.

Important: Bootstrap the state-storage infrastructure separately or through a carefully planned process. You cannot assume the remote backend already exists just because it is declared in your Terraform configuration.

**4.2 State locking**

Imagine two engineers apply changes simultaneously:
- Engineer A plans and applies a network change.
- Engineer B applies a server change against the same state.
Without appropriate concurrency controls, concurrent operations could conflict or corrupt the state.

State locking helps prevent multiple Terraform operations from modifying the same state simultaneously when the backend supports locking.

With a supported S3 backend, **use_lockfile = true** enables S3-native state locking.

For other backends, the locking mechanism varies. Some support locking automatically, while others use different approaches.

<mark>Why is locking important?</mark>

It helps prevent concurrent state modifications and reduces the risk of inconsistent infrastructure management.

If Terraform reports that the state is locked:
1. Identify the operation holding the lock.
2. Determine whether it is still running.
3. Wait for it to finish if appropriate.
4. Investigate and remove a stale lock only when you are certain it is safe.

Do not routinely force-unlock state. Removing a lock while another operation is active can create serious problems.

Remote state vs. state locking

| Remote state	| State locking |
| ------------ | ---------------|
| Stores state in a shared backend | Coordinates concurrent state operations |
| Supports team collaboration	| Helps prevent conflicting operations |
| Can provide centralized access control and backups	| Works only when the backend supports the locking mechanism|
|Example: S3 backend	| Example: S3 lock file


Interview takeaway: Remote state provides shared state storage; locking protects concurrent operations. They solve related but different problems.

# Resource Dependencies and Lifecycle Management
Terraform needs to determine the correct order for creating, updating, and deleting resources.

**5.1 Implicit dependencies**

When one resource references another resource's attribute, Terraform normally infers the dependency automatically.

Example:
```code
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "app" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```
Terraform understands that the subnet depends on the VPC because the subnet references:
```
aws_vpc.main.id
```
It creates the VPC before the subnet.

**5.2 Explicit dependencies**

Sometimes a dependency exists even though one resource does not directly reference an attribute from another resource.

In such cases, use depends_on.
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  depends_on = [
    aws_security_group.app
  ]
}
```

This tells Terraform that the instance depends on the security group resource.

However, if the security group is intended to control the instance's network access, you should model the actual security-group association explicitly rather than relying only on depends_on.

Best practice: Prefer implicit dependencies when possible. <mark>Use depends_on only when Terraform cannot infer a real dependency.</mark>

**5.3 Lifecycle management**

The lifecycle block changes how Terraform manages a resource.
```
create_before_destroy
```
Creates a replacement resource before destroying the old one when replacement is required and the provider and infrastructure constraints permit it.
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  lifecycle {
    create_before_destroy = true
  }
}
```
This can reduce downtime during replacement, but it does not guarantee zero downtime. Resource naming constraints, quotas, networking, and application architecture may prevent seamless replacement.
```
prevent_destroy
```
Prevents Terraform from destroying a resource while this lifecycle rule remains in its configuration.
```bash
resource "aws_db_instance" "production" {
  # Database configuration omitted

  lifecycle {
    prevent_destroy = true
  }
}
```
If a proposed operation would destroy the resource, Terraform raises an error.

Important: prevent_destroy is not a substitute for backups or access controls. Removing the resource block also removes the lifecycle rule from the configuration, so it is not an absolute deletion safeguard.
```
ignore_changes
```
Tells Terraform to ignore changes to specified attributes when planning updates.
```code
resource "aws_instance" "app_server" {
  ami           = var.ami_id
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = [
      tags
    ]
  }
}
```
This may be useful when another system is intentionally responsible for managing tags.

Use it carefully: ignoring changes can hide configuration drift that you would otherwise want to detect.
```
replace_triggered_by
```
Triggers replacement of a resource when a specified managed resource or resource attribute changes.

For example, you might use it when a change in a related infrastructure resource requires an application instance to be replaced.

Choose the referenced resource or attribute carefully because this behavior can result in additional infrastructure replacement.

# Importing Existing Infrastructure into Terraform
Importing is useful when your company already has infrastructure that was created manually or through another tool, and you want Terraform to manage it.

For example, your company has an existing AWS EC2 instance.

You want Terraform to manage it without creating a duplicate.

The import process associates the existing infrastructure object with a Terraform resource address.

Step 1: Identify the existing infrastructure
Find the existing EC2 instance ID:
```
i-0123456789abcdef0
```
Step 2: Write a matching resource configuration
Create main.tf:
```code
resource "aws_instance" "existing_server" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.medium"
}
```
This is only an illustrative starting point. The actual configuration should match the existing instance's relevant attributes and the provider's requirements.

Step 3: Initialize Terraform
```bash
terraform init
```
Step 4: Import the existing resource

Using the CLI:
```bash
terraform import aws_instance.existing_server i-0123456789abcdef0
```
Terraform associates the existing instance with the specified resource address in state.

Important: CLI import does not automatically generate a complete Terraform configuration file. You must write and maintain the configuration yourself.

Step 5: Review the state
```bash
terraform state show aws_instance.existing_server
```
Review the actual attributes of the imported instance.

Step 6: Reconcile configuration and infrastructure
Run:
```bash
terraform plan
```
Review the proposed changes carefully.

If Terraform proposes replacing or destroying the existing instance unexpectedly, do not apply the plan until you understand why.

Update the configuration to reflect the intended state and repeat planning until the result is acceptable.
Alternative: Declarative import blocks
Recent Terraform versions support import blocks.

Example:
```code
import {
  to = aws_instance.existing_server
  id = "i-0123456789abcdef0"
}
```
You still need the corresponding resource configuration.

Then run:
```bash
terraform plan
terraform apply
```
Terraform can plan and execute the import as part of the normal workflow.

The declarative approach is particularly useful in repeatable infrastructure migrations and CI/CD processes.

**CLI import vs. import block**
|CLI import	| Import block |
|-----------| -------------|
|Runs an import command directly |	Declares the import in configuration|
| Associates an existing object with a resource address in state	|Plans the import as part of the normal workflow|
|Does not generate complete resource configuration automatically |	Can be used with configuration-generation workflows |
| Useful for one-off imports |	Useful for reviewable, repeatable imports|


Important distinction: Importing an object into state does not mean Terraform has automatically reconstructed every detail of the infrastructure configuration. You must reconcile the configuration and state before making changes.

For further details, consult the official Terraform documentation.