# AWS Credentials Setup Guide for DDA Notebooks

This guide explains how to configure AWS credentials to run the DDA Jupyter notebooks successfully.

## Overview

The DDA notebooks (`DDA_SageMaker_Model_Training_and_Compilation.ipynb` and `DDA_Greengrass_Component_Creator.ipynb`) require AWS credentials to:
- Access Amazon S3 for model artifacts storage
- Use Amazon SageMaker for model training and compilation
- Create AWS IoT Greengrass components
- Make other AWS API calls

## Common Error

If you see this error when running the notebooks:

```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

It means AWS credentials are not configured. Follow one of the methods below to fix this.

## Methods to Configure AWS Credentials

### Method 1: Run in SageMaker Environment (Recommended)

**Best for:** Training and compiling models in the cloud

Running notebooks in Amazon SageMaker Notebook Instance or SageMaker Studio automatically provides credentials through the execution role.

**Steps:**
1. Open AWS Console → Amazon SageMaker
2. Choose "Notebook instances" or "Studio"
3. Create a new notebook instance or Studio domain
4. Ensure the execution role has these permissions:
   - `AmazonSageMakerFullAccess`
   - S3 read/write access to your bucket
   - AWS IoT Greengrass permissions
5. Upload and run the notebooks in SageMaker

**No additional credential configuration needed!**

---

### Method 2: AWS CLI Configuration

**Best for:** Running notebooks locally or on EC2 instances

Install and configure the AWS CLI to store credentials on your machine.

**Steps:**

1. **Install AWS CLI** (if not already installed):
   ```bash
   # Using pip
   pip install awscli

   # Or using package manager (Ubuntu/Debian)
   sudo apt-get install awscli

   # Or using Homebrew (macOS)
   brew install awscli
   ```

2. **Configure credentials**:
   ```bash
   aws configure
   ```

3. **Enter your credentials when prompted**:
   ```
   AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID
   AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY
   Default region name [None]: us-east-1  # or your preferred region
   Default output format [None]: json
   ```

4. **Verify configuration**:
   ```bash
   aws sts get-caller-identity
   ```

**Files created:**
- `~/.aws/credentials` - Contains access keys
- `~/.aws/config` - Contains region and output format

---

### Method 3: Environment Variables

**Best for:** Temporary credentials or CI/CD pipelines

Set AWS credentials as environment variables in your shell or notebook.

**Option A: In Terminal (before starting Jupyter)**

```bash
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_ACCESS_KEY"
export AWS_DEFAULT_REGION="us-east-1"

# Then start Jupyter
jupyter notebook
```

**Option B: In Notebook Cell**

Add this cell at the beginning of your notebook:

```python
import os

os.environ['AWS_ACCESS_KEY_ID'] = 'YOUR_ACCESS_KEY_ID'
os.environ['AWS_SECRET_ACCESS_KEY'] = 'YOUR_SECRET_ACCESS_KEY'
os.environ['AWS_DEFAULT_REGION'] = 'us-east-1'
```

⚠️ **Security Warning:** Don't commit credentials to Git! Add a `.env` file to `.gitignore` if using this method.

---

### Method 4: IAM Role (EC2 Instance)

**Best for:** Running notebooks on EC2 instances

Attach an IAM role to your EC2 instance to provide automatic credentials.

**Steps:**

1. **Create IAM Role:**
   - Go to AWS Console → IAM → Roles
   - Create new role for EC2
   - Attach policies:
     - `AmazonSageMakerFullAccess`
     - `AmazonS3FullAccess` (or more restrictive bucket-specific policy)
     - Custom policy for Greengrass operations:
       ```json
       {
           "Version": "2012-10-17",
           "Statement": [
               {
                   "Effect": "Allow",
                   "Action": [
                       "greengrass:*",
                       "iot:*"
                   ],
                   "Resource": "*"
               }
           ]
       }
       ```

2. **Attach Role to EC2 Instance:**
   - Go to EC2 Console
   - Select your instance
   - Actions → Security → Modify IAM role
   - Select the role created above
   - Save

3. **Verify** (from instance):
   ```bash
   aws sts get-caller-identity
   ```

**No code changes needed!** boto3 automatically uses the instance role.

---

## Creating AWS Access Keys

If you need to create new access keys for Methods 2 or 3:

1. Go to AWS Console → IAM → Users
2. Select your user (or create a new user)
3. Go to "Security credentials" tab
4. Click "Create access key"
5. Choose use case: "Command Line Interface (CLI)"
6. Download and save the credentials securely
7. ⚠️ **Never share or commit these credentials**

## Required IAM Permissions

Your AWS credentials (user or role) should have these permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sagemaker:*",
                "s3:GetObject",
                "s3:PutObject",
                "s3:CreateBucket",
                "s3:ListBucket",
                "greengrass:*",
                "iot:*",
                "iam:PassRole"
            ],
            "Resource": "*"
        }
    ]
}
```

For production use, restrict resources to specific ARNs instead of `"*"`.

## Verifying Credentials

To verify your credentials are working, run this in a Python cell:

```python
import boto3

try:
    sts = boto3.client('sts')
    identity = sts.get_caller_identity()
    print("✅ Credentials are configured!")
    print(f"Account: {identity['Account']}")
    print(f"User ARN: {identity['Arn']}")
except Exception as e:
    print("❌ Credentials not configured!")
    print(f"Error: {e}")
```

## Troubleshooting

### "Unable to locate credentials" Error

**Cause:** No credentials configured or boto3 can't find them.

**Solutions:**
1. Check if `~/.aws/credentials` file exists
2. Verify environment variables are set: `echo $AWS_ACCESS_KEY_ID`
3. If on EC2, verify IAM role is attached
4. Try running `aws configure` again

### "InvalidAccessKeyId" Error

**Cause:** Access key is incorrect or deleted.

**Solutions:**
1. Verify the access key ID is correct
2. Create a new access key in IAM console
3. Update credentials file or environment variables

### "AccessDenied" Error

**Cause:** Credentials are valid but lack required permissions.

**Solutions:**
1. Check IAM policies attached to your user/role
2. Add necessary permissions (see Required IAM Permissions above)
3. Ensure you have `iam:PassRole` permission for SageMaker

### Region Issues

**Cause:** Default region not set or incorrect.

**Solutions:**
```bash
# Set default region
aws configure set region us-east-1

# Or in Python
import boto3
boto3.setup_default_session(region_name='us-east-1')
```

## Security Best Practices

1. **Never commit credentials** to version control
2. **Use IAM roles** when possible (Method 4)
3. **Rotate access keys** regularly
4. **Use least privilege** permissions
5. **Enable MFA** on your AWS account
6. **Use temporary credentials** when possible (AWS STS)
7. **Add `.aws/` to `.gitignore`**

## Additional Resources

- [AWS CLI Configuration Guide](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Boto3 Credentials Guide](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)
- [SageMaker Execution Roles](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html)
