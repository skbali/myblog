+++
title = "Find un-tagged AWS resources"
summary = "Find AWS resources that are not tagged"
date = 2019-02-15T14:16:00Z   
tags = ["aws", "tags", "python"]
ShowReadingTime = true
draft = false
+++

**See my recent post accomplishing the same [task using a lambda function](/posts/aws/lambda/un-tagged-aws-resources/)**

Everyone who works with AWS resources at some point of time is asked how much is the spend for a project and in a traditional server environment, the cost of EC2, EBS, and snapshots make up a good part of the bill.

Amazon Web Services allows us to assign metadata to our AWS resources in the form of tags. Each tag is a simple label consisting of a customer defined key and an optional value. Setting tags for AWS resources makes it easier for us to manage, search for, and filter resources.  These tags can also be used for keeping track of costs by enabling these tags under 'Billing and Cost Allocation Tags'.

After you tag your AWS resource and activate it, you should be able to run reports and check the cost associated with tags and resources.

You should also modify your infrastructure build scripts to make sure all new AWS resources created have the correct cost allocation tags associated with them.

## Problem Statement
Let's assume that in your environment, maybe a non-production environment, you have a group of developers who have access rights to create AWS resources, you would have to make sure they follow your best practices and use your cost tags for the resources they create. 

But it is possible that during testing or working they fail to assign the cost allocation tags correctly. 
In this case, you will end up with a resource missing tag, and you would not even know.

## Solution
In this post, I am going to show you some sample code that you can use to find these AWS resources that do not have the cost allocation tag and alert you. 

In my case, I run this code as a script from a cron. However, the same code can be run inside a Lambda function, and you can use a CloudWatch event to trigger the function to run on a schedule.

Take a look at some of my earlier posts on [Lambda functions](/tags/lambda/) and how I schedule events using [CloudWatch events](/tags/eventbridge/).

In my sample code, I use SNS to send alerts regarding missing tags. However, you can extend the code in any other way. Instead of sending an alert, you could modify the code to create tags right then and there.

See this post to create a SNS topic and subscribe to it. [SNS Topic](/posts/aws/lambda/stop-ec2-instances/)    

## Code
In your python `main()` function or Lambda, set up functions calls to check for resources with missing tags.

```python
def main():
  
  ec2client = boto3.client('ec2')

  message = {}
  info = list_snapshots(ec2client)
  if info:
     message['Snapshots missing Tags'] = info     

  info = list_volumes(ec2client)
  if info:
     message['Volumes missing Tags'] = info     

  # Add more functions to look for other AWS resources     
  
  if message:
     send_sns(message)
```
Let's take a look at the functions one by one. The first one is checking snapshots in your account. Be sure to replace your account id in the code.

```python
def list_snapshots(ec2client):
  try:
      response = ec2client.describe_snapshots( OwnerIds=['your account id here'] )
  except Exception as e:
      print(e)
      return None

  info = []
  if response:
     for snaps in response['Snapshots']:
        if 'Tags' not in snaps:
            info.append(snaps['SnapshotId'])
        else:
            if 'CostCenter' not in [tags['Key'] for tags in snaps['Tags']]:
               info.append(snaps['SnapshotId'])
  else:
    return None

  return info
```

As you can see, in my case the cost allocation tag I use is called 'CostCenter' and my function looks for snapshots that do not have this tag.

The second function looks for volumes which do not have the 'CostCenter' tag in a similar manner.

```python
def list_volumes(ec2client):
  try:
      response = ec2client.describe_volumes()
  except Exception as e:
      print(e)
      return None

  info = []
  if response:
    for vols in response['Volumes']:
        if 'Tags' not in vols:
            info.append(vols['VolumeId'])
        else:
            if 'CostCenter' not in [tags['Key'] for tags in vols['Tags']]:
               info.append(vols['VolumeId'])
  else:
    return None

  return info
```

The next function that I list here will send out the SNS notification to alert us when a AWS resource with missing cost allocation tag is found.

```python
def send_sns(message): 
  Arn='arn:aws:sns:<region>:<account-id>:email-alert'

  client = boto3.client('sns')
  response = client.publish(
    TargetArn=Arn,
    Subject='Resources missing CostCenter Tags or all tags',
    Message=json.dumps({'default': json.dumps(message, indent=4)}),
    MessageStructure='json'
  )

  return
 ```

You can modify the code to add more functions to check for other AWS resources like EC2 instances, EBS volumes, etc.

## Set Tags Example
```bash
aws ec2 create-tags --resources $id \
    --tags Key=CostCenter,Value=admin Key=prometheus,Value=enabled
```
The variable `$id` can be any resource like an instance or a volume.

This command sets two tags for the resource.

You can put this in a for loop in a script if there are multiple resources

## Further Enhancements and Reading
- Add error/exception handling as per your requirements.
- Use your own alerting mechanism if you do not wish to use SNS.
- Extend the code to maybe create the tags based on some predefined business rule and also alert.
- [AWS cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
- [Best Practices for Tagging AWS Resources](https://docs.aws.amazon.com/whitepapers/latest/tagging-best-practices/tagging-best-practices.html)
