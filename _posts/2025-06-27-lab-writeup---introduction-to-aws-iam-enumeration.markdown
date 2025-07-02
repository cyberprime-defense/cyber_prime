---
title: Introduction to AWS IAM Enumeration
layout: post
date: 2025-06-27
categories: [Writeups, Cybr]
tags: [Cloud, AWS, IAM]
author: bailee
toc: true
description: Enumerate AWS IAM including users, groups, roles, and permissions.
---
Lab Platform: 
: [Cybr](https://cybr.com/hands-on-labs/lab/introduction-to-aws-iam-enumeration/)

Also thanks to Tyler Rambsey for his course [**Introduction to AWS Pentesting**](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwjOu_2akJOOAxUPFlkFHWWPJ3kQFnoECBoQAQ&url=https%3A%2F%2Facademy.simplycyber.io%2Fl%2Fpdp%2Fintroduction-to-aws-pentesting&usg=AOvVaw1UWsIF1neKU0Ktp-BfGS1c&opi=89978449), which introduced me to the Cybr Platform!

---

## Tools, Commands, & Resources

### Tools Used

- AWS CLI
- Pacu

### Commands Used

- `aws configure [--profile]`
    
    - The first command you use to input credentials. You can use the --profile modifier to specify a profile name, so you can use multiple accounts at the same time. 
    
- `aws sts get-caller-identity`
    
    - The ‘whois’ of AWS CLI. Will show information like UserId, AWS Account #, and ARN (Amazon Resource Name). This command is always allowed to run no matter what permissions the account has tied to it. 
    
- `aws iam get-user`
    
    - Similar to the above command, but will offer additional information like the date of account creation, and any tags attached to the account. 
    
- `aws iam get-account-authorization-details`
    
     - Get info about all IAM users, groups, roles, and policies in an AWS account, including their relationships to each other. 
    
- `aws iam list-users`
    
   - List users in an AWS account
    
- `aws iam list-access-keys [--user-name]`
    
   - List the access key information for a user, specify a --user-name if you want to see an account other than the one executing the command. 
    
- `aws iam list-user-policies --user-name <user-name>`
    
   - List policies directly attached to the specified user. 
    
- `aws iam get-user-policy --user-name <user-name> --policy-name <policy-name>`
    
   - Get details on a policy attached to a user, after specifying the user and policy name. 
    
- `aws iam list-groups`
    
   - List all groups in an AWS account 
    
- `aws iam list-groups-for-user --user-name <user-name>`
    
   - List all groups a user is in. 
    
- `aws iam get-group --group-name <group-name>`
    
   - Get details on an AWS group specified, including users in the group, group ARN, group ID, and creation date. 
    
- `aws iam list-group-policies --group-name <group-name>`
    
   - List all policies associated with a group name. 
    
- `aws iam get-group-policy --group-name <group-name> --policy-name <policy-name>`
    
   - Get details on a specific policy associated with a specific group. 
    
- `aws iam list-attached-group-policies --group-name <group-name>`
    
   - List all attached group policies to a group.
    
- `aws iam list-roles [--query “…”]`
    
   - List all roles in an AWS account. Use the --query flag to filter on values in the JSON. 
    
- `aws iam list-role-policies --role-name <role-name>`
    
   - List policies attached to a role in AWS
    
- `aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>`
    
   - Get the details on a policy attached to a role, specifying the role name and policy name. 
    

### References

- <https://www.strongdm.com/blog/aws-iam-roles-vs-policies>
- <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html>
- <https://aws.amazon.com/blogs/security/iam-policy-types-how-and-when-to-use-them/>
- <https://rhinosecuritylabs.com/aws/pacu-open-source-aws-exploitation-framework/>
- <https://github.com/RhinoSecurityLabs/pacu>
- <https://github.com/RhinoSecurityLabs/pacu/wiki/Module-Details>

---

## Lab Walkthrough

### Introduction 

We were given an access key and secret key and were instructed to do some enumeration on the provided account. First, I used the `aws configure` command, where I could input the credentials. This is sort of a login screen for CLI, but instead of a username and password, I’m providing the keys that were given to me. 

```
AWS Access Key ID [None]: AIDAQG....
AWS Secret Access Key [None]: M0AbFU7....
Default region name [None]: us-east-1
Default output format [None]: json
```

The region name (us-east-1) tells AWS I want to look at resources and services in that region, or global ones. Default output format means I want to see all output in JSON format. 

Next, I’m using the `aws sts get-caller-identity` CLI command. This is basically the ‘whois’ of AWS CLI. 

```
aws sts get-caller-identity
{
    "UserId": "AIDA2U2CARNHGESD6UCNL",
    "Account": "731893500750",
    "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Joel"
}
```
### Enumeration

From this output, I’ve learned my username (**introduction-to-aws-iam-enumeration-1749763374310-Joel**), and the AWS account ID the account belongs to. What else can I learn? 

This lab is all about learning enumeration in AWS. 

> “enumeration refers to the process of systematically gathering information about a target system or network, often used by attackers to gain a better understanding of its architecture, services, and potential vulnerabilities”
> 

An attacker may try to learn about everything they have access to with the new account access they have, so they can try to pivot, move laterally, or escalate privileges. 

A good way to do that is the command `aws iam get-account-authorization-details` , which would show the AWS accounts current configuration of IAM users, groups, and roles, as well as their relationships to each other. However, doesn’t look like that’s something I have permission to do. 

```
aws iam get-account-authorization-details

An error occurred (AccessDenied) when calling the GetAccountAuthorizationDetails operation: User: arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Joel is not authorized to perform: iam:GetAccountAuthorizationDetails on resource: * because no identity-based policy allows the iam:GetAccountAuthorizationDetails action
```

Well, can I list just the accounts, instead? 

```
aws iam list-users
{
    "Users": [
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Chris",
            "UserId": "AIDA2U2CARNHHKYYNMOWS",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Chris",
            "CreateDate": "2025-06-12T21:23:01+00:00"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Joel",
            "UserId": "AIDA2U2CARNHGESD6UCNL",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Joel",
            "CreateDate": "2025-06-12T21:22:59+00:00"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Mary",
            "UserId": "AIDA2U2CARNHDQIMVXBQG",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mary",
            "CreateDate": "2025-06-12T21:23:01+00:00"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Mike",
            "UserId": "AIDA2U2CARNHFP5NVXNSD",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mike",
            "CreateDate": "2025-06-12T21:22:58+00:00"
        }
    ]
}
```

Looks like I can. So there are 4 users, 1 of those being my account. There is Chris, Mary, and Mike left. What kinds of permissions do they have? 

```
aws iam list-user-policies --user-name introduction-to-aws-iam-enumeration-1749763374310-Joel
{
    "PolicyNames": [
        "AllowEnumerateRoles"
    ]
}

aws iam list-user-policies --user-name introduction-to-aws-iam-enumeration-1749763374310-Mike
{
    "PolicyNames": []
}

aws iam list-user-policies --user-name introduction-to-aws-iam-enumeration-1749763374310-Mary
{
    "PolicyNames": []
}
aws iam list-user-policies --user-name introduction-to-aws-iam-enumeration-1749763374310-Chris
{
    "PolicyNames": []
}
```

Of the 3 other users, our account is the only one that has the permission to ‘AllowEnumerateRoles’. This doesn’t mean that the other 3 don’t have any permissions, it just means they don’t have any inline policies - AKA, ones that are directly attached to the account. The 3 may be included in a group, which has policies attached to it - which just makes it far easier to manage than attaching policies to every single user. But back to ‘AllowEnumerateRoles’ - what does that allow us to do? 

```
aws iam get-user-policy --user-name introduction-to-aws-iam-enumeration-1749763374310-Joel --policy-name AllowEnumerateRoles
{
    "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Joel",
    "PolicyName": "AllowEnumerateRoles",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:GetRole",
                    "iam:GetRolePolicy",
                    "iam:ListRoles",
                    "iam:ListRolePolicies"
                ],
                "Resource": "*",
                "Effect": "Allow"
            }
        ]
    }
}
```

So this means we can use the commands ‘get-role’, ‘get-role-policy’, ‘list-roles’, and ‘list-role-policies’. These IAM actions are basically names for actions taken either in the console, CLI, or SDKs, so if we had access to the console, we would have permission to open up IAM and see this information on a screen. 

I’ll come back to that in a second, but can I list groups to see what permissions the others might have? 

```
aws iam list-groups
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Developers",
            "GroupId": "AGPA2U2CARNHG7WLHPOKV",
            "Arn": "arn:aws:iam::731893500750:group/introduction-to-aws-iam-enumeration-1749763374310-Developers",
            "CreateDate": "2025-06-12T21:22:59+00:00"
        },
        {
            "Path": "/",
            "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Infrastructure",
            "GroupId": "AGPA2U2CARNHPZVT6B3CA",
            "Arn": "arn:aws:iam::731893500750:group/introduction-to-aws-iam-enumeration-1749763374310-Infrastructure",
            "CreateDate": "2025-06-12T21:22:59+00:00"
        }
    ]
}
```

That is a yes. There are 2 groups - one for developers and one for infrastructure. However, I don’t know who’s in what group. There is a way to tell what group my account is in, though. 

```
aws iam list-groups-for-user --user-name introduction-to-aws-iam-enumeration-1749763374310-Joel
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Developers",
            "GroupId": "AGPA2U2CARNHG7WLHPOKV",
            "Arn": "arn:aws:iam::731893500750:group/introduction-to-aws-iam-enumeration-1749763374310-Developers",
            "CreateDate": "2025-06-12T21:22:59+00:00"
        }
    ]
}
```

So Joel is in the ‘Developers’ group. Let’s get some information on that group. 

```
aws iam get-group --group-name introduction-to-aws-iam-enumeration-1749763374310-Developers
{
    "Users": [
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Mike",
            "UserId": "AIDA2U2CARNHFP5NVXNSD",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mike",
            "CreateDate": "2025-06-12T21:22:58+00:00"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749763374310-Joel",
            "UserId": "AIDA2U2CARNHGESD6UCNL",
            "Arn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Joel",
            "CreateDate": "2025-06-12T21:22:59+00:00"
        }
    ],
    "Group": {
        "Path": "/",
        "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Developers",
        "GroupId": "AGPA2U2CARNHG7WLHPOKV",
        "Arn": "arn:aws:iam::731893500750:group/introduction-to-aws-iam-enumeration-1749763374310-Developers",
        "CreateDate": "2025-06-12T21:22:59+00:00"
    }
}
```

So besides our account, Mike is also in this group. What permissions does Joel (do we) have as part of this group? 

```
aws iam list-group-policies --group-name introduction-to-aws-iam-enumeration-1749763374310-Developers
{
    "PolicyNames": [
        "introduction-to-aws-iam-enumeration-1749763374310-devs-policy"
    ]
}
```

Okay, so we have a policy name. Let’s look it up. 

```
aws iam get-group-policy --group-name introduction-to-aws-iam-enumeration-1749763374310-Developers --policy-name introduction-to-aws-iam-enumeration-1749763374310-devs-policy
{
    "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Developers",
    "PolicyName": "introduction-to-aws-iam-enumeration-1749763374310-devs-policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:ListAccessKeys"
                ],
                "Resource": [
                    "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Joel",
                    "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mike"
                ],
                "Effect": "Allow"
            },
            {
                "Action": [
                    "iam:ListGroupPolicies",
                    "iam:ListAttachedPolicies",
                    "iam:ListPolicyVersions",
                    "iam:ListUserPolicies",
                    "iam:ListAttachedUserPolicies",
                    "iam:ListUsers",
                    "iam:ListGroups",
                    "iam:ListGroupsForUser",
                    "iam:GetPolicy",
                    "iam:GetPolicyVersion",
                    "iam:GetUser",
                    "iam:GetUserPolicy",
                    "iam:GetGroupPolicy",
                    "iam:GetGroup",
                    "iam:ListAttachedGroupPolicies"
                ],
                "Resource": "*",
                "Effect": "Allow"
            }
        ]
    }
}
```

Well, this makes a lot of sense. Resources Joel and Mike (the resources being accounts, in this case), are allowed to run all of the IAM actions on resource * (so everything and everywhere) in this AWS account. That explains why I was able to run all those commands in the first place. Well, since we’re allowed to, lets look at the ‘infrastructure’ group and see what they’re allowed to do! 

```
aws iam list-group-policies --group-name introduction-to-aws-iam-enumeration-1749763374310-Infrastructure
{
    "PolicyNames": [
        "introduction-to-aws-iam-enumeration-1749763374310-infra-policy"
    ]
}

aws iam get-group-policy --group-name introduction-to-aws-iam-enumeration-1749763374310-Infrastructure --policy-name introduction-to-aws-iam-enumeration-1749763374310-infra-policy
{
    "GroupName": "introduction-to-aws-iam-enumeration-1749763374310-Infrastructure",
    "PolicyName": "introduction-to-aws-iam-enumeration-1749763374310-infra-policy",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "iam:ListAccessKeys"
                ],
                "Resource": [
                    "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mary",
                    "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Chris"
                ],
                "Effect": "Allow"
            },
            {
                "Action": [
                    "iam:ListGroupPolicies",
                    "iam:ListPolicies",
                    "iam:ListPolicyVersions",
                    "iam:ListUserPolicies",
                    "iam:ListUsers",
                    "iam:ListGroups",
                    "iam:ListGroupsForUser",
                    "iam:GetPolicy",
                    "iam:GetPolicyVersion",
                    "iam:GetUser",
                    "iam:GetUserPolicy",
                    "iam:GetGroupPolicy"
                ],
                "Resource": "*",
                "Effect": "Allow"
            }
        ]
    }
}
```

This time, I’m going to check if there are any attached group policies. The way that these policies are different from the ones we saw above is that these are specific permissions that are attached individually to a group - so kind of like an inline policy, but for a group rather than an individual account. 

```
aws iam list-attached-group-policies --group-name introduction-to-aws-iam-enumeration-1749763374310-Infrastructure
{
    "AttachedPolicies": [
        {
            "PolicyName": "AWSCloudFormationReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AWSCloudFormationReadOnlyAccess"
        },
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

Looks like this group has 2 policies the other group doesn’t - ‘AWSCloudFormationReadOnlyAccess’ and ‘AmazonS3ReadOnlyAccess’. 

Moving on, let’s look at roles now. 

```
 aws iam list-roles
{
    "Roles": [
        {
            "Path": "/aws-reserved/sso.amazonaws.com/",
            "RoleName": "AWSReservedSSO_LabAdministrationAccess_29d84a00a0a0b815",
            "RoleId": "AROA2U2CARNHMALH5QSZ7",
            "Arn": "arn:aws:iam::731893500750:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LabAdministrationAccess_29d84a00a0a0b815",
            "CreateDate": "2024-07-31T17:48:08+00:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": "arn:aws:iam::731893500750:saml-provider/AWSSSO_9a01845db5f147b8_DO_NOT_DELETE"
                        },
                        "Action": [
                            "sts:AssumeRoleWithSAML",
                            "sts:TagSession"
                        ],
                        "Condition": {
                            "StringEquals": {
                                "SAML:aud": "https://signin.aws.amazon.com/saml"
                            }
                        }
                    }
                ]
            },
            "Description": "Administrator access for lab accounts",
            "MaxSessionDuration": 43200
        },
        {
            "Path": "/aws-reserved/sso.amazonaws.com/",
            "RoleName": "AWSReservedSSO_LabReadOnlyAccess_853ca0e509906d62",
            "RoleId": "AROA2U2CARNHOWV3AL5OF",
            "Arn": "arn:aws:iam::731893500750:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LabReadOnlyAccess_853ca0e509906d62",
            
  ...
```

There are a LOT of roles here. This isn’t abnormal - most organization will have a ton of roles. We need a way to search and filter, and the --query flag will allow us to do that. 

```
aws iam list-roles --query "Roles[?RoleName=='SupportRole']"
[
    {
        "Path": "/",
        "RoleName": "SupportRole",
        "RoleId": "AROA2U2CARNHBHQSL2AR6",
        "Arn": "arn:aws:iam::731893500750:role/SupportRole",
        "CreateDate": "2025-06-12T21:23:16+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::731893500750:root"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "ArnEquals": {
                            "aws:PrincipalArn": "arn:aws:iam::731893500750:user/introduction-to-aws-iam-enumeration-1749763374310-Mary"
                        }
                    }
                }
            ]
        },
        "Description": "Assumable role for internal support",
        "MaxSessionDuration": 3600
    }
]
```

I was given the name ‘SupportRole’ to search on for the Role name, and there it is! Looks like it is a role that Mary can assume, and the description is “Assumable role for internal support”. What all can this support role do? 

```
aws iam list-role-policies --role-name SupportRole
{
    "PolicyNames": [
        "AllowS3FullAccessForRole"
    ]
}

