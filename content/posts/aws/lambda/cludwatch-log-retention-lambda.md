+++
title = "Stop Long Running EC2 Instances"
summary = "Lambda function to set a retention period for CloudWatch logs"
date = 2020-06-15T10:57:00Z   
tags = ["aws", "lambda", "python", "serverless", "EventBridge", "CLI", "CloudWatch Logs"]
canonicalURL = "https://blog.skbali.com/2020/06/set-retention-for-cloudwatch-logs-using-a-lambda-function/"
ShowReadingTime = true
draft = true
+++

In one of my earlier post, I demonstrated how one could set retention for [CloudWatch Logs using CLI](/posts/aws/cli/set-cloudwatch-log-retention) command that could be run from cron or a CI/CD server like Jenkins.

A user asked if the same could be done from the console or a function.

I am not sure how one would go about setting the retention period from the console for all Log Groups in one go, but a Lambda Function can certainly be used to accomplish this.

In this post, I am going to show how we could set retention for CloudWatch Logs using  a Lambda function. 

A CloudWatch Events rule will be used to trigger the Lambda Function execution and this can be configured to run periodically.

## Background
Amazon provides us with CloudWatch logs to monitor, store and access log from various AWS services like, EC2 instances, Route 53, RDS, AWS CloudTrail, AWS VPC Flow Logs, Lambda and many others.

By default, CloudWatch Logs are kept indefinitely and never expire. We are allowed to set a retention period and at present it can be set to a period between 10 years and one day.

One of the big users of CloudWatch Logs is Lambda service. All logging statements from Lambda are written to CloudWatch Logs. As Lambda usage grows in an account, so will the amounts of logs in CloudWatch Logs.

All Lambda functions by default will create a Log Group inside CloudWatch logs matching their name. As an example, a `HellowWorld` Lambda function will create a Log Group `/aws/lambda/HelloWorld` with retention period set to `Never Expire`

In an AWS account with a lot of work going on in Serverless space one could end up with many log group which retain logs indefinitely.

Amazon has given us the ability to change the retention period therefore as an admin we should periodically review and set accordingly. However, in a multi-region account doing this weekly or daily can be a burden.

## Prerequisites
Make sure you have installed the [Serverless Framework](https://serverless.com/framework/docs/) on your workstation.

Ensure that the workstation from where you are going to run the serverless framework has all the necessary permissions to publish your Lambda function.
The workstation should have a `IAM Role` or `Access Keys` configured correctly.

## Procedure

### Serverless Framework
To create the template for our work, I run the commands as shown below.

```bash
sls create --template aws-python3 --path LogRetention
cd LogRetention
```

### Lambda Function
Now it is time for us to set up our Lambda function using the Serverless Framework.

To create the template for our function, I run the commands as shown below.

```bash
sls create --template aws-python3 --path stop-aws-instances
cd stop-aws-instances
```

{{< gist 32babcfa3be01ea36398661c3bbc8a04 >}}

Here you will see two files `serverless.yml` and `handler.py`, I am going to update these files with our code as shown below.

The file `serverless.yml` contains information about our Lambda function and the IAM role that will be needed by the function.

Let us take a look at the `serverless.yml` file now.

### Provider Section
- The Cloud Provider and runtime
- The profile that will be used to publish this function. Instead of a profile you can set the AWS Access Keys on the shell.
- Tags associated with my Lambda Function.
- The AIM role that will be associated  with the Lambda Function when it runs.

### Function Section
- memorySize for the Lambda function.
- The timeout period for the Lambda Function. If function takes longer than 10 seconds, it will timeout.
- The CloudWatch events schedule that runs our function twice a day at 00:01 and 12:01 GMT.
- `enabled:true` implies that the rule is enabled and the function would run as specified above.

The Lambda function code is in `handler.py` and is quite self-explanatory.

There is a paginator that has been set up for `describe_log_group` which will get all the Log Groups in your account in the region in which the function is running.

I have set `PageSize` to 10, but you can increase this to a higher number.

If the retrieved log group already has `retentionInDays` key then I am ignoring it, otherwise I use `put_retention_policy` for the log group and set it to `7 days`.

The code listed here is very basic and meant to be a running example. You should enhance it for running in a production environment.

### Deployment
If you are ready to deploy the function, go ahead and run the command as shown next.

```bash
sls deploy
```

This will trigger the process of creating the CloudFormation scripts which in turn will create a Lambda Function, an IAM role for your function and a CloudWatch Events Rule.

If you make any changes to either of the two files, you can simply run `sls deploy` again to push the latest code up to your account.

### Testing
You can either wait for the CloudWatch Events rule to trigger your function, or you can go to the Lambda console and create a test event to manually run your function

The following image shows that the function ran and set the log retention period for log groups which did not have any retention period.
![CloudWatch Logs](/aws/lambda/cwl-logs.jpg)

### Remove Resources
If you no longer wish to keep this function, you can remove the resources that were created by running this command.

```bash
sls remove
```
This will remove all the resources that were created. Remember you are responsible for all charges incurred.
Leaving a Lambda function with a CloudWatch Event rule enabled, will cost you even if there are no servers to terminate.

You may also have to clean up your CloudWatch Logs and S3 buckets where this code was deployed.

## Improvements
- Review exception handling, adding comments etc. to make it more meaningful for you.
- Configure the Lambda Function to run in other regions.
- Pass Environment variables to the function to decide which logs groups to process.
- Use Environment variables to pass retention days to the function.

## Conclusion
The use of AWS Lambda functions can be a powerful tool for managing resources in your AWS account.

This approach also demonstrates the flexibility and control that cloud services offer, allowing us to automate tasks that would otherwise require manual intervention.

However, it's important to remember that while automation can be beneficial, it also requires careful management to avoid unintended consequences.

Always ensure that your functions are thoroughly tested and that you understand the implications of the actions they perform.

Let me know if you have any questions or suggestions by leaving a comment on the `github gist`.

## Further Reading
- [Serverless Docs](https://www.serverless.com/framework/docs/providers/aws/guide/intro/)
- [EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cwe-now-eb.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [AWS IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)