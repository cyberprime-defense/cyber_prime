---
title: Identify IAM Breaches with CloudTrail and Athena
layout: post
date: 2025-07-12
categories: [Writeups, PwnedLabs]
tags: [Cloud, AWS, CloudTrail, Athena]
author: bailee
toc: true
description: Knowing that authentication details about the company was leaked on Pastebin, identify malicious IAM activity and any compromised IAM accounts, using AWS CloudTrail and Amazon Athena.
---
Lab Platform: 
: [PwnedLabs](https://pwnedlabs.io/labs/identify-iam-breaches-with-cloudtrail-and-athena)

---

## Tools & Commands

### Tools Used

- AWS CloudTrail
- AWS Athena
- AWS CLI

### Commands

- `aws athena start-query-execution --query-string "<query>" --result-configuration OutputLocation= <output-location>`
    - Start to run an Athena query within --query-string, then choose where to output it.
- `aws athena get-query-results --query-execution-id addb8d45-6285-4c42-985a-47395fd0e6e1 --query 'ResultSet.Rows[].Data[].VarCharValue' --output text > results.txt`
    - Get the results from an Athena query, choose how to output it, and into what file (optional).

---

## Lab Walkthrough

### Introduction
According to CloudTrail history, we saw that a LOT of ConsoleLogin events. A CloudTrail event looks like this one: 

```
{
    "eventVersion": "1.08",
    "userIdentity": {
        "type": "IAMUser",
        "accountId": "104506445608",
        "accessKeyId": "",
        "userName": "HIDDEN_DUE_TO_SECURITY_REASONS"
    },
    "eventTime": "2023-08-30T21:50:02Z",
    "eventSource": "signin.amazonaws.com",
    "eventName": "ConsoleLogin",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "195.70.73.130",
    "userAgent": "Go-http-client/1.1",
    "errorMessage": "No username found in supplied account",
    "requestParameters": null,
    "responseElements": {
        "ConsoleLogin": "Failure"
    },
    "additionalEventData": {
        "LoginTo": "https://console.aws.amazon.com",
        "MobileVersion": "No",
        "MFAUsed": "No"
    },
    "eventID": "02382ed1-cb63-4e81-b91b-cd136fd29570",
    "readOnly": false,
    "eventType": "AwsConsoleSignIn",
    "managementEvent": true,
    "recipientAccountId": "104506445608",
    "eventCategory": "Management",
    "tlsDetails": {
        "tlsVersion": "TLSv1.3",
        "cipherSuite": "TLS_AES_128_GCM_SHA256",
        "clientProvidedHostHeader": "signin.aws.amazon.com"
    }
}
```

So the problem is that there is a ton of these ‘AwsConsoleSignIn’ alerts that have the response “Failure” to ConsoleLogin. This suggests a brute force attack, which would make sense with the context that a bunch of creds were leaked publicly. However, there are so many logs to sort through, that we need some sort of tool to parse and filter through all of them. 

Introducing Athena - Amazon’s data query service. It uses SQL language to query the CloudTrail logs (which are in a bucket). Luckily, SQL queries read a lot like English. Let’s try one: (Note: I’m on the console right now so I’m only posting my queries, not the results, which are difficult to format.)

### Amazon Athena Query Service

```
SELECT count(*)
FROM cloudtrail_logs_aws_cloudtrail_logs_104506445608_4e45885e
WHERE eventname = 'ConsoleLogin'
AND eventTime LIKE '%2023-08-30%'
```

This query looks for CloudTrail logs at the specified locator, where the event name is ‘ConsoleLogin’ and the event happened on 8/30/2023. It brings back 1745 results… that’s a lot. How many of those are successful? Let’s add onto that query: 

```
SELECT * 
FROM cloudtrail_logs_aws_cloudtrail_logs_104506445608_4e45885e
WHERE eventname = 'ConsoleLogin'
AND eventTime LIKE '%2023-08-30%'
AND responseelements LIKE '%Success%'
```

So we’re selecting Cloudtrail logs where the event name is ‘ConsoleLogin’ and the event time is on 8/30/2023, and ResponseElements include a “Success” message in there. I clicked ‘run’ and got 2 results back - on accounts ‘root’ and ‘pfisher’. 

This searching can also be done from the CLI. I’ve been given an access key and secret access key, so I’ll configure my AWS CLI with those. 

```
┌──(kali㉿kali)-[~]
└─$ aws athena start-query-execution --query-string "SELECT useridentity, sourceipaddress FROM cloudtrail_logs_aws_cloudtrail_logs_104506445608_4e45885e WHERE eventname = 'ConsoleLogin' AND eventTime LIKE '%2023-08-30%' AND responseelements LIKE '%Success%'" --result-configuration OutputLocation="s3://aws-athena-query-results-104506445608-us-east-1/Unsaved/"
{
    "QueryExecutionId": "addb8d45-6285-4c42-985a-47395fd0e6e1"
}

┌──(kali㉿kali)-[~]
└─$ aws athena get-query-results --query-execution-id addb8d45-6285-4c42-985a-47395fd0e6e1 --query 'ResultSet.Rows[].Data[].VarCharValue' --output text > results.txt 

┌──(kali㉿kali)-[~]
└─$ cat results.txt       
useridentity    sourceipaddress {type=Root, principalid=104506445608, arn=arn:aws:iam::104506445608:root, accountid=104506445608, invokedby=null, accesskeyid=, username=null, sessioncontext=null}    195.70.73.130   {type=IAMUser, principalid=AIDARQVIRZ4UOASQZJJ4W, arn=arn:aws:iam::104506445608:user/pfisher, accountid=104506445608, invokedby=null, accesskeyid=null, username=pfisher, sessioncontext=null}        195.70.73.130
```

So it looks like user ‘pfisher’ had a successful login attempt after all those failed ones, from the IP address 195.70.73.130. How do I know that that IP was the cause of the failed ones too? Because I ran a Athena query that removed the “responseelements = * success*” portion of the query, which showed that the logins (all of the logins, failed or not) were from the following IP and user agent: 

```
 IP Address      User Agent
 195.70.73.130   Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36
 195.70.73.130   Go-http-client/1.1
```

### Editing the Query

Sweet, so the way to get the flag is to find any additional user agent(s) with a hash for a user agent. That login attempt would happen on a day that wasn’t the 30th, so I’d need to try other days. I could do one at a time, but I’d rather group them all at once. To do that, I modified the query to do this: 

```
┌──(kali㉿kali)-[~]
└─$ aws athena start-query-execution --query-string "SELECT sourceipaddress, useragent FROM cloudtrail_logs_aws_cloudtrail_logs_104506445608_4e45885e WHERE eventname = 'ConsoleLogin' **AND eventTime > '2023-01-01'"** --result-configuration OutputLocation="s3://aws-athena-query-results-104506445608-us-east-1/Unsaved/"
{
    "QueryExecutionId": "fa1576d5-7e0b-4122-91a4-73b1f287a0ae"
}
```

Basically, the > means any eventTime AFTER 2023-01-01, which would definitely get most days. This is probably overkill. But I found this: 

```
 IP Address      User Agent
 195.70.73.130   0646679eca96f84*****hasFlag****b18d677d3
 195.70.73.130   Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36
 195.70.73.130   Go-http-client/1.1
```