aws iam get-role-policy --role-name SupportRole --policy-name AllowS3FullAccessForRole
{
    "RoleName": "SupportRole",
    "PolicyName": "AllowS3FullAccessForRole",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "s3:*"
                ],
                "Resource": "*",
                "Effect": "Allow",
                "Sid": "AllowS3FullAccess"
            }
        ]
    }
}
```

So the role that Mary can assume can do *anything* with S3 *anywhere* in the AWS account. Sure wish we could assume that, but we are not Mary! That’s the end of the lab, but this is where I’d go next: 

- Are there any roles that Joel can assume? What can they do?
- If I could assume a role or get an account in that infrastructure group, what is in the S3 buckets?

Overall, this is a good lab that teaches the basics of IAM enumeration. 

## Extra Challenge

### Enumeration with Pacu

> Do everything in the lab, but do it over Pacu.
> 
> 1. Enumerate Users
> 2. Enumerate Policies
> 3. Enumerate Groups
> 4. Enumerate Roles

Looking through the Pacu documentation on Github ([link](https://github.com/RhinoSecurityLabs/pacu/wiki/Module-Details)), here are the modules I think could be useful: 

- iam__enum_roles
- iam__enum_users
- aws__enum_account
- iam__bruteforce_permissions
- iam__enum_permissions
- iam__enum_users_roles_policies_groups

That last one looks the most promising, but those are the ones I pulled from the documentation. Time to load up Pacu and see what we get! 

First things first, I need to connect to Pacu. 

```
┌──(kali㉿kali)-[~]
└─$ aws configure --profile pacuLabSC     
AWS Access Key ID [None]: AKIAQGYB**2PSS3RE47H
AWS Secret Access Key [None]: T3CkSraAy*****censored
Default region name [None]: 
Default output format [None]: json

