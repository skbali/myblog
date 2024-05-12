+++
title = "Query AWS CloudTrail Logs - CLI"
summary = "Query AWS CloudTrail logs using AWS cli"
date = 2024-04-14T16:45:00Z   
tags = ["aws", "cli", "CloudTrail", "CloudWatch", "Logs"]
ShowReadingTime = true
draft = false
+++

AWS CloudTrail is an AWS service that helps you enable governance, compliance, and operational and risk auditing of your AWS account. 
Actions taken by a user, role, or an AWS service are recorded as events in CloudTrail.

Visibility into your AWS account activity is a key aspect of security and operational best practices.

You can use CloudTrail to view, search, download, archive, analyze, and respond to account activity across your AWS infrastructure. 
You can identify who or what took which action, what resources were acted upon, when the event occurred, and other details to help you analyze and respond to activity in your AWS account.

There are many ways to query CloudTrail logs, one can use AWS Athena, third party software, a combination of Glue crawlers and Redshift. 

In this post I am going to show some practical examples of querying CloudTrail Logs using the AWS CLI.

## Query Examples
The CLI has two different commands for querying the logs, the first is by running a query against CloudTrail and the second is by running a query against CloudWatch Logs.

There are some differences in which the commands operate and how many events are brought back. Both commands allow for pagination by providing a token.

See links provided at the end of this post to read up more on the two methods to query events.

## Simple Query
```bash
aws cloudtrail lookup-events \
    --start-time $(date -v "-30M" +%s) \
    --query 'Events[].CloudTrailEvent' \
    --output text | jq
    
aws logs filter-log-events --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -v "-30M" +%s000) | jq
```
The above queries are simple queries which look back `30 minutes` into the logs. Notice the format of the date passed to the `--start-time` in the two queries. 
Both of them should output similar content. The output from these queries is not that easy to read, so I use `jq` a lot to view the results.

Let us take a look at another example where we filter and select only certain events.

## Query with Filter
```bash
aws cloudtrail lookup-events \
    --start-time $(date -v "-10M" +%s) \
    --query 'Events[].CloudTrailEvent' \
    --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com \
    --output text | jq

{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:sts::123456789012:assumed-role/stop-ec2-go-lambda-role/stop-ec2-go",
    "accountId": "123456789012",
    "sessionContext": {
      "sessionIssuer": {
        "type": "Role",
        "arn": "arn:aws:iam::123456789012:role/stop-ec2-go-lambda-role",
        "accountId": "123456789012",
        "userName": "stop-ec2-go-lambda-role"
      },
      "webIdFederationData": {},
      "attributes": {
        "creationDate": "2024-04-14T22:03:48Z",
        "mfaAuthenticated": "false"
      }
    }
  },
  "eventTime": "2024-04-14T22:03:48Z",
  "eventSource": "kms.amazonaws.com",
  "eventName": "Decrypt",
  "awsRegion": "us-east-1",
  "userAgent": "aws-internal/3 aws-sdk-java/1.12.676 Linux/4.14.336-257.562.amzn2.x86_64 OpenJDK_64-Bit_Server_VM/17.0.10+9-LTS java/17.0.10 kotlin/1.6.21 vendor/Amazon.com_Inc. cfg/retry-mode/standard",
  "requestParameters": {
    "encryptionAlgorithm": "SYMMETRIC_DEFAULT",
    "encryptionContext": {
      "aws:lambda:FunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:stop-ec2-go"
    }
  },
  "responseElements": null,
  "readOnly": true,
  "resources": [
    {
      "accountId": "123456789012",
      "type": "AWS::KMS::Key",
      "ARN": "arn:aws:kms:us-east-1:123456789012:key/xxxx"
    }
  ],
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012",
  "eventCategory": "Management",
  "tlsDetails": {
    "tlsVersion": "TLSv1.3",
    "cipherSuite": "TLS_AES_256_GCM_SHA384",
    "clientProvidedHostHeader": "kms.us-east-1.amazonaws.com"
  }
}
```

In the above example, I added a `lookup-attribute` for CloudTrail `lookup-event` query and in the CloudWatch command the `filter-pattern` was used to limit results.

I prefer the `--filter-pattern` option as I find it easier to use conditions compared to the `--lookup-attributes`.

```bash
aws logs filter-log-events --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -v "-30M" +%s000) \
    --filter-pattern '{ $.eventSource != "kms.amazonaws.com" }' \
    --query 'events[].message' \
    --output text | jq
```

In the above query I used `Not Equal to`  expression `--filter-pattern '{ $.eventSource != "kms.amazonaws.com" }'` to filter out KMS events.

If you prefer to filter or select using `jq` then the following example shows exactly how to do that.
```bash
aws logs filter-log-events --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -v "-50M" +%s000) \
    --filter-pattern '{ $.eventSource = "kms.amazonaws.com" }' \
    --query 'events[].message' \
    --output text |\
    jq '.| select(.eventName!="LookupEvents")'
```
In the above query, I am using `jq select` to filter out events.

## query and List Events with counts
```bsh
aws logs filter-log-events --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -v "-30M" +%s000) \
    --filter-pattern '{ $.eventSource != "kms.amazonaws.com" }' \
    --query 'events[].message' --output text | jq '.eventName' | sort | uniq -c | sort -n
  
   1 "CreateLogStream"
   1 "DescribeInstances"
   1 "GetAccountPublicAccessBlock"
   1 "ListMultiRegionAccessPoints"
   7 "AssumeRole"
  10 "DescribeEventAggregates"
  21 "GetBucketAcl"
 105 "FilterLogEvents"
```

## Query to find events with errors
```bash
aws logs filter-log-events --log-group-name CloudTrail/DefaultLogGroup \
    --start-time $(date -v "-30M" +%s000) \
    --filter-pattern '{ $.eventSource != "kms.amazonaws.com" }' \
    --query 'events[].message' --output text | jq '.|select(.errorCode != null)'
```
The above query will only pick events if there is an errorCode in the event.

Some can argue that it is better to filter the query itself rather than use `jq`, but I find `jq` very useful and more versatile, so I prefer using it.

## Further Reading
- [AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)
- [AWS CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
- [AWS CLI filter-log-events](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/logs/filter-log-events.html) 
