# GitHub Actions Workflow for CloudFormation Deployment

## Overview

This repository uses GitHub Actions to automatically deploy AWS CloudFormation stacks. The workflow is designed to be dynamic and extensible, making it easy to add new infrastructure components as the project grows.

## Workflow Features

- ✅ **Automatic deployment** on push to `main` or pull request creation
- ✅ **Manual deployment** via workflow dispatch for specific stacks
- ✅ **Template validation** before deployment
- ✅ **Idempotent operations** - safely handles both stack creation and updates
- ✅ **Detailed logging** with stack outputs and error reporting
- ✅ **PR comments** with deployment results and stack outputs
- ✅ **Sequential deployment** to avoid conflicts
- ✅ **Fail-fast disabled** to deploy independent stacks even if others fail

## Setup Instructions

### 1. Configure AWS Credentials (OIDC - Recommended)

For secure authentication without long-lived credentials, use OpenID Connect (OIDC):

1. Create an IAM OIDC identity provider in AWS:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

2. Create an IAM role with a trust policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:zashirah/zashirah-infra:ref:refs/heads/main"
           }
         }
       }
     ]
   }
   ```
   
   **Security Note:** This trust policy restricts access to the `main` branch only. For additional security:
   - To allow pull requests: `"repo:zashirah/zashirah-infra:pull_request"`
   - To allow specific branches: `"repo:zashirah/zashirah-infra:ref:refs/heads/BRANCH_NAME"`
   - To allow all branches (less secure): `"repo:zashirah/zashirah-infra:*"`

3. Attach necessary permissions to the role (e.g., CloudFormationFullAccess, IAMFullAccess)

4. Add the role ARN as a repository secret:
   - Name: `AWS_ROLE_ARN`
   - Value: `arn:aws:iam::ACCOUNT_ID:role/github-actions-role`

5. Optionally, set the AWS region:
   - Name: `AWS_REGION`
   - Value: `us-east-1` (or your preferred region)

### 2. Using Long-Lived Credentials (Alternative)

If OIDC is not available, use long-lived access keys:

1. Create an IAM user with necessary permissions
2. Generate access keys
3. Add as repository secrets:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_REGION` (optional, defaults to us-east-1)

4. Modify the workflow file `.github/workflows/deploy-cloudformation.yml`:
   
   **Replace these lines (around line 82):**
   ```yaml
   - name: Configure AWS credentials
     uses: aws-actions/configure-aws-credentials@v4
     with:
       role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
       aws-region: ${{ secrets.AWS_REGION || 'us-east-1' }}
   ```
   
   **With:**
   ```yaml
   - name: Configure AWS credentials
     uses: aws-actions/configure-aws-credentials@v4
     with:
       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
       aws-region: ${{ secrets.AWS_REGION || 'us-east-1' }}
   ```

**Security Warning:** Long-lived credentials are less secure than OIDC. Rotate keys regularly and use OIDC when possible.

## Stack Configuration

Stacks are defined in `.github/cloudformation-stacks.yaml`:

```yaml
stacks:
  - name: stack-name              # CloudFormation stack name
    template: template-file.yaml   # Path to template file
    capabilities:                  # Optional: required capabilities
      - CAPABILITY_IAM
      - CAPABILITY_NAMED_IAM
    parameters:                    # Optional: stack parameters
      ParameterKey: ParameterValue
    description: Stack description # Optional: for documentation
```

### Example Configuration

```yaml
stacks:
  - name: zashirah-iam-developer
    template: iam-developer-setup.yaml
    capabilities:
      - CAPABILITY_NAMED_IAM
    parameters:
      DeveloperUsername: developer
    description: IAM Developer User and Role with MFA
  
  - name: zashirah-vpc
    template: vpc-stack.yaml
    parameters:
      VpcCIDR: 10.0.0.0/16
    description: VPC and networking infrastructure
```

## Workflow Triggers

### Automatic Deployment

The workflow automatically runs when:
- Changes are pushed to the `main` branch
- Pull requests are opened or updated targeting `main`
- Changes affect YAML files or workflow configuration

### Manual Deployment