┌──(kali㉿kali)-[~]
└─$ pacu --import-keys pacuLabSC
*** pacu welcome screen ***
```

I used a profile with AWS configure, then gave Pacu that profile. Now, I have connected and could list out all of the modules from the command line, if I wanted to. 

```
Pacu (pacuLabSC:No Keys Set) > ls

[Category: EXFIL]

  ebs__download_snapshots
  rds__explore_snapshots
  s3__download_bucket

[Category: EVADE]

  cloudtrail__download_event_history
  cloudwatch__download_logs
  detection__disruption
  detection__enum_services
  elb__enum_logging
  guardduty__whitelist_ip
  waf__enum

[Category: ENUM]

  acm__enum
  apigateway__enum
  aws__enum_account
  aws__enum_spend
  cloudformation__download_data
  codebuild__enum
  cognito__enum
  ds__enum
  dynamodb__enum
  ebs__enum_volumes_snapshots
  ec2__check_termination_protection
  ec2__download_userdata
  ec2__enum
  ecr__enum
  ecs__enum
  ecs__enum_task_def
  eks__enum
  elasticbeanstalk__enum
  glue__enum
  guardduty__list_accounts
  guardduty__list_findings
  iam__bruteforce_permissions
  iam__decode_accesskey_id
  iam__detect_honeytokens
  iam__enum_action_query
  iam__enum_permissions
  iam__enum_users_roles_policies_groups
  iam__get_credential_report
  inspector__get_reports
  lambda__enum
  lightsail__enum
  mq__enum
  organizations__enum
  rds__enum
  rds__enum_snapshots
  route53__enum
  secrets__enum
  sns__enum
  systemsmanager__download_parameters
  transfer_family__enum

