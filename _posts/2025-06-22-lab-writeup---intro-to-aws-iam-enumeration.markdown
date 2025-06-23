---
title: Lab Writeup - Intro to AWS IAM Enumeration
layout: post
date: 2025-06-22
categories: [Writeups, PwnedLabs]
tags: [AWS, Cloud, CTF]
author: bailee
toc: true
description: You are tasked with enumerating the IAM user dev01 and mapping out any potentially compromised resources. Enumerate IAM roles, policies, and permissions.
---

**Lab Platform:** 
: [PwnedLabs](https://pwnedlabs.io/labs/intro-to-aws-iam-enumeration)

---

## Tools, Commands, & Resources 

### Tools Used
- AWS CLI

### Commands

- `aws sts get-caller-identity`
    - The ‘whoami’ of AWS CLI, shows ARN of the user, including AWS Account ID.
- `aws iam list-attached-user-policies –user-name <user-name>`
    - Get details about an inline policy
- `aws iam get-policy-version --policy-arn <policy-arn> --version-id <v#>`
    - Get details for a specific policy version since policies can have multiple versions
- `aws sts assume-role --role-arn <role-arn> --role-session-name <session-name>`
    - Assume an AWS STS role, supply the ARN for the role, and give the session an arbitrary name
- `aws s3 ls s3://<s3-name>`
    - List out the contents of a S3 bucket, and the directories within it.
- `aws s3 cp s3://<s3-name>`
    - Copy/download the specified file, use . at the end to ‘cat’ it to the screen

### References

- <https://cybr.com/courses/introduction-to-aws-enumeration/lessons/cheat-sheet-iam-enumeration-cli-commands-2/>
- <https://docs.aws.amazon.com/cli/latest/reference/s3/>

---

## Lab Walkthrough

### Enumeration

dev01, as found by `aws sts get-caller-identity` is our compromised user. To scope just what was compromised, we need to know details about the user, what groups they are in, and what IAM policies the user has (what all can it access/do?).

`aws iam list-attached-user-policies –user-name dev01`

The account has the following access policy:
- AmazonGuardDutyReadOnlyAccess
So the account has read only access to AWS GuardDuty alerts.

Additionally, the account has the inline policy `S3_Access`.
An inline policy differs from a managed policy because it’s for a specific & single IAM identity (user, group, role) with a 1-1 relationship between entity & policy.

Looking into the details of the GuardDuty alert, we see List* and Describe* actions.

![image.png](assets/img/introIAMEnumeration/image.png)

These commands can allow an attacker to enumerate resources in an account, so this could be an avenue to pivot investigation.

Now we’re viewing the policy attached to dev01 and finding what other resources are associated with this account that the account would have access to.

![image.png](assets/img/introIAMEnumeration/image1.png)

dev01 can get details about ‘BackendDev’ and ‘BackendDevPolicy’, so let’s look into the details surrounding those.

![image.png](assets/img/introIAMEnumeration/image2.png)

Welp, there it is. dev01 can use the permission `sts:AssumeRole` to take actions as BackendDev. This is a normal action for developers, it allows developers to take a role like this temporarily and use the elevated permissions it might have, without having those permissions all of the time. Luckily, we have the ability to see what BackendDev can do while the compromised dev01 was assuming it.

```bash
# aws iam get-policy-version --policy-arn arn:aws:iam::794929857501:policy/BackendDevPolicy --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "VisualEditor0",
                    "Effect": "Allow",
                    "Action": [
                        "ec2:DescribeInstances",
                        "secretsmanager:ListSecrets"
                    ],
                    "Resource": "*"
                },
                {
                    "Sid": "VisualEditor1",
                    "Effect": "Allow",
                    "Action": [
                        "secretsmanager:GetSecretValue",
                        "secretsmanager:DescribeSecret"
                    ],
                    "Resource": "arn:aws:secretsmanager:us-east-1:794929857501:secret:prod/Customers-QUhpZf"
                }
            ]
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2023-09-29T12:44:09+00:00"
    }
}
```

So BackendDev can Get and Describe secrets from Secrets Manager. Cool! (not). Specifically, for the resource `arn:aws:secretsmanager:us-east-1:794929857501:secret:prod/Customers-QUhpZf`. What’s that about?

Let’s also look at the inline policy `S3_Access` and see what it can do, since it’s an policy attached to dev01.

```bash
"Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListBucket",
                    "s3:GetObject"
                ],
                "Resource": [
                    "arn:aws:s3:::hl-dev-artifacts",
                    "arn:aws:s3:::hl-dev-artifacts/*"
                ]
            }
```

So it has access to List & Get objects in the S3 bucket “hl-dev-artifacts” and any object in that file path.

The key is hiding somewhere in the S3 bucket. I’m going to assume the BackendDev role to see if I can’t get in there.

```bash
# aws sts assume-role --role-arn arn:aws:iam::794929857501:role/BackendDev --role-session-name test1
```

```bash
# aws sts get-caller-identity
{
    "UserId": "AROA3SFMDAPO2RZ36QVN6:test1",
    "Account": "794929857501",
    "Arn": "arn:aws:sts::794929857501:assumed-role/BackendDev/test1"
}
```
### Exfiltration

I spent a long time figuring out these commands, but I learned to use the following command:

```bash
# aws s3 ls s3://hl-dev-artifacts
2023-10-01 14:39:53       1235 android-kotlin-extensions-tooling-232.9921.47.pom
2023-10-01 14:39:53     214036 android-project-system-gradle-models-232.9921.47-sources.jar
2023-10-01 14:38:05         32 flag.txt
```

Here’s a breakdown of that command:
- **aws s3** - using the AWS CLI and the S3 command-set to interact with S3 objects
- **ls** - short for ‘list’
- **s3://hl-dev-artifacts** - this is how s3 buckets are addressed in AWS, kind of like www.:// or ftp://. We learned the name of the bucket in the policy above.

To ‘cat’ the contents of the file, like in linux, you use the following command:
`# aws s3 cp s3://hl-dev-artifacts/flag.txt -`
‘cp’ stands for copy. The - at the end says to copy it to standard out (the command prompt window).

And that’s how you get the flag!