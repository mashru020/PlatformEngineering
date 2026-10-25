Day 5: Terraform security and CI/CD
Study:
- Running Terraform from a CI/CD pipeline.
- Plan review and approval before production changes.
- OIDC-based authentication to AWS from GitHub Actions.
- Terraform formatting and validation checks.
- Secret management.
- Drift detection.
- Policy and security checks using tools such as Checkov or Trivy.
Practical exercise:
Create a GitHub Actions workflow that:
1. Checks out the repository.
2. Runs formatting and validation.
3. Performs security checks.
4. Generates a Terraform plan.
5. Requires an approval before applying production changes.