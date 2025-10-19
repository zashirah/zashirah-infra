# Example: Adding a New CloudFormation Stack

This example demonstrates how to add a new S3 bucket stack to the repository.

## Step 1: Create the CloudFormation Template

Create a file named `s3-bucket.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Simple S3 bucket with versioning'

Parameters:
  BucketPrefix:
    Type: String
    Default: 'zashirah'
    Description: 'Prefix for the S3 bucket name'

Resources:
  MyBucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: !Sub '${BucketPrefix}-data-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      Tags:
        - Key: 'ManagedBy'
          Value: 'CloudFormation'
        - Key: 'Purpose'
          Value: 'Data Storage'

Outputs:
  BucketName:
    Description: 'Name of the S3 bucket'
    Value: !Ref MyBucket
    Export:
      Name: !Sub '${AWS::StackName}-BucketName'
  
  BucketArn:
    Description: 'ARN of the S3 bucket'
    Value: !GetAtt MyBucket.Arn
    Export:
      Name: !Sub '${AWS::StackName}-BucketArn'
```

## Step 2: Update Stack Configuration

Edit `.github/cloudformation-stacks.yaml` and add:

```yaml
stacks:
  - name: zashirah-iam-developer
    template: iam-developer-setup.yaml
    capabilities:
      - CAPABILITY_NAMED_IAM
    parameters:
      DeveloperUsername: developer
    description: IAM Developer User and Role with MFA
  
  # Add this new entry:
  - name: zashirah-s3-data
    template: s3-bucket.yaml
    parameters:
      BucketPrefix: zashirah
    description: S3 bucket for data storage
```

## Step 3: Validate Locally (Optional)

Before committing, validate the template:

```bash
aws cloudformation validate-template \
  --template-body file://s3-bucket.yaml
```

## Step 4: Commit and Push

```bash
git add s3-bucket.yaml .github/cloudformation-stacks.yaml
git commit -m "Add S3 bucket stack"
git push
```

## Step 5: Deployment

The GitHub Actions workflow will automatically:
1. Detect the changes to YAML files
2. Validate the new template
3. Deploy the new stack alongside existing stacks
4. Report the outputs

## Advanced Examples

### Stack with Multiple Parameters

```yaml
- name: zashirah-vpc
  template: vpc.yaml
  parameters:
    VpcCIDR: 10.0.0.0/16
    Environment: production
    EnableFlowLogs: 'true'
  description: VPC with public and private subnets
```

### Stack with Multiple Capabilities

```yaml
- name: zashirah-lambda-api
  template: lambda-api.yaml
  capabilities:
    - CAPABILITY_IAM
    - CAPABILITY_AUTO_EXPAND  # For nested stacks or transforms
  parameters:
    FunctionName: api-handler
  description: Lambda function with API Gateway
```

### Stack with No Parameters

```yaml
- name: zashirah-kms-keys
  template: kms-keys.yaml
  capabilities:
    - CAPABILITY_IAM
  description: KMS encryption keys
```

## Testing New Stacks

To test a new stack without affecting others:

1. Use workflow dispatch to deploy only your stack:
   - Go to Actions → Deploy CloudFormation Stacks
   - Click "Run workflow"
   - Enter your stack name (e.g., `zashirah-s3-data`)
   - Click "Run workflow"

2. Verify the deployment in the AWS Console

3. If successful, merge your changes to `main`

## Cleanup

To remove a stack:

1. Delete it via AWS CLI:
   ```bash
   aws cloudformation delete-stack --stack-name zashirah-s3-data
   ```

2. Remove the entry from `.github/cloudformation-stacks.yaml`

3. Commit and push the changes
