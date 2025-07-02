---
title: Prevent Breaches with AWS IAM Access Analyzer
layout: post
date: 2025-07-01
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, IAM]
author: bailee
toc: true
description: Set up IAM Access Analyzer, identify present issues, and remediate them.

---
Lab Platform: 
 : [PwnedLabs](https://pwnedlabs.io/labs/prevent-breaches-with-aws-iam-access-analyzer)

---

## Tools, Commands, & Resources

### Tools Used
- AWS IAM Access Analyzer

### Commands

- `aws iam update-login-profile --user-name [username] --password [password]`
    - Update credentials for console access, provide a username and password.

### References

- <https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html>

--- 


## Lab Walkthrough

I configured my AWS profile with the provided credentials and checked my role. 

```
┌──(kali㉿kali)-[~]
└─$ aws configure                             
AWS Access Key ID [****************B2X7]: AKIAVC******NJTZRNB
AWS Secret Access Key [****************pi/f]: 3yXql31h***************c3i51aY/U/19fX5
Default region name [us-east-1]: 
Default output format [json]: 
                                                                                                                
┌──(kali㉿kali)-[~]
└─$ aws sts get-caller-identity               
{
    "UserId": "AIDAVCO2MZSIFYIQIAYEE",
    "Account": "348887239824",
    "Arn": "arn:aws:iam::348887239824:user/security"
}
```

Because we’re using IAM Access Analyzer in the console, the only other command I will be using is `aws iam update-login-profile --user-name [user-name] --password [password]` . That will allow me to log into the console. 

```
┌──(kali㉿kali)-[~]
└─$ aws iam update-login-profile --user-name security --password [Hidden :) ]
```

I logged in and made sure my region was us-west-2, per the instructions. Then, I searched up **Access Analyzer** and navigated to that screen.

IAM Access Analyzer ([documentation link](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html)) helps to identify and analyze unintended external access to resources, identify unused access, validate IAM policies against best practices and custom standards, and generate policies based on CloudTrail activity.

Because one has not yet been created, I’m going to ‘Create analyser’. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image.png)

And I’m going to keep everything default in the creation process. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image1.png)

Now when I view it, I’m going to reload the page and this is what I see! 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image2.png)

Access Analyzer found that there are write permissions on the ‘huge-logistics-tmp’ S3 bucket from all principals, which means anyone, anywhere, could write to this S3 bucket. Actually, it can do more than that! 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image3.png)

The list of permissions just keep going on - anyone can do anything with this S3 bucket! 

When I click on the resource link, it shows a folder in the S3 bucket called ‘to_delete’. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image4.png)

And what’s in there? Just an accessKey file. Basically anyone can get those keys! 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image5.png)

This is where I would either escalate this issue and have this remediated, or remediate it myself if I had that say in my organization. Also, I would assume that these access keys had already been compromised and start looking for any malicious activity from that account. 

To remove it, I’ll go back to the view with the bucket (huge-logistics-tmp-24dd66b8edf9), then the tab ‘Permissions’, then scroll down to ‘Bucket Policy’, then ‘Edit’. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image6.png)

On the edit view, I’m going to replace the `“Principal”:”*”` to the IAM user accounts ‘security’ and ‘kate’ (though I’d want to be sure it isn’t compromised first). 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image7.png)

After rescanning the Access Analyzer finding, it’s no longer flagging! 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image8.png)

Let’s go back to the remaining 4 findings and take another one. There’s another one for the ‘huge-logistics-tmp’ bucket. This one allows list permissions for all principals. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image9.png)

Like last time, I’ll click on the link to the resource, go to the ‘Permissions’ tab, and scroll down to ‘Access control list (ACL)’, and click ‘Edit’. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image10.png)

I’m going to untick the ‘List’ next to everyone, which has the red caution icon. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image11.png)

Looks like that has been resolved! Let’s move onto the next finding. 

Here is another bucket (huge-logistics-custdata) with Read and List permissions for all principals. Per the instructions, user accounts ‘francesco’  and ‘security’ should be the only accounts with access to this info, so I’m going to edit the policy like in the first alert to only allow those 2 accounts those permissions. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image12.png)

That’s been resolved - 2 more! 

This next one is a little different. There is a role called ‘OrganizationAccountAccessRole’ that has sts:AssumeRole privileges for the external AWS account ID 036528129738. According to the instruction, internal documentation says that this account is a member of our AWS Organization. Since this is intended and not malicious, we can click ‘Archive’ and go to the next finding. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image13.png)

Last finding - an EC2 snapshot has Write and List permissions for external account ID 794929857501. According to internal documentation, this account belongs to a consultant working with the company. However, this EC2 snapshot contains sensitive creds, so access should only be given for as long as it is needed, and no longer. For now until access is needed again, lets remove access (and maybe rotate the credentials). 

I’m going to click on the resource URL, then click on the radio button next to the snapshot name (iac_backups_snapshot), then click on Actions in the top right corner > Snapshot settings > Modify Permissions. 

![image.png](/assets/img/preventBreachesIAMAccessAnalyzer/image14.png)

From here, I’m going to select the radio button next to the acount under ‘Shared accounts’, and select ‘Remove selected’ from the top right of that box. To save it, I select ‘Modify permissions’ in the bottom right hand corner. Now once I rescan, it is resolved and all of the findings are gone. 

Hint: I already found the flag, it was in one of the S3 buckets I secured ;)
