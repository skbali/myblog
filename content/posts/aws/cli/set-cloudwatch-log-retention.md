+++
title = "Set Retention For CloudWatch Logs - CLI"
summary = "Set a retention period for CloudWatch logs using aws cli"
date = 2018-12-15T04:45:00Z   
tags = ["aws", "cli", "CloudWatch Logs"]
canonicalURL = "https://blog.skbali.com/2022/06/my-first-go-lambda-function/"
ShowReadingTime = true
draft = false
+++

Amazon provides us with CloudWatch logs to monitor, store and access log from various AWS services like, EC2 instances, Route 53, RDS, AWS CloudTrail, AWS VPC Flow Logs, Lambda and many others.

By default, CloudWatch Logs are kept indefinitely and never expire. We are allowed to set a retention period and at present it can be set to a period between 10 years and one day.

One of the big users of CloudWatch Logs is Lambda service. All logging statements from Lambda are written to CloudWatch Logs. As Lambda usage grows in an account, so will the amounts of logs in CloudWatch Logs.

All Lambda functions by default will create a Log Group inside CloudWatch logs matching their name. As an example, a `HellowWorld` Lambda function will create a Log Group `/aws/lambda/HelloWorld` with retention period set to `Never Expire`

In an AWS account with a lot of work going on in Serverless space, one could end up with many `log groups` which retain logs indefinitely.

Amazon has given us the ability to change the retention period therefore as an admin we should periodically review and set accordingly. However, in a multi-region account doing this weekly or daily can be a burden.

## Solution
We can write a script or code to take care of this problem for us. In this post I will focus on using a bash shell script to solve this, you could do this via a Lambda function as well if you wish.

### Script
```bash
#!/bin/bash

declare -r retention="30"
for L in $(aws logs describe-log-groups \
    --query 'logGroups[?!not_null(retentionInDays)] | [].logGroupName' \
    --output text)
do
   aws logs  put-retention-policy --log-group-name ${L} \
   --retention-in-days ${retention}
done
```

In the script above, the retention days is fixed, but this can be passed as an argument to it.

First, we query all log groups which do not have a retention period set. Then in a `for` loop we iterate over the `log groups` and set the desired retention period.

After you run this script, you should see the new retention set in the console for the log group.

![CloudWatch Logs](/aws/cli/cwl-logs.jpg)

## Further Reading And Improvements
- Pass or set the AWS Region to run this across regions in your account if you have more than one region active.
- Add logging and error handling to the script.
- Set up the script to run from `cron` or `Jenkins` to run periodically.
- [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
- [CloudWatch Log Groups and Streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)