# Cross-Account CI/CD Deployment Guide

This guide provides step-by-step instructions for setting up a cross-account CI/CD pipeline using AWS Developer Tools.

## Prerequisites

- Two AWS accounts:
  - Central/Pipeline Account
  - Target/Production Account
- S3 bucket for artifacts (in central account)
- GitHub repository for application code

## Step 1: Deploy Central Account Resources

1. **Log in to your central AWS account**
2. **Deploy Central Resources**:
   ```bash
   aws cloudformation create-stack \
     --stack-name infra-shared-services-central-resources \
     --template-body file://../templates/Central-Resources-Minimal.yaml \
     --parameters \
       ParameterKey=AppName,ParameterValue=infra \
       ParameterKey=Environment,ParameterValue=shared-services \
       ParameterKey=ExistingBucketName,ParameterValue=cross-account-cicd-artifacts-bucket \
     --capabilities CAPABILITY_NAMED_IAM
   ```

3. **Update KMS Key Policy**:
   - Go to AWS Console > KMS > Customer managed keys
   - Select the key created by the stack
   - Edit the key policy to add target account access:
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

4. **Complete GitHub Connection**:
   - Go to Developer Tools > Settings > Connections
   - Find the connection created by the stack
   - Click "Update pending connection"
   - Follow the prompts to authorize AWS to access your GitHub account

## Step 2: Deploy Target Account Resources

1. **Log in to your target AWS account**

2. **Deploy CodeDeploy Resources**:
   ```bash
   aws cloudformation create-stack \
     --stack-name infra-prod-codedeploy \
     --template-body file://../templates/CodeDeploy-Resources-Only.yaml \
     --parameters \
       ParameterKey=AppName,ParameterValue=infra \
       ParameterKey=Environment,ParameterValue=prod \
       ParameterKey=CentralAccountId,ParameterValue=CENTRAL_ACCOUNT_ID \
       ParameterKey=ArtifactBucketName,ParameterValue=cross-account-cicd-artifacts-bucket \
       ParameterKey=KMSKeyArn,ParameterValue=KMS_KEY_ARN \
       ParameterKey=ExistingInstanceId,ParameterValue=EC2_INSTANCE_ID \
     --capabilities CAPABILITY_NAMED_IAM
   ```

3. **Deploy Cross-Account IAM Role**:
   ```bash
   aws cloudformation create-stack \
     --stack-name infra-prod-remote-iam \
     --template-body file://../templates/Remote-IAM.yaml \
     --parameters \
       ParameterKey=AppName,ParameterValue=infra \
       ParameterKey=Environment,ParameterValue=prod \
       ParameterKey=ArtifactBucket,ParameterValue=cross-account-cicd-artifacts-bucket \
       ParameterKey=AccountID,ParameterValue=CENTRAL_ACCOUNT_ID \
       ParameterKey=KMSARN,ParameterValue=KMS_KEY_ARN \
     --capabilities CAPABILITY_NAMED_IAM
   ```

4. **Configure EC2 Instance**:
   - Install CodeDeploy agent:
   ```bash
   sudo yum update -y
   sudo yum install -y ruby wget
   cd /home/ec2-user
   wget https://aws-codedeploy-eu-west-1.s3.amazonaws.com/latest/install
   chmod +x ./install
   sudo ./install auto
   sudo service codedeploy-agent start
   ```
   
   - Install and start nginx:
   ```bash
   sudo yum install -y nginx
   sudo systemctl start nginx
   sudo systemctl enable nginx
   ```

   - Add Name tag to EC2 instance:
   ```bash
   aws ec2 create-tags \
     --resources EC2_INSTANCE_ID \
     --tags Key=Name,Value=infra-prod-instance
   ```

## Step 3: Create Pipeline in Central Account

1. **Log in to your central AWS account**

2. **Get Cross-Account Role ARN from target account**:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name infra-prod-remote-iam \
     --query "Stacks[0].Outputs[?OutputKey=='CrossAccountRole'].OutputValue" \
     --output text
   ```

3. **Deploy Pipeline**:
   ```bash
   aws cloudformation create-stack \
     --stack-name infra-prod-pipeline \
     --template-body file://../templates/cross-account-pipeline.yaml \
     --parameters \
       ParameterKey=GitHubOwnerType,ParameterValue=personal \
       ParameterKey=GitHubOwner,ParameterValue=YOUR_GITHUB_USERNAME \
       ParameterKey=GitHubRepo,ParameterValue=cicd-cross-account \
       ParameterKey=GitHubBranch,ParameterValue=main \
       ParameterKey=CrossAccountRoleArn,ParameterValue=CROSS_ACCOUNT_ROLE_ARN \
       ParameterKey=TargetAccountId,ParameterValue=TARGET_ACCOUNT_ID \
     --capabilities CAPABILITY_NAMED_IAM
   ```

## Step 4: Prepare GitHub Repository

1. **Create GitHub Repository**:
   - Create a new repository named "cicd-cross-account"
   - Add the application files from the app-files directory

2. **Required Files**:
   - appspec.yml - Deployment instructions for CodeDeploy
   - buildspec.yml - Build instructions for CodeBuild
   - index.html - Sample web page
   - scripts/install_dependencies.sh - Script to install dependencies
   - scripts/start_application.sh - Script to start the application

3. **Push to GitHub**:
   ```bash
   git add .
   git commit -m "Initial commit for cross-account deployment"
   git push origin main
   ```

## Step 5: Monitor and Test

1. **Monitor Pipeline Execution**:
   - Go to AWS Console > CodePipeline in the central account
   - Watch the pipeline execute through Source and Deploy stages

2. **Verify Deployment**:
   - Go to the target account
   - Find the EC2 instance's public IP
   - Open in a browser: http://<public-ip>
   - You should see your deployed web page

3. **Test Changes**:
   - Make a change to the index.html file
   - Commit and push to GitHub
   - Watch the pipeline automatically deploy the change

## Troubleshooting

1. **Pipeline Source Stage Fails**:
   - Verify GitHub connection is properly set up
   - Check repository and branch names

2. **Pipeline Deploy Stage Fails**:
   - Verify cross-account role permissions
   - Check if CodeDeploy agent is running on EC2 instance
   - Verify KMS key policy allows target account access

3. **EC2 Instance Can't Access S3 Bucket**:
   - Verify EC2 instance role has proper S3 permissions
   - Check KMS key policy for decrypt permissions