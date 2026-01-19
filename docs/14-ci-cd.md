# CI/CD and Automation

This project uses [GitHub Actions](https://github.com/features/actions) for Continuous Integration (CI). The pipeline is configured to automatically validate code quality whenever changes are pushed to the repository.

## Workflow: `ci.yml`

The CI workflow performs the following checks:
1.  **Ansible Lint**: Validates the syntax and best practices of the Ansible playbooks and roles located in the `ansible/` directory.
2.  **Documentation Lint**: Checks Markdown files in `docs/` for formatting issues to ensure high-quality documentation.

## GitHub Hosted Runners & Network Security

To optimize costs, this project utilizes **GitHub Hosted Runners** (the standard `ubuntu-latest` environments provided by GitHub).

### Important Limitation
GitHub Hosted Runners execute on the public internet. They **do not** have access to your private AWS VPC resources (such as the private Subnet where the Head Node or Monitoring instances reside) by default.

Because of this:
*   The current CI pipeline is limited to **Static Analysis** (linting, syntax checking).
*   It **cannot** directly trigger deployments or run configuration updates against your running cluster instances.

### Future Expansion
If automated deployment is required in the future, consider one of the following strategies:
1.  **Self-Hosted Runners**: Deploy a GitHub Runner inside your AWS VPC. This allows the runner to communicate directly with your private instances.
2.  **AWS Systems Manager (SSM)**: Configure the GitHub Hosted Runner to authenticate with AWS (via OIDC) and use SSM Run Command to execute tasks on the private instances.
