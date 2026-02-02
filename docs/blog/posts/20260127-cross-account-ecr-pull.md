---
date: 2026-01-27
title: Cross Account ECR Image Pull in EKS Cluster
categories:
  - General
tags:
  - AWS
  - EKS
  - Docker
slug: cross-account-ecr-pull
---

This is for cases when you have one AWS Account (ex: 1234567890) with all [ECR](https://aws.amazon.com/ecr/) images and the workload is running in any AWS accounts like 1234567891, 1234567892, 1234567893, etc.

> [!CAUTION]
> This above line is not just one criterion that will help you decide if you want to pull from one account.
> The decision will also depend on things like:
> - if the accounts are in same region
> - if there is NAT Gateway involved in downloading the images
> - if ECR download is happening using VPC interface
> Give them a thought before finalising your final workflow. This will save you from unexpected AWS cost suddenly.
> Having said this, lets move on with rest of the article

For each of your accounts (1234567891, 1234567892, 1234567893, etc), add this registry policy in your account with ECR (1234567890):
```json
{
    "Effect": "Allow",
    "Principal": {
        "AWS": "arn:aws:iam::1234567891:root"
    },
    "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:ListImages"
    ],
    "Resource": "arn:aws:ecr:*:1234567890:repository/*"
}
```

use the policy for other accounts. This will be the policy document that needs to be added:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowECRAccessTo1234567891",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::1234567891:root"
      },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:*:1234567890:repository/*"
    },
    {
      "Sid": "AllowECRAccessTo1234567892",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::1234567892:root"
      },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:*:1234567890:repository/*"
    },
    {
      "Sid": "AllowECRAccessTo1234567892",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::1234567892:root"
      },
      "Action": [
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:ListImages"
      ],
      "Resource": "arn:aws:ecr:*:1234567890:repository/*"
    }
  ]
}
```

The equivalent AWS cli command to apply this policy will be, where `<aws-region-name>` is the name of the region where AWS account `1234567890` exists and `<file-name>` is the name of the file which was created for this JSON data:
```shell
aws --region <aws-region-name> ecr put-registry-policy --policy-text file://<file-name>.json
```
