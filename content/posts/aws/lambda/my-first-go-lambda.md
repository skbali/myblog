+++
title = "My First Go Lambda"
summary = "Deploying my first go Lambda function using Terraform"
date = 2022-06-19T14:26:00Z   
tags = ["aws", "lambda", "go", "terraform"]
ShowReadingTime = true
draft = false
+++

Python is an easy-to-read language and has great support and libraries in the developer community. Most of my AWS Lambda functions are written in Python.

However, some of my recent projects revolve around EKS and I have been looking at more code in Go and decided to learn it.

The first personal project that came to mind was trying out an AWS Lambda function written in go using the latest AWS SDK for Go V2.

This post is all about deploying my first go Lambda function, there is no pros and cons or comparison between Python and go.

All my previous post have been using Serverless, which I love to use for personal projects. however, this deployment I am going to do using Terraform.

As I get more comfortable with Go, I might convert my other Python Lambda functions to Go to get a better grasp of the language

## Prerequisites
I do most of my work on my Ubuntu workstation and have setup a local AWS profile to access my AWS account. Ensure you have an AWS account and configured your credentials to access AWS.

Configure your AWS Provider in `main.tf` by either creating a profile or using AWS credentials. See AWS credentials. [See Access](https://www.terraform.io/docs/providers/aws/index.html/).

Download and Install [Terraform](https://www.terraform.io/downloads).

Install [Go](https://go.dev/doc/install) on your workstation.

## Procedure
There are two steps involved in publishing your Lambda function

- Compiling your Go code into a binary

- Deploying a lambda function via Terraform that uses the binary created in the first step.  

**Note:** Running this demo will incur some cost on AWS. You are responsible for all charges incurred in your account. You can lower your cost by cleaning up and removing all resources after you are done.

Instead of just writing a Hello world, I wanted to I wanted to use the SDK to query some AWS resources, so the first go Lambda is going to attempt in listing all running AWS EC2 instances in my account.

## Github Link
You can download the code for this post from my [github repository](https://github.com/skbali/first-go-lambda/).

If you are new to Go, checkout some of the [Go Tutorials](https://go.dev/doc/tutorial/getting-started/) and [Go Docs](https://go.dev/doc/).

This is my first attempt at Go, I may not have followed all the Go rules, formatting, error handling correctly. As I learn more, I will surely update my code and this post.

### main.go
The init function is where the EC2 client is configured to run in `us-east-1`.

```go
    func init() {
	    cfg, err := config.LoadDefaultConfig(context.TODO(), config.WithRegion("us-east-1"))
	    if err != nil {
		    panic("configuration error, " + err.Error())
	    }

	    client = ec2.NewFromConfig(cfg)
    }
```

In a future update, will try to find a way to pass that as a parameter instead of a fixed string.

As per [AWS docs](https://docs.aws.amazon.com/lambda/latest/dg/golang-handler.html), setting the package to be main and the main function has the HandleRequest call. I dont have an event being passed, so decided to leave it out.

Within the HandleRequest function, set up a `DescribeInstances` call with a filter to list only `running` EC2 instances.

```go
	result, err := client.DescribeInstances(context.TODO(), &ec2.DescribeInstancesInput{
		Filters: []types.Filter{
			{
				Name: aws.String("instance-state-name"),
				Values: []string{
					"running",
				},
			},
		},
	})
```

Additional filters or tags could be added by you to narrow down the result if needed.

Then loop through the result to print the EC2 instance information.

```go
	var status []string
	for _, r := range result.Reservations {
		for _, ins := range r.Instances {
			status = append(status, fmt.Sprintf("InstanceID: %v State: %v", *ins.InstanceId, ins.State.Name))
		}
	}
```

### Building the binary
cd into the `running-ec2/code` folder and compile the code by running 

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

### iam.tf
In this file, we add permission for Lambda to work with CloudWatch logs and the ability to Describe Instances.

### versions.tf
Used to set my provider versions.

### Deployment

Go ahead and run a Terraform init next.

Followed by a Terraform plan. If the plan is good, you can go ahead and run an apply, which will create the AWS resources.

`terraform apply -auto-approve`

## Testing
You can either wait for the function to run, or you can invoke it from the command line as shown below to confirm the function was deployed correctly.
 
```bash
aws lambda invoke --function-name arn:aws:lambda:us-east-1:0123456789012:function:first-go-lambda response.json && cat response.json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
["InstanceID: i-0777e6b054614xxxx State: running"]
```

## Summary
Using Terraform, it was quite easy to deploy the Lambda function using Go runtime.

Do not forget to run `terraform destroy` to remove all the resources that were created. Leaving resources in your account will incur cost.

Let me know if you have any questions or suggestions by leaving a comment on the `github repository` linked above.

## Further Reading
- [Terraform Docs](https://developer.hashicorp.com/terraform/docs)
- [Terraform Access](https://www.terraform.io/docs/providers/aws/index.html)
- [AWS IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)