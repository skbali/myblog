+++
title = "Set Retention for CloudWatch Logs - Golang"
summary = "Golang Lambda function to set a retention period for CloudWatch logs"
date = 2024-03-30T16:57:00Z   
tags = ["aws", "lambda", "Go", "terraform", "EventBridge", "CLI", "CloudWatch Logs"]
ShowReadingTime = true
draft = false
+++

I have been converting my python Lambdas to Go and using Terraform to deploy them.

This post is about converting the [CloudWatch Log Retention](/posts/aws/lambda/cloudwatch-log-retention-lambda/) Lambda to Go and deploying it using Terraform.

In one of my earlier post, I demonstrated how one could set retention for [CloudWatch Logs using CLI](/posts/aws/cli/set-cloudwatch-log-retention) command that could be run from cron or a CI/CD server like Jenkins.

In this post, I am going to show how we could set retention for CloudWatch Logs using a Lambda function written in Goland and deployed using Terraform. 

A CloudWatch Events rule will be used to trigger the Lambda Function execution and this can be configured to run periodically.

## Background
Amazon provides us with CloudWatch logs to monitor, store and access log from various AWS services like, EC2 instances, Route 53, RDS, AWS CloudTrail, AWS VPC Flow Logs, Lambda and many others.

By default, CloudWatch Logs are kept indefinitely and never expire. We are allowed to set a retention period and at present it can be set to a period between 10 years and one day.

One of the big users of CloudWatch Logs is Lambda service. All logging statements from Lambda are written to CloudWatch Logs. As Lambda usage grows in an account, so will the amounts of logs in CloudWatch Logs.

All Lambda functions by default will create a Log Group inside CloudWatch logs matching their name. As an example, a `HellowWorld` Lambda function will create a Log Group `/aws/lambda/HelloWorld` with retention period set to `Never Expire`

In an AWS account with a lot of work going on in Serverless space one could end up with many log group which retain logs indefinitely.

Amazon has given us the ability to change the retention period therefore as an admin we should periodically review and set accordingly. However, in a multi-region account doing this weekly or daily can be a burden.

## Prerequisites
Download and Install [Terraform](https://www.terraform.io/downloads).

Install [Go](https://go.dev/doc/install) on your workstation.

The workstation should have a `IAM Role` or `Access Keys` configured correctly.

## Procedure

### Github Link
You can download the code for this post from my [github repository](https://github.com/skbali/cloudwatch-log-retention).

### Lambda Function
Now it is time for us to set up our Lambda function using Terraform.

```go
		result, err := client.DescribeLogGroups(context.TODO(), &cloudwatchlogs.DescribeLogGroupsInput{
			LogGroupNamePrefix: aws.String("/aws/lambda/"),
			Limit:              aws.Int32(maxResults),
			NextToken:          nextToken,
		})
        if err != nil {
            return []string{failFunction}, err
        }
        for _, logGroup := range result.LogGroups {
			if logGroup.RetentionInDays == nil {
                fmt.Println(*logGroup.LogGroupName)
                _, err := client.PutRetentionPolicy(context.TODO(), &cloudwatchlogs.PutRetentionPolicyInput{
                        LogGroupName:    logGroup.LogGroupName,
                        RetentionInDays: aws.Int32(defaultRetentionInDays),
                })
                if err != nil {
                    return []string{failFunction}, err
                }				
            }
        }
		
```
The code above is a snippet from the `main.go` file. The function loops through all the log group that do not have a retention policy set.

if no retention policy is set, it sets the retention policy to the default of `7 days`.

In the above code I am only looking for log groups that are created by Lambda functions. You can modify the code to look for other log groups.

### Building the binary
cd into the `cwl/code` folder and compile the code by running

```bash
  GOARCH=amd64 GOOS=linux go build -tags lambda.norpc -o bootstrap main.go
```
which should create a binary bootstrap.

cd back to the `project root` folder.

## Terraform
Now that we have created the binary for our AWS Lambda function, we can move on to the Terraform steps.

### main.tf
In this file, we create a zip archive with our go binary.

The resource `aws_lambda_function` creates our AWS Lambda function with the zip file as its source and other settings to indicate the runtime is go.

I also create a Cloudwatch Event rule to run the function every 12 hours.

The resource `aws_lambda_permission` sets up Events to invoke our AWS Lambda function.

### iam.tf
In this file, we add permission for Lambda to work with CloudWatch logs and the ability to Describe and Stop Instances. It is also set up to let the lambda publish to the SNS topic.

### requirements.tf
Used to set my provider versions.

### Deployment
Go ahead and run a Terraform init next.

Followed by a Terraform plan. If the plan is good, you can go ahead and run an apply, which will create the AWS resources.

`terraform apply -auto-approve`

### Testing
You can either wait for the CloudWatch Events rule to trigger your function, or you can go to the Lambda console and create a test event to manually run your function

The following image shows that the function ran and set the log retention period for log groups which did not have any retention period.
![CloudWatch Logs](/aws/lambda/cwl-logs-lambda.jpg)

### Remove Resources
If you no longer wish to keep this function, you can remove the resources that were created by running this command.

```bash
terraform destroy
```

This will remove all the resources that were created. Remember you are responsible for all charges incurred.
Leaving a Lambda function with a CloudWatch Event rule enabled, will cost you even if there are no servers to terminate.

You may also have to clean up your CloudWatch Logs.

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

Let me know if you have any questions or suggestions by leaving a comment on the `github repository` linked above.

## Further Reading
- [Terraform Docs](https://developer.hashicorp.com/terraform/docs)
- [Terraform Access](https://www.terraform.io/docs/providers/aws/index.html)
- [EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cwe-now-eb.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [AWS IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)