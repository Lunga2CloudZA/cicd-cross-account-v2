# Cross-Account CI/CD with AWS Developer Tools

This project demonstrates a GitOps approach to infrastructure deployment using AWS Developer Tools to implement a cross-account CI/CD pipeline.

## Repository Structure

```
Cross-Account-CICD/
├── app-files/              # Application files for deployment
│   ├── appspec.yml         # CodeDeploy configuration
│   ├── buildspec.yml       # CodeBuild configuration
│   ├── index.html          # Sample web application
│   └── scripts/            # Deployment scripts
│       ├── install_dependencies.sh
│       └── start_application.sh
├── docs/                   # Documentation
│   ├── DEPLOYMENT-GUIDE.md # Step-by-step deployment instructions
│   ├── GITOPS-METHODOLOGY.md # GitOps methodology explanation
│   ├── README.md           # Project overview
│   ├── SECURITY-CONTROLS.md # Security controls documentation
│   └── SOW-EXAMPLE.md      # Example Statement of Work
├── scripts/                # Utility scripts
└── templates/              # CloudFormation templates
    ├── Central-Resources-Minimal.yaml # Central account resources
    ├── CodeDeploy-Resources-Only.yaml # Target account CodeDeploy resources
    ├── Remote-IAM.yaml     # Target account IAM roles
    ├── cross-account-pipeline.yaml # Pipeline configuration
    └── kms-policy.json     # KMS key policy example
```

## Quick Start

1. Deploy central account resources:
   ```bash
   aws cloudformation deploy \
     --template-file templates/Central-Resources-Minimal.yaml \
     --stack-name central-resources \
     --parameter-overrides \
       AppName=infra \
       Environment=shared-services \
       ExistingBucketName=cross-account-cicd-artifacts-bucket \
     --capabilities CAPABILITY_NAMED_IAM
   ```

2. Deploy target account resources:
   ```bash
   aws cloudformation deploy \
     --template-file templates/CodeDeploy-Resources-Only.yaml \
     --stack-name codedeploy-resources \
     --parameter-overrides \
       AppName=infra \
       Environment=prod \
       CentralAccountId=CENTRAL_ACCOUNT_ID \
       ArtifactBucketName=cross-account-cicd-artifacts-bucket \
       KMSKeyArn=KMS_KEY_ARN \
       ExistingInstanceId=EC2_INSTANCE_ID \
     --capabilities CAPABILITY_NAMED_IAM
   ```

3. Deploy cross-account IAM roles:
   ```bash
   aws cloudformation deploy \
     --template-file templates/Remote-IAM.yaml \
     --stack-name remote-iam \
     --parameter-overrides \
       AppName=infra \
       Environment=prod \
       ArtifactBucket=cross-account-cicd-artifacts-bucket \
       AccountID=CENTRAL_ACCOUNT_ID \
       KMSARN=KMS_KEY_ARN \
     --capabilities CAPABILITY_NAMED_IAM
   ```

4. Deploy pipeline:
   ```bash
   aws cloudformation deploy \
     --template-file templates/cross-account-pipeline.yaml \
     --stack-name cross-account-pipeline \
     --parameter-overrides \
       GitHubOwnerType=personal \
       GitHubOwner=YOUR_GITHUB_USERNAME \
       GitHubRepo=cicd-cross-account \
       CrossAccountRoleArn=CROSS_ACCOUNT_ROLE_ARN \
       TargetAccountId=TARGET_ACCOUNT_ID \
     --capabilities CAPABILITY_NAMED_IAM
   ```

## Documentation

For detailed information, see the following documentation:

- [Project Overview](docs/README.md)
- [Deployment Guide](docs/DEPLOYMENT-GUIDE.md)
- [GitOps Methodology](docs/GITOPS-METHODOLOGY.md)
- [Security Controls](docs/SECURITY-CONTROLS.md)
- [Example Statement of Work](docs/SOW-EXAMPLE.md)

## License

This project is licensed under the MIT License - see the LICENSE file for details.