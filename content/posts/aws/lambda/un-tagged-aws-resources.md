+++
title = "Find un-tagged AWS resources using Lambda function"
summary = "Lambda function to find AWS resources that are not tagged"
date = 2024-05-12T04:16:00Z   
tags = ["aws", "tags",  "lambda", "go", "terraform", "Notification", "EventBridge", "ec2", "ebs", "snapshots"]
ShowReadingTime = true
draft = false
+++

In an earlier post, I showed how to [find untagged AWS resources using Python](/posts/aws/find-un-tagged-aws-resources/). In this post, I am going to show you how to do the same using a Lambda function written in Go an deployed using Terraform.

Everyone who works with AWS resources at some point of time is asked how much is the spend for a project and in a traditional server environment, the cost of EC2, EBS and Snapshots make up a good part of the bill.

I also added query to find Lambda functions that are not tagged just to improve upon the previous post.

Amazon Web Services allows us to assign metadata to our AWS resources in the form of tags. Each tag is a simple label consisting of a customer defined key and an optional value. Setting tags for AWS resources makes it easier for us to manage, search for, and filter resources.  These tags can also be used for keeping track of costs by enabling these tags under 'Billing and Cost Allocation Tags'.

After you tag your AWS resource and activate it, you should be able to run reports and check the cost associated with tags and resources.

You should also modify your infrastructure build scripts to make sure all new AWS resources created have the correct cost allocation tags associated with them.

## Problem Statement
Let's assume that in your environment, maybe a non-production environment, you have a group of developers who have access rights to create AWS resources, you would have to make sure they follow your best practices and use your cost tags for the resources they create. 

But it is possible that during testing or developing they fail to assign the cost allocation tags correctly. In this case you will end up with a resource missing tag, and you would not even know.

## Solution
In this post, I am going to show you how to set up a lambda function that you can use to find these AWS resources that do not have the cost allocation tag and alert you. 

In the previous post, we just had a script that had to be scheduled to run. Lambda functions are a great way to run code on a schedule and can be used to monitor your account for untagged resources.

CloudWatch Events rule can be configured to trigger the Lambda function on a schedule. The schedule could be once a day or more.

I use SNS to send alerts regarding missing tags. However, you can extend the code in any other way. Instead of sending an alert, you could modify the code to create tags right then and there.

In the Terraform code, I use a single SNS topic for all my notifications therefore I have not added the SNS topic creation code in this post.

See this post to create a SNS topic and subscribe to it. [SNS Topic](/posts/aws/lambda/stop-ec2-instances/)

## Code
The Go code is passed in the `TOPIC_ARN` which is used for sending out e-mail notifications.

```go
topicArn = os.Getenv("TOPIC_ARN")
```

In this lambda I wanted to try and bring in concurrency to the code, so that multiple resources could be checked at the same time. I used the `sync.WaitGroup` to achieve this.
```go
	results := make(chan string, 4)
	var wg sync.WaitGroup
	wg.Add(4)
	go func() {
		defer wg.Done()
		results <- checkResources("Snapshots", checkSnapshots)
	}()
	go func() {
		defer wg.Done()
		results <- checkResources("Instances", checkEC2Instances)
	}()
	go func() {
		defer wg.Done()
		results <- checkResources("Volumes", checkEBSInstances)
	}()
	go func() {
		defer wg.Done()
		results <- checkResources("Lambda", checkLambdaFunctions)
	}()
	wg.Wait()
	close(results)
```
A `results` channel is created to receive the results from the goroutines. The `sync.WaitGroup` is used to wait for all the goroutines to finish.

```go
	var out strings.Builder
	for result := range results {
		if result != "" {
			out.WriteString(result + "\n\n")
		}
	}

	_, err := snsClient.Publish(context.TODO(), &sns.PublishInput{
		Message:  aws.String(out.String()),
		TopicArn: aws.String(topicArn),
		Subject:  aws.String("Un-Tagged Resources"),
	})
```
The results are then read from the channel and sent out as a single message to the SNS topic.

The other functions `checkSnapshots`, `checkEC2Instances`, `checkEBSInstances` and `checkLambdaFunctions` check for the resources with missing tags.


The `checkResources` function is used to call the other functions concurrently.

```go
func checkResources(resourceType string, checkFunc func() ([]string, error)) string {
	msg, err := checkFunc()
	if err != nil {
		fmt.Println(err)
		return err.Error()
	}
	if len(msg) > 0 {
		return fmt.Sprintf("%s without CostCenter tag: %v", resourceType, msg)
	}
	return ""
}
```

You can modify the code to add more functions to check for other AWS resources like Load Balancers and resources that have a significant cost associated with them.

## Compile
The code can be compiled using the following command:
```bash
GOARCH=arm64 GOOS=linux go build -tags lambda.norpc -o build/bootstrap main.go
OR if you want an x86_64 lambda function
GOARCH=amd64 GOOS=linux go build -tags lambda.norpc -o build/bootstrap main.go
```
I have converted almost all my Lambda functions to ARM64 to save on costs.

Once the Go binary is created, you can run the terraform code to deploy the Lambda function.

## Terraform
I was a heavy user of Serverless Framework but I have started using Terraform for my AWS deployments. I find it easier to manage and maintain.

The Terraform code is used to deploy the Lambda function and the CloudWatch Event rule that triggers the Lambda function.

- `iam.tf` is used to set up the IAM roles and permissions for the Lambda function.
- `main.tf` is used to create the Lambda function and the CloudWatch Event rule.
- `state.tf` is pointing to a S3 bucket for the state file. 

```bash
 terraform init
 terraform plan
 terraform apply -auto-approve
```

That is all is needed to deploy the Lambda function. Now you can trigger it to run from the AWS console if you do not wish to wait for the CloudWatch Event rule to trigger it.

You will get an SNS notification if there are any untagged resources in your account.

Here is an image of the SNS notification I received.

![SNS Notification](/aws/lambda/sns-notification-lambda-no-tags.png)

## Set Tags Example
You can set tags using the AWS CLI. For example, to set tags for an EC2 instance you can use the following command
```bash
aws ec2 create-tags --resources $id \
    --tags Key=CostCenter,Value=admin Key=prometheus,Value=enabled
```
The variable `$id` can be any resource like an instance or a volume.

This command sets two tags for the resource.

You can put this in a for loop in a script if there are multiple resources.

## GitHub Link
The code for this Lambda function is available on my GitHub repository. [Find Untagged Resources](https://github.com/skbali/un-tagged-aws-resources)

Leave a comment on the repository if you have any questions or suggestions.

## Further Enhancements and Reading
- Use your own alerting mechanism if you do not wish to use SNS.
- Extend the code to maybe create the tags based on some predefined business rule and also alert.
- [AWS cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [Best Practices for Tagging AWS Resources](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tagging-best-practices.html)
