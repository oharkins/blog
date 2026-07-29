---
title: "Setting Up AWS Pipeline Execution Role for GitHub Actions OIDC"
date: "2024-11-09"
description: "Step-by-step guide to creating and configuring an AWS IAM role for GitHub Actions OIDC authentication"
categories: ["AWS", "Security"]
tags: ["aws", "iam", "github-actions", "oidc", "security"]
author: "Odis Harkins"
---

## Creating the Pipeline Execution Role

To deploy your Hugo site to AWS S3 using GitHub Actions OIDC, you'll need to set up a specific IAM role. Here's a step-by-step guide to creating and configuring this role.

### Step 1: Configure the OIDC Provider

First, you need to create an OIDC provider in AWS IAM if you haven't already:

1. Navigate to the AWS IAM Console
2. Go to Identity Providers
3. Click "Add Provider"
4. Select "OpenID Connect"
5. For the Provider URL, enter: `https://token.actions.githubusercontent.com`
6. For the Audience, enter: `sts.amazonaws.com`
7. Click "Add provider"

### Step 2: Create the IAM Role

1. Go to IAM Roles in the AWS Console
2. Click "Create Role"
3. Select "Web Identity"
4. Choose the GitHub OIDC provider you just created
5. For the Audience, select `sts.amazonaws.com`
6. Add the following trust relationship:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "repo:<GITHUB-USERNAME>/<REPOSITORY-NAME>:*"
                }
            }
        }
    ]
}
```

Replace:
- `<YOUR-AWS-ACCOUNT-ID>` with your AWS account ID
- `<GITHUB-USERNAME>` with your GitHub username or organization name
- `<REPOSITORY-NAME>` with your repository name

### Step 3: Create and Attach the Required Policy

Create a new policy with the following permissions:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:GetBucketLocation"
            ],
            "Resource": [
                "arn:aws:s3:::<YOUR-BUCKET-NAME>",
                "arn:aws:s3:::<YOUR-BUCKET-NAME>/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudfront:CreateInvalidation",
                "cloudfront:GetInvalidation",
                "cloudfront:ListInvalidations"
            ],
            "Resource": [
                "arn:aws:cloudfront::<YOUR-AWS-ACCOUNT-ID>:distribution/<YOUR-DISTRIBUTION-ID>"
            ]
        }
    ]
}
```

Replace:
- `<YOUR-BUCKET-NAME>` with your S3 bucket name
- `<YOUR-AWS-ACCOUNT-ID>` with your AWS account ID
- `<YOUR-DISTRIBUTION-ID>` with your CloudFront distribution ID (if using CloudFront)

### Step 4: Get the Role ARN

After creating the role:

1. Go to the role in the IAM Console
2. Copy the Role ARN
3. It should look like: `arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<ROLE-NAME>`

### Step 5: Use the Role in Your GitHub Actions Workflow

Update your GitHub Actions workflow to use the role:

```yaml
      - name: Deploy Hugo to S3 and Cloudfront using OIDC
        uses: oharkins/hugo-to-s3-action-oidc@v0.0.2
        with:
          hugo-version: 0.136.5
          target: production
          pipeline-execution-role: "arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:role/<ROLE-NAME>"
          aws-region: us-west-2
```

## Security Considerations

When setting up the pipeline execution role:

1. **Principle of Least Privilege**: Only grant the permissions that are absolutely necessary for the deployment process.

2. **Repository Constraints**: Use the `StringLike` condition in the trust relationship to limit which repositories can assume the role.

3. **Tag Restrictions**: Consider adding conditions to restrict role assumption to specific tags or branches.

4. **Regular Auditing**: Periodically review the role permissions and trust relationships to ensure they remain appropriate.

## Troubleshooting Common Issues

### Unable to Assume Role

If you get an error about assuming the role:

1. Verify the trust relationship is correctly configured
2. Check that the GitHub repository name matches exactly
3. Ensure the OIDC provider is properly set up

### Permission Denied Errors

If you get S3 or CloudFront permission errors:

1. Verify the policy permissions match your S3 bucket name
2. Check if the CloudFront distribution ID is correct
3. Ensure all required actions are included in the policy

### Invalid Resource Errors

If you get invalid resource errors:

1. Double-check all ARNs in your policies
2. Verify your AWS account ID is correct
3. Ensure your S3 bucket name is correct

Always follow the principle of least privilege when setting up IAM roles and policies. Grant only the permissions that are absolutely necessary for your deployment process.