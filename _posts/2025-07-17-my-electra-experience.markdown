---
title: My Electra Experience
layout: post
date: 2025-07-17
description: What I learned from PwnedLab's Electra cyber-range. 
categories: [Blog, Training Reviews]
tags: [Cloud, AWS, Web Exploitation, CTF]
author: bailee
toc: true
---

## Electra - a PwnedLabs Cyber Range

I recently purchased the yearly membership on [PwnedLabs](https://pwnedlabs.io/dashboard) after participating in their ‘AWS Cloud Attack and Defense’ bootcamp. Over the months, I have slowly been finding interest in CTFs, attack simulations, and vulnerable machines/systems to attack. More than that, I’ve also been very interested in the AWS and Azure clouds - especially on the topic of incident response in the cloud. 

I am working on a separate review of the bootcamps’ content, but I wanted to specifically write about PwnedLab’s AWS cyber-range, **Electra**. 

![image.png](/assets/img/electra/image.png)
 

Like I mentioned, Electra (as well as two other cyber-ranges) require the pro-annual subscription. However, I think this price is well worth the knowledge I took away from this cyber range. 

Electra writeups are currently not permitted as the lab is still active and in use, however I am able to generally cover what subjects I learned and impart some lessons I took away from the experience. Overall, I think the journey of initial access to total exfiltration was smooth, logical, and satisfying. It felt exciting to enumerate more and more resources across the environment and put the pieces together on how to use those resources to gain additional access to yet more resources. I have joked that I felt like the conspiracy guy while working through Electra - but it really does feel like putting a puzzle together! 

![image.png](/assets/img/electra/image1.png)

## An Electra Summary

So, here’s a quick summary on what Electra is all about. You start with two IP addresses and a OpenVPN configuration file which will allow you access to the PwnedLabs environment. With that starting info, you’ll be able to work your way the cyber range by collecting 9 flags from different locations. The flags are asked for sequentially, but there were a couple times that I found them out of order, however I was always able to work my way a little backward. 

Throughout the range, you’ll exercise not just AWS attack tactics but also learn about some web exploitation. 

> Wait, web exploitation? In *my* AWS exploitation lab?

Yep, web exploitation! I was thinking the web attack stuff was a bit out of left field too, but after talking it over with some others, I think it makes sense now. See, in some real penetration test scenarios, you might not just be given AWS credentials out of the blue (though that can happen in some pentest scenarios). You may have to look for references to AWS from a website and take advantage of a lack of input sanitization, or an unprotected API endpoint. Or, you might need to guess for subdomains, files or directories that will have more information to use in an attack scenario. 

The web exploit I executed in Electra was pretty advanced, but I learned a lot struggling to get there. That’s one big recommendation I have for anyone considering doing Electra: feel comfortable spending time on outside research! There is no point in banging your head against a wall - do some slightly unrelated learning and things will start to click in your mind. I ended up using some resources on TryHackMe, PortSwigger Academy, and about three thousand Stack Overflow forum pages. 

Speaking of recommendations - let’s talk more about what I learned from this cyber range. 

## My Tips and Tricks

### Don’t be afraid to go down a couple rabbit holes.

Electra took me a little over two weeks to work through, where I was spending probably 4+ hours a day on it (sometimes far more on the days off - and this was the Fourth of July weekend). I spent *far more time* researching what vulnerabilities I was seeing in the range environment and how to use them, or learning about how to use different tools. I’ll repeat what I said above because I think it’s very important: 

**Get good at learning new stuff.**

See some interesting references in the source code? Try to figure out what it’s doing. Is the web application using a service you only know about vaguely? Find the documentation for the service. Not sure what the syntax is for AWS-Enumerator? Bookmark its GitHub page. Google and AI are your new best friends. 

> Here’s some resources I used that you might some value in as well:
> 
> - [TryHackMe](https://tryhackme.com/dashboard) Web Exploitation rooms
> - [PortSwigger Academy](https://www.notion.so/Electra-Review-2332f7e3828280e88a5ac3390ea8a9cb?pvs=21)
> - [CyberChef](https://gchq.github.io/CyberChef/)

### Document, document, document!!

Take the best notes of your life. If you get some interesting output, definitely take a screenshot or copy and paste it into your note-taking solution of choice. Write down the steps you take to exploit a vulnerability, or gain additional access, because you may need to re-do it at some point and you don’t want to have to reinvent the wheel once you’ve already figured it out. 

![image.png](/assets/img/electra/image2.png)

Also, when you’re enumerating access in AWS, write down everything you find, from IAM principals to Lambda functions to VPCs! Doing that will definitely come back to help you in the future. Taking notes on every single step is not only going to make Electra much easier, but it’s also good practice for real-world security jobs, whether you’re working in red or blue. 

I plan on writing a little post about my note-taking solution(s), but for now, I’ll just recommend solutions like Obsidian and Notion. Obsidian is great if you have slight OCD, and Notion is great if you want to be ✨aesthetic ✨. 

### Trust your tools, but verify

I love tools that automate some of that boring manual checking. Tools like [AWS-Enumerator](https://github.com/shabarkin/aws-enumerator), [Pacu](https://github.com/RhinoSecurityLabs/pacu), and [CloudFox](https://github.com/BishopFox/cloudfox) are awesome for automating out all of those long AWS CLI commands and formatting the information out in a nice and easy-to-read format! However, don’t rely so much on those tools that you end up missing or forgetting something. All of the tools are great but will sometimes overlook some details or make false assumptions, leading you down the wrong road. 

AWS has some great CLI documentation for its services like IAM, Lambda, EC2, S3…. etc…  [here](https://docs.aws.amazon.com/cli/). However, it’s also a good idea to put together an AWS CLI command cheat sheet for enumeration. There are some CLI commands you’ll end using a lot, such as: 

- `aws sts get-caller-identity`
- `aws iam list-user-attached-policies`
- `aws iam get-policy --policy-arn <arn>`
- `aws ec2 describe-instances --region <region>`

Having your own list, or finding a good one like this one on [Cybr](https://cybr.com/courses/introduction-to-aws-enumeration/lessons/cheat-sheet-iam-enumeration-cli-commands-2/) or this one by [Tech with Tyler](https://www.techwithtyler.dev/cloud-security/cli-cheat-sheet) to reference will be a lifesaver in this cyber range. 

## #WorthIt?

If you enjoy a good puzzle, like to take notes (or want to get better at taking notes), and want to learn about 100 new things - YES! I will tell anyone who will listen to my rantings and ravings that I learned more about AWS security (and even some web exploitation) in the last 2 weeks working through Electra than I have while falling asleep to an AI-generated video about AWS concepts. The fact that it’s hands-on is so fun, and like I’ve said, the feeling of accomplishment once you’ve finished it is just the best. 

From a real-world-relevance perspective, I think it does prepare very well for actual pentest scenarios. I’m very firmly rooted in the blue-team, incident-response side of security, but knowing the tactics attackers could use in an AWS environment only guides and informs my investigations. 

Curious? Give [PwnedLabs](https://pwnedlabs.io/dashboard) free labs a try and see what I mean.