# Implementing OIDC Authentication for GitHub Actions with AWS IAM

This repository demonstrates how to set up OpenID Connect (OIDC) authentication for GitHub Actions using AWS Identity and Access Management (IAM) through Terraform. The OIDC setup allows GitHub Actions to securely assume roles in AWS without managing long-lived credentials, enhancing both security and ease of use.

To enable GitHub Actions to securely assume an AWS IAM role using OpenID Connect (OIDC), the following steps are taken in the Terraform configuration:



## Terraform Configuration Overview

### 1. Define the TLS Certificate Data Source: 
The TLS certificate for the OIDC provider is fetched using the `tls_certificate` data source.
```hcl
data "tls_certificate" "eks" {
url = "https://token.actions.githubusercontent.com"
}
```

### 2. Create the OIDC Provider in AWS:
An AWS IAM OpenID Connect Provider is created to allow GitHub Actions to authenticate using OIDC.
```hcl
resource "aws_iam_openid_connect_provider" "git" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
  url             = "https://token.actions.githubusercontent.com"
}
```

### 3. Define the IAM Policy Document:
A policy document is created to allow the `sts:AssumeRoleWithWebIdentity` action, which is necessary for GitHub Actions to assume the role.
```hcl
data "aws_iam_policy_document" "git_aws_oidc" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.git.url, "https://", "")}:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "${replace(aws_iam_openid_connect_provider.git.url, "https://", "")}:sub"
      values   = ["repo:amosegonmwan/terraform-module-call:*"]
    }

    principals {
      identifiers = [aws_iam_openid_connect_provider.git.arn]
      type        = "Federated"
    }
  }
}
```

### 4. Create the IAM Role:
An IAM role is created that GitHub Actions can assume using the OIDC provider.
```hcl
resource "aws_iam_role" "git_action" {
  assume_role_policy = data.aws_iam_policy_document.git_aws_oidc.json
  name               = "git-actions-oidc"
}
```

### 5. Attach a Policy to the IAM Role:
A policy is attached to the IAM role, granting it the necessary permissions.
```hcl
resource "aws_iam_policy" "git_action" {
  name = "git-actions-oidc"

  policy = jsonencode({
    Statement = [{
      Action   = ["*"]
      Effect   = "Allow"
      Resource = "*"
    }]
    Version = "2012-10-17"
  })
}
```

### 6. Attach the Policy to the IAM Role:
Finally, the policy is attached to the IAM role.
```hcl
resource "aws_iam_role_policy_attachment" "git_actions_oidc_attachment" {
  role       = aws_iam_role.git_action.name
  policy_arn = aws_iam_policy.git_action.arn
}
```

### 7. Output the IAM Role ARN:
The ARN of the IAM role is output for use in your GitHub Actions workflow.
```hcl
output "git_actions_oidc" {
  value = aws_iam_role.git_action.arn
}
```

### 8. Configure the AWS Provider:
The AWS provider is configured to operate in a specific region.
```hcl
provider "aws" {
  region = "us-west-2"
}
```

This setup allows secure interaction between GitHub Actions and AWS using OIDC, enabling the GitHub workflow to assume an IAM role with the necessary permissions.

