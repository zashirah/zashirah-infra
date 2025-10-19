# zashirah-infra

Infrastructure as Code for the Zashirah project using AWS CloudFormation.

## IAM Developer Setup

The `iam-developer-setup.yaml` template provisions secure IAM resources for development access:

### Resources Created

- **IAM User** (`developer`): Basic user with ReadOnlyAccess and password change capabilities
- **Access Keys**: For programmatic access via AWS CLI/SDK
- **Admin Role** (`DeveloperAdminRole`): Role with AdministratorAccess requiring MFA
- **Assume Role Policy**: Allows the developer user to assume the admin role with MFA

### Security Features

- **MFA Requirement**: Role assumption requires multi-factor authentication
- **Principle of Least Privilege**: Base user has read-only access; elevated privileges require role assumption
- **Session Duration**: Admin role sessions limited to 12 hours maximum

### Deployment

1. Deploy the CloudFormation stack:
   ```bash
   aws cloudformation create-stack \
     --stack-name zashirah-iam-developer \
     --template-body file://iam-developer-setup.yaml \
     --capabilities CAPABILITY_NAMED_IAM \
     --parameters ParameterKey=DeveloperUsername,ParameterValue=developer
   ```

2. Retrieve the stack outputs (including AccessKeyId):
   ```bash
   aws cloudformation describe-stacks \
     --stack-name zashirah-iam-developer \
     --query 'Stacks[0].Outputs'
   ```

3. Get the SecretAccessKey:
   - The secret key is only available immediately after stack creation
   - If you miss capturing it, you'll need to create a new access key
   - To rotate keys: Navigate to IAM > Users > [your-username] > Security credentials
   - **Important**: Create the new key first, update your configuration, then delete the old key

4. Configure AWS CLI with the access key:
   ```bash
   aws configure --profile zashirah-dev
   ```

5. Enable MFA for the IAM user in the AWS Console

6. Assume the admin role with MFA (replace `<MFA-CODE>` with your current MFA token):
   ```bash
   aws sts assume-role \
     --role-arn <AdminRoleArn-from-outputs> \
     --role-session-name admin-session \
     --serial-number arn:aws:iam::<account-id>:mfa/developer \
     --token-code <MFA-CODE> \
     --profile zashirah-dev
   ```

### Cleanup

To delete the stack and all resources:
```bash
aws cloudformation delete-stack --stack-name zashirah-iam-developer
```

**Note**: Delete any access keys manually before deleting the stack for security.