[Category: RECON_UNAUTH]

  ebs__enum_snapshots_unauth
  iam__enum_roles
  iam__enum_users

[Category: ESCALATE]

  cfn__resource_injection
  iam__privesc_scan

[Category: EXPLOIT]

  api_gateway__create_api_keys
  cognito__attack
  ebs__explore_snapshots
  ec2__startup_shell_script
  ecs__backdoor_task_def
  eks__collect_tokens
  lightsail__download_ssh_keys
  lightsail__generate_ssh_keys
  lightsail__generate_temp_access
  systemsmanager__rce_ec2

[Category: LATERAL_MOVE]

  cloudtrail__csv_injection
  organizations__assume_role
  sns__subscribe
  vpc__enum_lateral_movement

[Category: PERSIST]

  ec2__backdoor_ec2_sec_groups
  iam__backdoor_assume_role
  iam__backdoor_users_keys
  iam__backdoor_users_password
  lambda__backdoor_new_roles
  lambda__backdoor_new_sec_groups
  lambda__backdoor_new_users
```

That’s a lot of modules! I think I want to try the module `iam__enum_permissions` to see if Pacu can find the same permissions that took so long to find in the original lab.  

```
Pacu (pacuLabSC:No Keys Set) > run iam__enum_permissions
  Running module iam__enum_permissions...
