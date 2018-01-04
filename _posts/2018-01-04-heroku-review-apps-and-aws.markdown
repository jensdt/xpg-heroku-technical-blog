---
layout: post
title:  "Heroku review apps and AWS: automate your resources"
date:   2018-01-04 09:00:00 +0100
categories: heroku aws
---
I love automating infrastructure tasks. So I was very excited with the release of [Heroku review apps](https://devcenter.heroku.com/articles/github-integration-review-apps). Automatic creation of my Heroku apps? Yes please!
   
But when trying this for one of my applications, I hit a roadblock. We integrate with a few AWS resources, such as [SQS](https://aws.amazon.com/sqs/). When you post a message on an SQS queue, it is only delivered to one of the listeners.
So when we have multiple review apps deployed, they'll share the queue with each other (and the environment they are inheriting from) and a message can end up on any of the applications. Not good! 

We need a way to isolate our AWS resources.

## Prerequisites

This posts assumes you know a bit about Heroku pipelines. I won't go into detail about the setup, so you should be familiar with Pipelines, review apps and the app.json file.

## Step 1: A simple SQS app

Before we go ahead, let's set up a simple example NodeJS project and deploy it to Heroku. If you want to follow along,
you can check out the code on [GitHub](https://github.com/jensdt/heroku-aws-review-app-example/tree/step-1-basic-sqs).

### Heroku setup

1. Create a new Heroku application. I've called mine 'review-and-aws-dev'.
2. Add it to the 'development' stage of a new pipeline.
3. Link your pipeline to your GitHub repo
4. Enable automatic creation of the review apps whenever a PR is raised


### AWS Setup

1. Create an SQS queue. I've called mine 'review-testqueue-dev'
2. Create an IAM user with read+write access on every queue where the name *starts* with review-testqueue-dev (this will be important later). For example:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "sqs:DeleteMessage",
                "sqs:ReceiveMessage",
                "sqs:SendMessage"
            ],
            "Resource": "arn:aws:sqs:eu-west-1:YOUR-AWS-ACCOUNT-ID:review-testqueue-dev*"
        }
    ]
}
```

### Deploy the application.

To run the application locally or on Heroku, you need to set the following environment variables:

```
AWS_ACCESS_KEY_ID=your-access-key            # Must be an IAM user with read/write rights on the queue
AWS_SECRET_ACCESS_KEY=your-secret-key
SQS_BASE_URL="https://sqs.eu-west-1.amazonaws.com/your-aws-account-id" 
SQS_QUEUE_NAME=review-testqueue-dev          # Use whatever name you used
```

So set these, and then deploy the application. If you view the logs, you should see our worker is sending/receiving messages.

### Create a pull request and review app.

If we now create a PR, a review app will be automatically created. However, if you tail the logs of both applications, you'll see that messages being sent by the review app and the development app
are processed by both workers - they are not separated.


## Step 2: Making sure the review apps use their own queues.

### Dynamic queue names

To avoid the queues clashing, we will make a few small changes to our application. First, we will [make sure we have the name of our Heroku app available](https://devcenter.heroku.com/articles/github-integration-review-apps#heroku_app_name-and-heroku_parent_app_name) in our review apps by changing our app.json:

```
"env":{
    <already existing keys> ...
    
    "HEROKU_APP_NAME": {
      "required": true
    }
```

Then, we will change our code to append the Heroku app name to the queue name if this environment variable is set. See [the diff on GitHub](https://github.com/jensdt/heroku-aws-review-app-example/compare/step-1-basic-sqs...step-2-different-queue-names#diff-168726dbe96b3ce427e7fedce31bb0bc).
Note: this is why our IAM policy check was to see if the name of the queue *starts* with a certain value.

### Create a new PR

If we now create a new PR, we'll see the app tries to find a different queue, one with the name: 'review-testqueue-dev-review-and-aws-dev-pr-2' in my case. Of course, this queue does not exist, so we get errors.

But - if we go into our SQS console and create the queue, the errors go away and the app works, using its own dedicated queue. Success!

## Step 3: Automating the queue creation

So now we have separation of our queues, but we still need to create them manually. In order to do this automatically, we will use [CloudFormation](https://aws.amazon.com/cloudformation/) together with a [postdeploy](https://devcenter.heroku.com/articles/github-integration-review-apps#the-postdeploy-script) script.

### Create a new IAM user

We will create a new IAM user for our CloudFormation code since we don't want to give our application user rights to do so. Create a new IAM user with this policy (modify the resources to suit your case):

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "cloudformation:DescribeStackEvents",
                "cloudformation:CreateStack",
                "cloudformation:DeleteStack",
                "sqs:DeleteQueue",
                "sqs:GetQueueAttributes",
                "sqs:CreateQueue"
            ],
            "Resource": [
                "arn:aws:sqs:eu-west-1:YOUR-AWS-ACCOUNT-ID:review-testqueue-dev*",
                "arn:aws:cloudformation:eu-west-1:YOUR-AWS-ACCOUNT-ID:stack/review-and-aws-dev*/*"
            ]
        }
    ]
}
```

Set the access key and secret access key for this user as an environment variable on Heroku:

```
CF_AWS_ACCESS_KEY_ID=your-access-key
CF_AWS_SECRET_ACCESS_KEY=your-secret-key
```

Since our CloudFormation template is very simple, we just inline it. If you want, you could load it from a JSON file and fill in the dynamic properties (like the queue name):

```
{
    Resources: {
        "MyAutoCreatedQueue": {
            Type: "AWS::SQS::Queue",
            Properties: {
                QueueName: queueName
            }
        }
    }
}
```

We also pause until creation is complete. The full code of the postdeploy script can be found on [GitHub](https://github.com/jensdt/heroku-aws-review-app-example/blob/step-3-automation/postdeploy.js)

The last step is then to execute this postdeploy script by modifying our app.json:

```
"scripts": {
    "postdeploy": "node postdeploy.js"
  },
```

Don't forget to also inherit the new CF_ variables from our DEV environment:

```
"env": {
    <already existing keys> ...
    
    "CF_AWS_ACCESS_KEY_ID": {
        "required": true
    },
    "CF_AWS_SECRET_ACCESS_KEY": {
        "required": true
    },
}
```

## Step 4: It works!

If this is all deployed, creating a new Pull Request will create a new review app. The postdeploy script will be run, which creates the CloudFormation stack and all required AWS resources.
All your AWS resources are now nicely isolated for your review apps.

## Future extensions

Further steps that can be taken with this:

- Instead of a postdeploy script, use a release task to make sure the AWS resources are always up-to-date, based on the CloudFormation template, by updating the stack if it already exists instead of creating it.
- Use a predestroy task to clean up your AWS resources when the pull requests are closed

## In summary

By using a postdeploy task, we can easily script actions that need to be taken when creating review apps. 
Don't be afraid to do so - it will help you move to a 'infrastructure-as-code' mindset. 

We were forced to write our AWS resources as a CloudFormation template instead of creating them manually, 
but this not only gives us our isolated review apps, it also made us describe our AWS resources as code. This is a useful exercise in itself - suppose you want to replicate your 
application to a different AWS region. If you already have a CloudFormation template, it is as simple as creating a new stack in the new region.