Day 2: Terraform modules and reusable architecture
Study:
- Creating reusable Terraform modules.
- Module inputs and outputs.
- Module versioning.
- Environment-specific configuration.
- for_each versus count.
- Secrets and sensitive variables.
- Terraform testing, formatting, validation, and static analysis.
Practical exercise:

Create this project structure:
terraform/
├── modules/
│   ├── vpc/
│   ├── ec2/
│   ├── security-group/
│   └── s3/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── README.md

Design it so the same modules can be reused across environments without copying the entire configuration.