[iam__enum_permissions] Confirming permissions for users:
[iam__enum_permissions]   introduction-to-aws-iam-enumeration-1749787301102-Joel...
[iam__enum_permissions]     Confirmed Permissions for introduction-to-aws-iam-enumeration-1749787301102-Joel
[iam__enum_permissions] iam__enum_permissions completed.

[iam__enum_permissions] MODULE SUMMARY:

  20 Confirmed permissions for user: introduction-to-aws-iam-enumeration-1749787301102-Joel.
   0 Confirmed permissions for 0 role(s).
   0 Unconfirmed permissions for 0 user(s).
   0 Unconfirmed permissions for 0 role(s).
Type 'whoami' to see detailed list of permissions.
```

It found 20 roles for our user - Joel. What are those? Looks like I can use `whoami` to get those details. 

```
Pacu (pacuLabSC:No Keys Set) > whoami
{
  "UserName": "introduction-to-aws-iam-enumeration-1749787301102-Joel",
  "RoleName": null,
  "Arn": "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Joel",
  "AccountId": "014498641567",
  "UserId": "AIDAQGYBPW2PWCBIYY2NF",
  "Roles": null,
  "Groups": [
    {
      "Path": "/",
      "GroupName": "introduction-to-aws-iam-enumeration-1749787301102-Developers",
      "GroupId": "AGPAQGYBPW2PTBLEPCSTE",
      "Arn": "arn:aws:iam::014498641567:group/introduction-to-aws-iam-enumeration-1749787301102-Developers",
      "CreateDate": "Fri, 13 Jun 2025 04:01:45",
      "Policies": [
        {
          "PolicyName": "introduction-to-aws-iam-enumeration-1749787301102-devs-policy"
        }
      ]
    }
  ],
  "Policies": [
    {
      "PolicyName": "AllowEnumerateRoles"
    }
  ],
  "AccessKeyId": "AKIA**********PSS3RE47H",
  "SecretAccessKey": "T3CkSraAyLnjmIl5MWpC********************",
  "SessionToken": null,
  "KeyAlias": null,
  "PermissionsConfirmed": true,
  "Permissions": {
    "Allow": {
      "iam:listaccesskeys": {
        "Resources": [
          "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Joel",
          "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Mike"
        ]
      },
      "iam:listusers": {
        "Resources": [
          "*"
        ]
      },
      "iam:listattachedpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:getuserpolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:getgroup": {
        "Resources": [
          "*"
        ]
      },
      "iam:listattachedgrouppolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getpolicyversion": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroupsforuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:listattacheduserpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:listuserpolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:getuser": {
        "Resources": [
          "*"
        ]
      },
      "iam:getgrouppolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgroups": {
        "Resources": [
          "*"
        ]
      },
      "iam:listpolicyversions": {
        "Resources": [
          "*"
        ]
      },
      "iam:listgrouppolicies": {
        "Resources": [
          "*"
        ]
      },
      "iam:listroles": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrole": {
        "Resources": [
          "*"
        ]
      },
      "iam:getrolepolicy": {
        "Resources": [
          "*"
        ]
      },
      "iam:listrolepolicies": {
        "Resources": [
          "*"
        ]
      }
    },
    "Deny": {}
  }
}

