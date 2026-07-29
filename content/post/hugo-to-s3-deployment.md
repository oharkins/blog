---
title: "Deploy Hugo Sites to AWS S3 Using GitHub Actions OIDC"
date: "2024-11-02"
description: "A comprehensive guide to using the hugo-to-s3-action-oidc GitHub Action for automated Hugo deployments to AWS S3"
categories: ["DevOps", "Tutorials"]
tags: ["hugo", "aws", "github-actions", "cicd", "automation", "oidc"]
author: "Odis Harkins"
---

Looking to automate your Hugo website deployments to AWS S3? The `hugo-to-s3-action-oidc` GitHub Action makes this process seamless by handling both the build and deployment in a single step, all while using secure OIDC authentication.

## Prerequisites

Before you begin, ensure you have:

1. A Hugo website in a GitHub repository
2. An AWS S3 bucket set up for static website hosting
3. AWS configured with GitHub OIDC authentication
4. Appropriate IAM roles and permissions

For detailed instructions on setting up OIDC in AWS, refer to the [official GitHub documentation](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services).

For detailed instructions on setting up Pipeline Execution Role in AWS, refer to the [Pipeline Execution Role](/blog/pipeline-execution-role/).

## Basic Usage

Here's a basic example of how to use the action:

```yaml
name: Hugo Build and Deploy to S3
on:
  workflow_dispatch:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

permissions:
  id-token: write
  contents: read

jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Check out master
        uses: actions/checkout@master
      
      - name: Build and deploy
        uses: oharkins/hugo-to-s3-action-oidc@v0.0.2
        with:
          hugo-version: 0.136.5
          pipeline-execution-role: <your execution role>
          aws-region: us-west-2
```

## Advanced Configuration Options

### Custom Config File

If your Hugo site uses multiple configuration files, you can specify a custom config file path using the `config` input parameter:

```yaml
jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Check out master
        uses: actions/checkout@master
      
      - name: Build and deploy
        uses: oharkins/hugo-to-s3-action-oidc@v0.0.2
        with:
          hugo-version: 0.136.5
          config: path/config.toml
          pipeline-execution-role: <your execution role>
          aws-region: us-west-2
```

### Deployment Targets

The action supports multiple deployment targets, which is particularly useful when managing different environments (e.g., staging and production). You can specify the target using the `target` input parameter.

#### Workflow Configuration

```yaml
jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Check out master
        uses: actions/checkout@master
      
      - name: Build and deploy
        uses: oharkins/hugo-to-s3-action-oidc@v0.0.2
        with:
          hugo-version: 0.136.5
          target: production
          pipeline-execution-role: <your execution role>
          aws-region: us-west-2
```

#### Hugo Configuration

To support multiple deployment targets, configure them in your Hugo configuration file:

```yaml
baseURL: /
languageCode: en-us
title: Example Site

deployment:
  targets:
    - name: "staging"
      URL: "s3://stg.example.com?region=us-west-2"
    - name: "production"
      URL: "s3://example.com?region=us-west-2"
```

If no target is specified in your GitHub Action, the first deployment target in your Hugo configuration will be used by default.

## Input Parameters

The action accepts the following input parameters:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `hugo-version` | Yes | Version of Hugo to use for building the site |
| `pipeline-execution-role` | Yes | AWS IAM role ARN for OIDC authentication |
| `aws-region` | Yes | AWS region where your S3 bucket is located |
| `config` | No | Path to custom Hugo configuration file |
| `target` | No | Deployment target name as defined in Hugo config |

## Security Considerations

This action uses OIDC (OpenID Connect) for AWS authentication, which provides several security benefits:

1. No need to store AWS credentials as GitHub secrets
2. Short-lived credentials that are automatically rotated
3. Fine-grained control over which repositories can access your AWS resources

## Troubleshooting

If you encounter issues:

1. Verify your OIDC configuration in AWS
2. Check that your IAM role has the necessary permissions
3. Ensure your S3 bucket names match the URLs in your Hugo configuration
4. Review GitHub Actions logs for detailed error messages

## Conclusion

The `hugo-to-s3-action-oidc` GitHub Action simplifies the process of deploying Hugo sites to AWS S3 while maintaining security best practices through OIDC authentication. Whether you're managing a single site or multiple environments, this action provides the flexibility and features needed for automated deployments.