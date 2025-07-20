# Cross-Account CI/CD Pipeline with AWS Developer Tools

This project demonstrates a GitOps approach to infrastructure deployment using AWS Developer Tools to implement a cross-account CI/CD pipeline. The solution enables centralized management of infrastructure deployments across multiple AWS accounts.

## Architecture Overview

![Cross-Account CI/CD Architecture](../diagrams/architecture.png)

The architecture consists of:

1. **Central/Pipeline Account**: Contains the CI/CD pipeline and shared resources
   - AWS CodePipeline for orchestration
   - S3 bucket for artifacts storage
   - KMS key for encryption
   - GitHub connection for source code integration

2. **Target Account(s)**: Where infrastructure is deployed
   - Cross-account IAM roles for secure access
   - AWS CodeDeploy for application deployment
   - EC2 instances running the deployed application

## Key Components

### CloudFormation Templates

| Template | Description |
|----------|-------------|
| `Central-Resources-Minimal.yaml` | Creates central resources (KMS key, roles, GitHub connection) |
| `Remote-IAM.yaml` | Creates cross-account IAM roles in target accounts |
| `CodeDeploy-Resources-Only.yaml` | Sets up CodeDeploy and EC2 instance role in target accounts |
| `cross-account-pipeline.yaml` | Creates the CI/CD pipeline in the central account |

### IAM Roles and Security

- **Cross-Account Role**: Allows the central account to assume a role in target accounts
- **CloudFormation Role**: Allows CloudFormation to create resources in target accounts
- **KMS Key Policy**: Controls encryption/decryption permissions across accounts
- **Least Privilege Principle**: All roles follow least privilege access

### GitOps Workflow

1. Infrastructure code is stored in a GitHub repository
2. Changes are made through pull requests and code reviews
3. Merges to main branch trigger the CI/CD pipeline
4. Pipeline deploys changes to target accounts
5. Version control provides audit trail and rollback capability

## Deployment Guide

### Prerequisites

- Multiple AWS accounts (central and target)
- S3 bucket for artifacts
- GitHub repository for infrastructure code

### Deployment Steps

1. **Set up Central Account**:
   - Deploy `Central-Resources-Minimal.yaml`
   - Update KMS key policy to allow target account access
   - Complete GitHub connection setup

2. **Set up Target Account**:
   - Deploy `CodeDeploy-Resources-Only.yaml`
   - Deploy `Remote-IAM.yaml`
   - Configure EC2 instance with CodeDeploy agent

3. **Create Pipeline**:
   - Deploy `cross-account-pipeline.yaml` in central account
   - Configure with target account role ARNs

4. **Push Code to GitHub**:
   - Add application files to repository
   - Pipeline will automatically deploy to target account

## Benefits

- **Centralized Management**: Manage deployments to multiple accounts from a single pipeline
- **Security**: Cross-account roles with least privilege access
- **Auditability**: All changes tracked in version control
- **Automation**: Automated testing and deployment reduces manual errors
- **Scalability**: Easy to add new target accounts or environments

## References

1. [AWS Cross-Account Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
2. [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)
3. [AWS CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html)
4. [Introduction to DevOps on AWS](https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/introduction.html)