```

Okay, this looks right to me. These are the roles from the group Joel shares with Mike (the developer one). I feel pretty good about that, so I’m going to try the `iam__enum_users_roles_policies_groups` module now. 

```
Pacu (pacuLabSC:No Keys Set) > run iam__enum_users_roles_policies_groups
  Running module iam__enum_users_roles_policies_groups...
[iam__enum_users_roles_policies_groups] Found 4 users
[iam__enum_users_roles_policies_groups] Found 14 roles
[iam__enum_users_roles_policies_groups] No Policies Found
[iam__enum_users_roles_policies_groups]   FAILURE: MISSING NEEDED PERMISSIONS
[iam__enum_users_roles_policies_groups] Found 2 groups
[iam__enum_users_roles_policies_groups] iam__enum_users_roles_policies_groups completed.

[iam__enum_users_roles_policies_groups] MODULE SUMMARY:

  4 Users Enumerated
  14 Roles Enumerated
  0 Policies Enumerated
  2 Groups Enumerated
  IAM resources saved in Pacu database.
```

Okay, so it looks like Pacu found 4 users, 14 roles, 2 groups, and no policies. I think that’s because we only have permissions for our group to list policies, so I might need to try something else. However, I’m going to see what it found - using the `data` command will print out everything Pacu has found so far. And there is a lot!

```
Pacu (pacuLabSC:No Keys Set) > data

