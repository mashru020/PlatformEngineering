Day 1: Terraform fundamentals and state management
Study:
- Terraform workflow: init, validate, plan, apply, and destroy.
- Providers, resources, data sources, variables, outputs, and locals.
- State files and why state matters.
- Remote state and state locking.
- Resource dependencies and lifecycle management.
- Importing existing infrastructure into Terraform.
Practical exercise:
- Write Terraform configuration to provision an AWS VPC and an EC2 instance.
- Store state remotely in an S3 backend, with appropriate state-locking configuration for your Terraform version and backend.
- Use separate configurations or workspaces where appropriate to understand environment isolation.
Interview question: What happens if two engineers run Terraform against the same environment simultaneously?