Use the "Actions" tab in GitHub:
1. Select "Deploy CloudFormation Stacks" workflow
2. Click "Run workflow"
3. Optionally specify a specific stack name to deploy
4. Click "Run workflow" button

## Deployment Process

1. **Prepare**: Parse configuration and identify stacks to deploy
2. **Validate**: Check CloudFormation template syntax
3. **Check**: Determine if stack exists (create vs update)
4. **Deploy**: Execute CloudFormation deployment
5. **Report**: Display outputs and comment on PR (if applicable)

## Stack Outputs

After successful deployment, the workflow:
- Displays stack outputs in the GitHub Actions log
- Comments on pull requests with formatted output tables
- Includes deployment timestamp

Example PR comment:
```
✅ Stack Deployed: zashirah-iam-developer

Template: iam-developer-setup.yaml

Outputs:
| Key | Value | Description |
|-----|-------|-------------|
| DeveloperUserArn | arn:aws:iam::123456789012:user/developer | ARN of the IAM developer user |
| AdminRoleArn | arn:aws:iam::123456789012:role/DeveloperAdminRole | ARN of the Administrator role |

---
Deployed at 2024-10-19T15:00:00.000Z
```

## Adding New Stacks

To add a new CloudFormation stack:

1. Create your CloudFormation template file in the repository root
2. Add an entry to `.github/cloudformation-stacks.yaml`
3. Commit and push changes
4. The workflow will automatically deploy the new stack

Example:
```yaml
- name: zashirah-s3-bucket
  template: s3-bucket.yaml
  parameters:
    BucketName: my-unique-bucket-name
  description: S3 bucket for static assets
```

## Troubleshooting

### Deployment Failures

If a deployment fails:
1. Check the GitHub Actions log for error messages
2. The workflow automatically displays recent failed stack events
3. Review the CloudFormation console in AWS for detailed error information

### Common Issues

**Authentication errors:**
- Verify `AWS_ROLE_ARN` secret is correctly set
- Check IAM role trust policy allows GitHub Actions
- Ensure OIDC provider is configured in AWS

**Insufficient permissions:**
- Review IAM role permissions
- Add necessary permissions for CloudFormation operations
- Add permissions for resource creation (e.g., IAM, EC2)

**Template validation errors:**
- Run local validation: `aws cloudformation validate-template --template-body file://template.yaml`
- Check template syntax and resource properties

**Parameter errors:**
- Verify parameter keys match template parameter names
- Check parameter values meet template constraints

## Security Best Practices

- Use OIDC instead of long-lived credentials
- Follow principle of least privilege for IAM permissions
- Review CloudFormation templates before merging PRs
- Use stack policies to prevent accidental resource deletion
- Rotate access keys regularly if using long-lived credentials
- Enable CloudTrail to audit CloudFormation actions

## Workflow Customization

### Deploy Only Specific Stacks

Use workflow dispatch with the `stack_name` input to deploy a single stack:
1. Go to Actions → Deploy CloudFormation Stacks
2. Click "Run workflow"
3. Enter the stack name (e.g., `zashirah-iam-developer`)
4. Click "Run workflow"

### Modify Deployment Strategy

Edit `.github/workflows/deploy-cloudformation.yml` to customize deployment behavior:

**Deploy stacks concurrently (around line 67):**
```yaml
strategy:
  matrix:
    stack: ${{ fromJson(needs.prepare.outputs.stacks) }}
  fail-fast: false
  max-parallel: 3  # Changed from 1 to deploy up to 3 stacks at once
```

**Stop all deployments on first failure (around line 67):**
```yaml
strategy:
  matrix:
    stack: ${{ fromJson(needs.prepare.outputs.stacks) }}
  fail-fast: true  # Changed from false
  max-parallel: 1
```

**Add stack-specific timeouts (add to deploy job):**
```yaml
timeout-minutes: 30  # Add this line after "runs-on: ubuntu-latest"
```

### Add Post-Deployment Actions

After the deployment step, add additional steps:
- Run integration tests
- Update documentation
- Notify team via Slack/email
- Tag releases

## Resources

- [AWS CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [AWS Configure Credentials Action](https://github.com/aws-actions/configure-aws-credentials)
- [OIDC Setup Guide](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