Session data:
aws_keys: [
    <AWSKey: from_default-014498641567>
    <AWSKey: None>
]
id: 3
created: "2025-06-13 04:09:27.061262"
is_active: true
name: "pacuLabSC"
access_key_id: "AKIAQGYBPW2PSS3RE47H"
secret_access_key: "******" (Censored)
session_regions: [
    "all"
]
IAM: {
    "Users": [
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749787301102-Chris",
            "UserId": "AIDAQGYBPW2PS7HLQ57KB",
            "Arn": "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Chris",
            "CreateDate": "Fri, 13 Jun 2025 04:01:48"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749787301102-Joel",
            "UserId": "AIDAQGYBPW2PWCBIYY2NF",
            "Arn": "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Joel",
            "CreateDate": "Fri, 13 Jun 2025 04:01:45"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749787301102-Mary",
            "UserId": "AIDAQGYBPW2PVT6ZFDX4N",
            "Arn": "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Mary",
            "CreateDate": "Fri, 13 Jun 2025 04:01:47"
        },
        {
            "Path": "/",
            "UserName": "introduction-to-aws-iam-enumeration-1749787301102-Mike",
            "UserId": "AIDAQGYBPW2PTZ6R7VKEL",
            "Arn": "arn:aws:iam::014498641567:user/introduction-to-aws-iam-enumeration-1749787301102-Mike",
            "CreateDate": "Fri, 13 Jun 2025 04:01:45"
        }
    ],
