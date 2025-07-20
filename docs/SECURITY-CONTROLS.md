# Security Controls for Cross-Account CI/CD

This document outlines the security controls implemented in our cross-account CI/CD solution to ensure secure infrastructure deployment across multiple AWS accounts.

## Identity and Access Management

### Cross-Account Roles

Our solution implements the following cross-account roles with least privilege access:

1. **CrossAccountRole**:
   - Assumed by the pipeline in the central account
   - Limited to specific actions required for deployment
   - Trust relationship restricted to the central account
   - Example policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::CENTRAL_ACCOUNT_ID:root"
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```

2. **CloudFormationRole**:
   - Used by CloudFormation to create resources
   - Permissions limited to required services
   - Assumed only during deployment
   - Example policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Service": "cloudformation.amazonaws.com"
         },
         "Action": "sts:AssumeRole"
       }
     ]
   }
   ```

### EC2 Instance Role

The EC2 instance role follows least privilege principles:

- Access limited to S3 artifacts bucket
- KMS decrypt permissions for specific key only
- CodeDeploy agent permissions
- Example policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::ARTIFACT_BUCKET_NAME",
        "arn:aws:s3:::ARTIFACT_BUCKET_NAME/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "KMS_KEY_ARN"
    }
  ]
}
```

## Encryption

### KMS Key Management

Our solution uses AWS KMS for encryption with the following controls:

1. **Centralized Key Management**:
   - KMS key created in central account
   - Key policy allows specific cross-account access
   - Key rotation enabled

2. **Cross-Account Key Policy**:
   - Target accounts granted decrypt-only permissions
   - Permissions limited to specific roles
   - Example policy:
   ```json
   {
     "Sid": "Allow target account to use the key",
     "Effect": "Allow",
     "Principal": {
       "AWS": "arn:aws:iam::TARGET_ACCOUNT_ID:root"
     },
     "Action": [
       "kms:Decrypt",
       "kms:DescribeKey"
     ],
     "Resource": "*"
   }
   ```

3. **S3 Bucket Encryption**:
   - Server-side encryption with KMS
   - Bucket policy enforces encryption
   - Example policy:
   ```json
   {
     "Sid": "DenyUnEncryptedObjectUploads",
     "Effect": "Deny",
     "Principal": "*",
     "Action": "s3:PutObject",
     "Resource": "arn:aws:s3:::BUCKET_NAME/*",
     "Condition": {
       "StringNotEquals": {
         "s3:x-amz-server-side-encryption": "aws:kms"
       }
     }
   }
   ```

## Network Security

### Security Groups

EC2 instances are protected with security groups:

- Inbound access limited to required ports (HTTP/HTTPS)
- SSH access restricted to authorized IPs or bastion hosts
- All other ports closed by default

### VPC Configuration

- Resources deployed in private subnets where possible
- Public subnets only for resources requiring internet access
- NAT Gateway for outbound internet access from private subnets
- VPC Flow Logs enabled for network monitoring

## Monitoring and Auditing

### CloudTrail

- CloudTrail enabled in all accounts
- Logs stored in S3 with encryption
- Log file validation enabled
- Multi-region logging configured

### CloudWatch

- CloudWatch Logs for CodePipeline, CodeBuild, and CodeDeploy
- Metric alarms for pipeline failures
- Dashboard for monitoring deployment status
- Log retention policies configured

### AWS Config

- Config rules to enforce security best practices
- Continuous monitoring of resource configurations
- Automated remediation for non-compliant resources

## Secure CI/CD Practices

### Source Control

- Branch protection rules for main branch
- Pull request reviews required
- Signed commits enforced
- Secrets detection in pre-commit hooks

### Artifact Management

- Immutable artifacts with versioning
- Integrity validation with checksums
- Secure storage in S3 with encryption
- Access logging enabled

### Deployment Controls

- Environment-specific approvals
- Automated security scanning
- Rollback capability for failed deployments
- Deployment guardrails for production

## Compliance Documentation

Our solution includes documentation to demonstrate compliance with:

- AWS Well-Architected Framework Security Pillar
- NIST Cybersecurity Framework
- CIS AWS Foundations Benchmark
- SOC 2 requirements (if applicable)

## Tools and Services for Identity and Permissions Management

1. **AWS IAM**:
   - Role-based access control
   - Policy conditions for fine-grained permissions
   - Permission boundaries for delegation

2. **AWS Organizations**:
   - Service control policies (SCPs)
   - Account segregation by environment
   - Centralized policy management

3. **AWS SSO**:
   - Centralized identity management
   - Integration with corporate directory
   - Attribute-based access control

4. **AWS KMS**:
   - Centralized key management
   - Cross-account access controls
   - Key rotation and monitoring