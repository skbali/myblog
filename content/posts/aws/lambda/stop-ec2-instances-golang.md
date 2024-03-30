+++
title = "Stop Long Running EC2 Instances - Golang"
summary = "Lambda function in Golang to stop long running ec2 instances"
date = 2024-03-29T22:24:00Z   
tags = ["aws", "lambda", "go", "terraform", "Notification", "EventBridge", "ec2"]
ShowReadingTime = true
draft = false
+++

I have been converting my python Lambdas to Go and using Terraform to deploy them. 

This post is about converting the [stop-ec2-instances](/posts/aws/lambda/stop-ec2-instances/) Lambda to Go and deploying it using Terraform. 

A Lambda function is a great way to monitor your account for long-running instances and can be used to shut down those instances to save money.

So if you forget to shut down instances, the lambda function will notify you and shut it down.

If for some reason, you wanted to keep the server up for some more time, you could change the tag associated with the instances which the lambda function looks at.

I decided to deploy a Lambda Function which can use instance tags to determine if an instance has gone past its allotted runtime. 
The function will notify by sending an email when it detects that an instance has been selected to be stopped

## Prerequisites
Download and Install [Terraform](https://www.terraform.io/downloads).

Install [Go](https://go.dev/doc/install) on your workstation.

We need a SNS Topic, which can be seen in the [stop-ec2-instances](/posts/aws/lambda/stop-ec2-instances/) post.

I decided not to add the SNS Topic in Terraform since it is already created in the previous post.

## Procedure

## Github Link
You can download the code for this post from my [github repository](https://github.com/skbali/stop-ec2-instances).

If you are new to Go, checkout some of the [Go Tutorials](https://go.dev/doc/tutorial/getting-started/) and [Go Docs](https://go.dev/doc/).

### Lambda Function
Now it is time for us to set up our Lambda function using Terraform.

```go
func init() {
	cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion(os.Getenv("REGION")))
	if err != nil {
		panic("configuration error, " + err.Error())
	}

	client = ec2.NewFromConfig(cfg)
	snsClient = sns.NewFromConfig(cfg)

	topicArn = os.Getenv("TOPIC_ARN")
	maxHours, err = strconv.Atoi(os.Getenv("MAX_HOURS"))
	if err != nil {
		panic("configuration error, " + err.Error())
	}
}
```

The variable `topicArn` is set to the SNS Topic ARN which was created in the previous post.
Variable `maxHours` is set to the maximum number of hours an instance can run before it is stopped.
```go
    Filters: []types.Filter{
        {
            Name: aws.String("instance-state-name"),
            Values: []string{
                "running",
            },
        },
        {
            Name: aws.String("tag-key"),
            Values: []string{
                "AllowStop",
            },
        },
    }
```

The above filter lets us find all running instances that have a tag `AllowStop`. 
The value of this tag should be set too how long are you willing to keep an instance running.

In the checkInstance function, we check the instance launch time and compare it with the current time. 
If the instance has been running for more than the allowed time, we send an SNS message to the topic at a pre-determined time which is a bit before the threshold.

Once an instance crosses the threshold, we stop the instance and send an SNS message to the topic.

```go
    _, err := snsClient.Publish(context.TODO(), &sns.PublishInput{
        Message:  aws.String(out.String()),
        TopicArn: aws.String(topicArn),
        Subject:  aws.String("EC2 Instance Shutdown Warning"),
    })
```
The string messages returned from the checkInstance function are sent to the SNS Topic.

### Building the binary
cd into the `stop_ec2/code` folder and compile the code by running

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

I also create a Cloudwatch Event rule to run the function every 30 min.

The resource `aws_lambda_permission` sets up Events to invoke our AWS Lambda function.

THe `data` resource is used to look up our SNS topic.

### iam.tf
In this file, we add permission for Lambda to work with CloudWatch logs and the ability to Describe and Stop Instances. It is also set up to let the lambda publish to the SNS topic.

### requirements.tf
Used to set my provider versions.

### Deployment
Go ahead and run a Terraform init next.

Followed by a Terraform plan. If the plan is good, you can go ahead and run an apply, which will create the AWS resources.

`terraform apply -auto-approve`

### Testing
Let's go ahead and set a tag on one of the instances we want our Lambda function to shut down.

```bash
aws ec2 create-tags --resources i-someinstanceid --tags Key=AllowStop,Value=1
```

The above command will set a tag on your instance and the Lambda will stop this instance if it has been running for more than an hour.
You can either wait for the CloudWatch Events rule to trigger your function, or to test this function immediately from your workstation you can run this CLI command with your account number set (command and output listed):

```bash
aws lambda invoke --function-name arn:aws:lambda:us-east-1:<your account>:function:stop-ec2-go response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```
Note: With the default setting, the function will not allow an instance to run more than 10 min (600 seconds) if the AllowStop tag is set to a value lower than 1. 


### Remove Resources
If you no longer wish to keep this function, you can remove the resources that were created by running this command.

```bash
terraform destroy
```
This will remove all the resources that were created. Remember you are responsible for all charges incurred. 
Leaving a Lambda function with a CloudWatch Event rule enabled, will cost you even if there are no servers to terminate.

You may also have to clean up your CloudWatch Logs.

## Improvements
Review exception handling, adding comments etc. to make it more meaningful for you.

Use fractions or seconds or other time units to specify runtime duration.

## Conclusion
The use of AWS Lambda functions can be a powerful tool for managing resources in your AWS account. 
By leveraging the Serverless Framework, we can easily deploy a function that monitors and stops long-running EC2 instances, potentially saving significant costs. 

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