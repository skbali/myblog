+++
title = "Stop Long Running EC2 Instances"
summary = "Lambda function to stop long running ec2 instances"
date = 2021-12-25T20:11:00Z   
tags = ["aws", "lambda", "python", "serverless", "sns", "Notification", "EventBridge", "CLI", "ec2"]
canonicalURL = "https://blog.skbali.com/2021/12/lambda-function-to-stop-long-running-ec2-instances/"
ShowReadingTime = true
draft = false
+++

While working in my personal AWS account, sometimes I forget to shut EC2 instances down. Most of these instances do not cost much to run, however the charges keep adding up and no one likes to waste money.

A Lambda function is a great way to monitor your account for long-running instances and can be used to shut down those instances to save money.

So if you forget to shut down instances, the lambda function will notify you and shut it down.

If for some reason, you wanted to keep the server up for some more time, you could change the tag associated with the instances which the lambda function looks at.

I decided to deploy a Lambda Function which can use instance tags to determine if an instance has gone past its allotted runtime. 
The function will notify by sending an email when it detects that an instance has been selected to be stopped

## Prerequisites
Make sure you have installed the [Serverless Framework](https://serverless.com/framework/docs/) on your workstation.

Ensure that the workstation from where you are going to run the serverless framework has all the necessary permissions to publish your Lambda function. 
The workstation should have a `IAM Role` or `Access Keys` configured correctly.

We need a SNS Topic which I have shown how to set up below.

## Procedure

### SNS
Let us go ahead and create a SNS topic with an email subscription first.

Run these two commands from your linux shell.

```bash
topicARN=$(aws sns create-topic --name site-monitor --output text)
aws sns subscribe --topic-arn $topicARN --protocol email --notification-endpoint <your email>
```

**Make sure you edit line and set your email for SNS Subscription.**
Once you run these commands, you will receive an email as shown below.

![SNS Subscription](/aws/lambda/sns-notification-1.png)

Make sure you edit line and set your email for SNS Subscription.

Once you run these commands, you will receive an email as shown below.


### Lambda Function
Now it is time for us to set up our Lambda function using the Serverless Framework.

To create the template for our function, I run the commands as shown below.

```bash
sls create --template aws-python3 --path stop-aws-instances
cd stop-aws-instances
```

Here you will see two files `serverless.yml` and `handler.py`, I am going to update these files with our code as shown below.

The file `serverless.yml` contains information about our Lambda function and the IAM role that will be needed by the function.

File `handler.py` is where we will write the code which accomplishes our task.

{{< gist e7402ec7d8a15353e969de99b8142dc8 >}}

Let's walk through the `handler.py` file.
**But first a few important edits in this file.**

- Make sure you edit your `account number` on line #5 to ensure your topic ARN is valid.
- Line #9 sets the max hours that an instance is allowed to run.

Following is a brief description and workflow of the function, without going into too much detail.

### Method check_and_stop_ec2
The method starts by making a describe instances call to find all running instances that contain a tag AllowStop.

A for loop will loop through all the instances returned in the response object.

For each of the instances, I am looking for the AllowStop tag and the value associated with it.

If the value is greater than the max_hours specified then it is replaced.

If the value is  0 or less than 0, I am changing the threshold value to a low number so that the instance can be shutdown right away.

### serverless.yml
Let us take a look at the `serverless.yml` file now.

Few important edits in this file.

- Line #8 specify the name of your deployment bucket.
- Line #10 set a tag value that makes sense to you.
- Line #36 sets the interval at which this Lambda runs and examines the running instances.

In this file, we are setting the function runtime information, the bucket where serverless will deploy the code too. 
The IAM role needed for the function to query tags and stop EC2 instances.

We also specify the memory allocated to our function, the timeout period and the architecture. 
In this example I set the architecture to arm64 to save even more on Lambda execution costs.

The events section in the file specifies that this function should run every 5 minutes. 
Serverless will take care of setting up the Event and its frequency to invoke your function accordingly.

### Deployment
It is now time to deploy our function. The deployment can be done by running the command sls deploy as shown below:
```bash
sls deploy
```

This will trigger the process of creating the CloudFormation scripts which in turn will create a Lambda Function, an IAM role for your function and a CloudWatch Events Rule.

After the resource creation completes, go to CloudFormation to see the stack information to confirm the details. 
Note: This is not needed but good to look up. If there are errors in the execution, you will most likely see them on the command line itself.

If you make any changes to either of the two files, you can simply run `sls deploy` again to push the latest code up to your account.

### Testing
Let's go ahead and set a tag on one of the instances we want our Lambda function to shutdown.

```bash
aws ec2 create-tags --resources i-someinstanceid --tags Key=AllowStop,Value=1
```

The above command will set a tag on your instance and the Lambda will stop this instance if it has been running for more than an hour.
You can either wait for the CloudWatch Events rule to trigger your function, or to test this function immediately from your workstation you can run this CLI command with your account number set (command and output listed):

```bash
aws lambda invoke --function-name arn:aws:lambda:us-east-1:<your account>:function:stop-aws-instances-prd-save-money response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```
Note: With the default setting, the function will not allow an instance to run more than 10 min (600 seconds) if the AllowStop tag is set to a value lower than 1. 
This can be changed by editing line #38 of `handler.py`.

### Remove Resources
If you no longer wish to keep this function, you can remove the resources that were created by running this command.

```bash
sls remove
```
This will remove all the resources that were created. Remember you are responsible for all charges incurred. 
Leaving a Lambda function with a CloudWatch Event rule enabled, will cost you even if there are no servers to terminate.

You may also have to clean up your CloudWatch Logs and S3 buckets where this code was deployed.

## Improvements
Review exception handling, adding comments etc. to make it more meaningful for you.

Use fractions or seconds or other time units to specify runtime duration.

## Conclusion
The use of AWS Lambda functions can be a powerful tool for managing resources in your AWS account. 
By leveraging the Serverless Framework, we can easily deploy a function that monitors and stops long-running EC2 instances, potentially saving significant costs. 

This approach also demonstrates the flexibility and control that cloud services offer, allowing us to automate tasks that would otherwise require manual intervention. 

However, it's important to remember that while automation can be beneficial, it also requires careful management to avoid unintended consequences. 

Always ensure that your functions are thoroughly tested and that you understand the implications of the actions they perform.

Let me know if you have any questions or suggestions by leaving a comment on the `github gist`.

## Further Reading
- [Serverless Docs](https://www.serverless.com/framework/docs/providers/aws/guide/intro/)
- [EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cwe-now-eb.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [AWS IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)