```

Well, there are the 4 users we found before. That looks good! Moving on to the next part of the data output…

```
    "Roles": [
        {
            "Path": "/aws-reserved/sso.amazonaws.com/",
            "RoleName": "AWSReservedSSO_LabAdministrationAccess_b723c55366310365",
            "RoleId": "AROAQGYBPW2PXEOZUJNU4",
            "Arn": "arn:aws:iam::014498641567:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LabAdministrationAccess_b723c55366310365",
            "CreateDate": "Wed, 31 Jul 2024 17:57:46",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": "arn:aws:iam::014498641567:saml-provider/AWSSSO_7e35b8e6ec32d9c4_DO_NOT_DELETE"
                        },
                        "Action": [
                            "sts:AssumeRoleWithSAML",
                            "sts:TagSession"
                        ],
                        "Condition": {
                            "StringEquals": {
                                "SAML:aud": "https://signin.aws.amazon.com/saml"
                            }
                        }
                    }
                ]
            },
            "Description": "Administrator access for lab accounts",
            "MaxSessionDuration": 43200
        },
        {
            "Path": "/aws-reserved/sso.amazonaws.com/",
            "RoleName": "AWSReservedSSO_LabReadOnlyAccess_b1a0a6267b8bb4c5",
            "RoleId": "AROAQGYBPW2PW6ZTOMXWI",
            "Arn": "arn:aws:iam::014498641567:role/aws-reserved/sso.amazonaws.com/AWSReservedSSO_LabReadOnlyAccess_b1a0a6267b8bb4c5",
            "CreateDate": "Wed, 31 Jul 2024 17:57:46",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Federated": "arn:aws:iam::014498641567:saml-provider/AWSSSO_7e35b8e6ec32d9c4_DO_NOT_DELETE"
                        },
                        "Action": [
                            "sts:AssumeRoleWithSAML",
                            "sts:TagSession"
                        ],
                        "Condition": {
                            "StringEquals": {
                                "SAML:aud": "https://signin.aws.amazon.com/saml"
                            }
                        }
                    }
                ]
            },
            "Description": "Read only access for lab accounts",
            "MaxSessionDuration": 43200
        },
        {
            "Path": "/aws-service-role/guardduty.amazonaws.com/",
            "RoleName": "AWSServiceRoleForAmazonGuardDuty",
            "RoleId": "AROAQGYBPW2P72VUHZQSP",
            "Arn": "arn:aws:iam::014498641567:role/aws-service-role/guardduty.amazonaws.com/AWSServiceRoleForAmazonGuardDuty",
            "CreateDate": "Thu, 02 Jan 2025 05:02:27",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "guardduty.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "A service-linked role required for Amazon GuardDuty to access your resources. ",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/aws-service-role/malware-protection.guardduty.amazonaws.com/",
            "RoleName": "AWSServiceRoleForAmazonGuardDutyMalwareProtection",
            "RoleId": "AROAQGYBPW2PQSGKWZ654",
            "Arn": "arn:aws:iam::014498641567:role/aws-service-role/malware-protection.guardduty.amazonaws.com/AWSServiceRoleForAmazonGuardDutyMalwareProtection",
            "CreateDate": "Thu, 02 Jan 2025 05:02:27",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "malware-protection.guardduty.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "Description": "A service-linked role required for Amazon GuardDuty Malware Scan to access your resources. ",
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/aws-service-role/cloudtrail.amazonaws.com/",
            "RoleName": "AWSServiceRoleForCloudTrail",
            "RoleId": "AROAQGYBPW2P3NONFXZUO",
            "Arn": "arn:aws:iam::014498641567:role/aws-service-role/cloudtrail.amazonaws.com/AWSServiceRoleForCloudTrail",
            "CreateDate": "Wed, 31 Jul 2024 17:18:57",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "cloudtrail.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        },
        {
            "Path": "/aws-service-role/config.amazonaws.com/",
            "RoleName": "AWSServiceRoleForConfig",
            "RoleId": "AROAQGYBPW2P7J5MPRM5O",
            "Arn": "arn:aws:iam::014498641567:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig",
            "CreateDate": "Tue, 04 Mar 2025 19:54:00",
            "AssumeRolePolicyDocument": {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "config.amazonaws.com"
                        },
                        "Action": "sts:AssumeRole"
                    }
                ]
            },
            "MaxSessionDuration": 3600
        },
        ... etc
```
Okay, that’s a lot of roles, but we knew that from the original lab. So this looks right to me. 

```
 "Groups": [
        {
            "Path": "/",
            "GroupName": "introduction-to-aws-iam-enumeration-1749787301102-Developers",
            "GroupId": "AGPAQGYBPW2PTBLEPCSTE",
            "Arn": "arn:aws:iam::014498641567:group/introduction-to-aws-iam-enumeration-1749787301102-Developers",
            "CreateDate": "Fri, 13 Jun 2025 04:01:45"
        },
        {
            "Path": "/",
            "GroupName": "introduction-to-aws-iam-enumeration-1749787301102-Infrastructure",
            "GroupId": "AGPAQGYBPW2PWVBJKDSCX",
            "Arn": "arn:aws:iam::014498641567:group/introduction-to-aws-iam-enumeration-1749787301102-Infrastructure",
            "CreateDate": "Fri, 13 Jun 2025 04:01:45"
        }
    ]
}
```

And there are the 2 groups we knew about! 

Now, I’m wondering how to get the policies. The way I did it in the original lab was to…

… hi, 15-minute later Prime here. I already got the policies at the very beginning with the module `iam__enum_permissions`. Clearly, I’m still differentiating between the terms ‘policy’ and ‘permission’. 

All in all, Pacu made short work of that lab! Basically in two whole modules, I had the same information that took me ~20 CLI commands to do. I can see how this would be a useful tool.