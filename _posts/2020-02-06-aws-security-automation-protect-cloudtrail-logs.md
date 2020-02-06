---
layout: post
title: "Serverless Security Automation - Part 1: Protect AWS CloudTrail logs from deactivation"
category: Cloud
author: Milo
---
This post will cover the following content for getting started with Serverless Security Concepts on AWS using the CDK:

* Install CDK dependencies with [poetry](https://python-poetry.org/ "poetry's homepage") - currently the best dependency and package manager for Python out there
* Use the Python language to define infrastructure with the CDK
* Detect deactivation of CloudTrail logs using CloudWatch Event Rules
* Use AWS Lambda to reactivate CloudTrail logs and block misbehaving users

When reading this, the first question that might have popped up in your head might be: "Why do we use Python when the CDK is written in Typescript?" The answer is simple: Because everything ending on '\*script' sucks, that's why ;-)

The purpose of our application is to create an AWS CloudTrail, that is being monitored for changes by a CloudWatch Event Rule. Each modification triggers a Lambda that forwards the received change event to an SNS topic and notifies all subscribers via mail. In case someone tries to stop the logging, the Lambda reactivates it and gives the delinquent a juicy slap on the fingers by revoking all of his permissions.

![Flow Diagram: Serverless Security Automation - CloudTrail reactivation](/assets/flow_diagram_cloudtrail_reactivation.png "Flow Diagram: Serverless Security Automation - CloudTrail reactivation")

1. User modifies CloudTrail log.
2. A CloudWatch Events rule detects the change.
3. CloudWatch triggers a Lambda and passes the event details to it.
4. Lambda publishes the change event to an SNS topic. Subscribers are notified via email.
5. Lambda checks if CloudTrail logging was disabled
6. If CloudTrail logging was disabled, the Lambda reactivates the log.
7. If CloudTrail logging was disabled, the Lambda strips the delinquent of all permissions by awarding him with the `AWSDenyAll` policy.

### Disclaimer

This is neither an introduction to CDK nor to AWS or Python. This post is for readers who have at least gone through introductory material (e. g. Getting Started guides) for CDK and AWS. After you've gone through some initial tutorials you might be asking 'hm, nice, now I can create an S3 bucket or a VPC with the CDK. But how do I actually create something useful? Something that actually does something? How do I make components interact with each other?' If you're asking yourself these questions - this post is for you!

Before we proceed, a quick word of caution: This application is neither neccessarily useful for real-world production use, nor is it waterproof in terms of security. For example, a user with sufficient permissions could simply delete our 'Security Lambda' or the S3 bucket of our CloudTrail log. So don't be a smartass about it. The main purpose of this is to get you acquainted with concepts that can support you in some way with your everyday chores. I do not take responsibility for any direct or indirect damages caused by the use of the concepts or the code presented here.

Now that we got all the legal crap out of the way, let's continue on to the interesting stuff...

### Prerequisites

Since this is a step-by-step instruction on how to deploy our application using the AWS CDK with the Python language, I will explain all steps neccessary to reproduce the application from start to finish. However, I assume that you have some prerequisites in place. After all, this tutorial is about serverless cloud (security) automation and the CDK, not about installing stuff ;-)

If you are impatient and just want to play around with the application, you are welcome to check out this [GitHub repo](https://www.github.com/milo0/cloudtrail_protection "GitHub repository for cloudtrail protection CDK stack").

To follow along with this tutorial, you will need the following ingredients:

* An AWS account with Admin access
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html "AWS Command Line Interface installation")
* Python
* [poetry](https://python-poetry.org/ "poetry's homepage") (a dependency and package management tool for Python). It is not mandatory to use poetry, but I like to use it because of its simplicity. You can also use pip+virtualenv, pipenv or whatever your preference is. I have worked with these tools in the past and believe me: there is a reason why I switched to poetry ;-)
* NPM (Node Package Manager)
* [AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) (duh!) At the time of writing the most recent version is 1.22.0

# Initialize a new CDK app and its dependencies

Initializing a new CDK app is pretty simple. Just execute the following command:

```bash
mkdir cloudtrail_protection && cd $_ && cdk init --language=python
```

This will create the following directory structure:

```
.
├── .env
│   ├── bin
│   ├── include
│   ├── lib
│   └── pyvenv.cfg
├── .git
│   ├── HEAD
│   ├── config
│   ├── description
│   ├── hooks
│   ├── index
│   ├── info
│   ├── objects
│   └── refs
├── README.md
├── app.py
├── cdk.json
├── cloudtrail_protection
│   ├── __init__.py
│   └── cloudtrail_protection.py
├── requirements.txt
├── setup.py
└── source.bat
```

This is the default CDK app skeleton, which assumes that you'll be using pip and virtualenv. If you're willing to use these tools, you can procede by following the instructions that appear on your commandline after executing the above command. However, since we want to mix things up a little bit by using poetry, we will diverge from the usual path.

So let's proceed by first removing some unneeded files and directories:

```bash
rm -rf .env requirements.txt setup.py source.bat
```

The functionality of these files will be covered entirely by poetry's equivalents. `source.bat` won't be needed anyways, since that file is for Windows (yuk!).

Now we can initialize our project with poetry:

```bash
poetry init -n
```

This will create a file called `pyproject.toml` in our directory. It is used by poetry to orchestrate your project and its dependencies. You can also have poetry guide you through the initialization process interactively by leaving out the `-n` flag, but for our purposes this 'silent' mode will suffice. We can now add our dependencies:

```bash
poetry add aws-cdk-core aws-cdk-aws-{iam,sns,sns-subscriptions,events-targets,cloudtrail}
```

After executing this line, poetry will do two things:
1. Create a hidden directory called `.venv`, which yields the virtualenvironment for our project. This is the directory you have to set as your project SDK in your IDE.
2. Add the specified dependencies with version constraints to the `pyproject.toml` file.

Step 2 can also been done by adding the dependencies manually to `pyproject.toml` and executing `poetry install` after that, but using `poetry add` is way easier in this case. If you want to find out more about how to add dependencies and version constraints with poetry, you can have a look at the [documentation](https://python-poetry.org/docs/basic-usage/ "poetry basic usage documentation").

---
**NOTE:**
If you cloned the [GitHub repository](https://www.github.com/milo0/cloudtrail_protection) for this project, you **will** have to run `poetry install` before proceding.

---

You can check if everything is going well so far by exemplarily importing libraries in Python. You can do this in two ways:
1. Activate the virtualenv created by poetry with `poetry shell` and use Python normally from there.
2. Precede all commands that depend on resources from the virtualenv with `poetry run`.

We will use the second option, since it seems to be more reliable. I have experienced multiple occasions where using `poetry shell` didn't yield the desired results. You can test your setup by executing the following line:

```bash
poetry run python -c 'from aws_cdk import core, aws_cloudtrail'
```

If that doesn't fail, you are most likely ready to go.

Sweet, now that we've got everything set up, let's proceed to writing some code.

# Create a Test user with Admin permissions

For testing purposes we will define a user with Admin permissions. This user will be our 'throwaway delinquent' who tries to deactivate the CloudTrail log. Since our Lambda will strip the log-stopping user of all permissions, we would lock ourselves out of our account if we were to use our default Admin user.

Let's first get our imports in place by opening `./cloudtrail_protection/cloudtrail_protection_stack.py` and adding the following lines at the top of the file:

```python
from aws_cdk import (
    aws_cloudtrail as cloudtrail,
    aws_events as events,
    aws_events_targets as events_targets,
    aws_iam as iam,
    aws_lambda as _lambda,
    aws_sns as sns,
    aws_sns_subscriptions as subs,
    core,
)
```

Defining our user is as easy as adding the following to the `__init__` method of our `CloudtrailProtectionStack` class:

```python
user = iam.User(self, 'myuser',
                managed_policies=[iam.ManagedPolicy.from_aws_managed_policy_name('AdministratorAccess')])
```

The resulting `./cloudtrail_protection/cloudtrail_protection_stack.py` should look like this:

```python
from aws_cdk import (
    aws_cloudtrail as cloudtrail,
    aws_events as events,
    aws_events_targets as events_targets,
    aws_iam as iam,
    aws_lambda as _lambda,
    aws_sns as sns,
    aws_sns_subscriptions as subs,
    core,
)


class CloudTrailProtectionStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)
        user = iam.User(self, 'myuser',
                        managed_policies=[iam.ManagedPolicy.from_aws_managed_policy_name('AdministratorAccess')])
```

and your `app.py` should look like this:

```python
#!/usr/bin/env python3

from aws_cdk import core

from cloudtrail_protection.cloudtrail_protection_stack import CloudTrailProtectionStack

env_DE = core.Environment(region='eu-central-1', account=core.Aws.ACCOUNT_ID)

app = core.App()

CloudTrailProtectionStack(app, 'cloudtrail-protection', env=env_DE)

app.synth()
```

Instead of `eu-central-1` you can choose the region of your liking.

To see if everything works and to get a first sense of achievement we will now deploy our stack. But for CDK to work properly we have to take two more steps. First, we have to provide credentials for our AWS account. Maybe you already have those configured by using the AWS CLI in previous projects. If that's the case, then everything is fine. If not, then you can still set your credentials temporarily via environment variables, which is what we will do here for the sake of simplicity. There are more sophisticated and secure possibilities for providing account credentials, e. g. using `aws-vault` to get the temporary session token of a user with restricted privileges in order to assume an MFA-protected role with elevated privileges. Stuff like this is beyond the scope of this post, but maybe I'll explain it in a dedicated one ;-)

---
**IMPORTANT:**
You should precede the following two commands with a whitespace. The reason for this is that everything that follows the whitespace doesn't get stored to your bash/ZSH history. You do **not** want your account credentials to be discoverable via a `grep` over your command line history!

---

```bash
 export AWS_ACCESS_KEY_ID=insert_your_access_key_here
 export AWS_SECRET_ACCESS_KEY=insert_your_secret_access_key_here
```

After that we have the last preparation step: Bootstrapping your AWS environment. This creates the neccessary infrastructure for CDK to function properly, which at the time of writing consists of only one S3 bucket.

---
**NOTE:**
Bootstrapping your AWS environment may incur charges, because it will accumulate files in the deployed S3 bucket. These charges should amount to only a few cents per month. The cost is not worth mentioning, but you know how it is: If you don't mention it, then people will start nagging at you ;-) Accounts eligible for free tier might get away unscathed, but I don't guarantee it.

---

```bash
poetry run cdk bootstrap
```

Now we are **finally** able to deploy our stack, which currently only consists of an IAM user. Anyways, let's do it:

```bash
poetry run cdk deploy
```

CDK will list the changes it is about to apply to your infrastructure. As we are going to make security-relevant changes (we are creating an IAM user) CDK will ask if you are really sure about your decision. Please confirm and wait for a few seconds until CDK has done its job. After that you can log into your AWS account using your Admin credentials and have a look in IAM. There you should see our newly created user appearing as `cloudtrail-protection-myuser<HASH>`.

# Create the CloudTrail

To create the CloudTrail just add the following lines below the user definition:

```python
trail = cloudtrail.Trail(self, 's3-account-activity',
                         enable_file_validation=True,
                         include_global_service_events=True,
                         is_multi_region_trail=True,
                         management_events=cloudtrail.ReadWriteType.ALL)
```

This will create a CloudTrail that will log all API events in your account across all regions to a central S3 bucket. Note that you do not have to create the S3 bucket explicitly. CDK will automatically create it for you and set the correct permissions in IAM.

# Deploy the CloudTrail reactivation Lambda

In your project root please create a directory called '`lambda`' and in there create a Python file with your Lambda code:

```bash
mkdir ./lambda && touch ./lambda/cloudtrail_reactivator.py
```

For the sake of brevity we will not list the contents of the Lambda function here. Please head over to the project's [GitHub repo](https://www.github.com/milo0/cloudtrail_protection) and copy/paste its contents into your file. Most of it is self-explanatory and padded with enough comments to understand the essential parts. Nonetheless, we will quickly outline its function here after we're done adding it to our stack.

In `./cloudtrail_protection/cloudtrail_protection_stack.py` please add the following lines to your stack `__init__` to create the neccessary resources for the Lambda:

```python
fn = _lambda.Function(self, 'cloudtrail_reactivator',
                      description='Reactivates stopped CloudTrail logs',
                      code=_lambda.Code.from_asset('./lambda'),
                      handler='cloudtrail_reactivator.handler',
                      runtime=_lambda.Runtime.PYTHON_3_8,
                      initial_policy=[
                          # Allow Lambda to re-activate CloudTrail logging.
                          iam.PolicyStatement(resources=[trail.trail_arn],
                                              actions=['cloudtrail:DescribeTrails',
                                                       'cloudtrail:GetTrailStatus',
                                                       'cloudtrail:StartLogging'],
                                              effect=iam.Effect.ALLOW),
                          # Allow Lambda to attach policies to user.
                          iam.PolicyStatement(resources=[user.user_arn],
                                              actions=['iam:AttachUserPolicy'],
                                              effect=iam.Effect.ALLOW,
                                              conditions={'ArnEquals': {"iam:PolicyARN": "arn:aws:iam::aws:policy/AWSDenyAll"}})
                      ])
```

This block defines your Lambda resource. The policy statements defined here will be attached inline to the Lambda execution role. They allow describing the state for detecting `StopLogging` events and reactivating it via the `StartLogging` API call. In addition, the Lambda is permitted to attach the `AWSDenyAll` policy to the user.

## What does the Lambda do?

When the Lambda gets triggered, it checks the received event JSON for a `StopLogging` event invoked on the CloudTrail. If there is none, then it simply forwards the event JSON to an SNS Topic. The ARN for the SNS Topic gets passed to the Lambda via environment variables. If it detects `StopLogging` however, it extracts the user name from the event JSON and uses it to revoke all of his permissions using a Boto3 call to IAM. The incident is then reported to the SNS topic. It includes the event JSON and in addition reports the misbehaving user in the subject of the message.

# Create an SNS Topic and subscriptions

Now that we have a CloudTrail and a Lambda in place, we need a mechanism that allows us to get notified about events via mail. For that we need to create an SNS Topic and add an email subscription:

```python
topic = sns.Topic(self, 'CloudTrailLoggingStateTransition')
topic.add_subscription(subs.EmailSubscription('enter_your_mail_address@here.com'))
```

This will cause every message being published to the SNS Topic to be forwarded to your email. These messages will be JSON-formatted events published by our Lambda in case of specific API calls being performed on the CloudTrail log. The API calls will be detected by a CloudWatch Events Rule. We will define this Rule and the set of monitored calls shortly. But before we do that, we first have to grant our Lambda the permission to publish messages to SNS. The great thing about CDK is that we do not have to deal with the specifics of the required permission policy. We can simply use the following line and CDK will handle all the nasty IAM fidgeting for us:

```python
topic.grant_publish(fn)
```

This by itself won't work however, because Lambda doesn't know the ARN of the SNS Topic to target. If you look at our Lambda code in `./lambda/cloudtrail_reactor.py` you will notice that the Lambda receives the ARN via an environment variable with the line `sns_arn = os.environ['SNS_ARN']`.To make this to work we have to add the environment variable placeholder for the Lambda as follows:

```python
fn.add_environment('SNS_ARN', topic.topic_arn)
```

We're almost done. As already mentioned, the final thing we have to do is to define a CloudWatch Events Rule with a set of CloudTrail API calls that will trigger our Lambda. We can do this by using CDK's `EventPattern` class and pass the resulting object to the CloudTrail's `on_cloud_trail_event` method. We will set it to target our Lambda, so that it will be invoked every time one of the defined API calls is detected:

```python
event_pattern = events.EventPattern(source=['aws.cloudtrail'],
                                    detail={'eventName':   ['StopLogging',
                                                            'DeleteTrail',
                                                            'UpdateTrail',
                                                            'RemoveTags',
                                                            'AddTags',
                                                            'CreateTrail',
                                                            'StartLogging',
                                                            'PutEventSelectors'],
                                            'eventSource': ['cloudtrail.amazonaws.com']})
trail.on_cloud_trail_event('CloudTrailStateChange',
                           description='Detects CloudTrail log state changes',
                           target=events_targets.LambdaFunction(fn),
                           event_pattern=event_pattern)
```

And we're done. Your final stack definition should look like this:

```python
from aws_cdk import (
    aws_cloudtrail as cloudtrail,
    aws_events as events,
    aws_events_targets as events_targets,
    aws_iam as iam,
    aws_lambda as _lambda,
    aws_sns as sns,
    aws_sns_subscriptions as subs,
    core,
)


class CloudTrailProtectionStack(core.Stack):

    def __init__(self, scope: core.Construct, id: str, **kwargs) -> None:
        super().__init__(scope, id, **kwargs)

        trail = cloudtrail.Trail(self, 's3-account-activity',
                                 enable_file_validation=True,
                                 include_global_service_events=True,
                                 is_multi_region_trail=True,
                                 management_events=cloudtrail.ReadWriteType.ALL)

        fn = _lambda.Function(self, 'cloudtrail_reactivator',
                              description='Reactivates stopped CloudTrail logs',
                              code=_lambda.Code.from_asset('./lambda'),
                              handler='cloudtrail_reactivator.handler',
                              runtime=_lambda.Runtime.PYTHON_3_8,
                              initial_policy=[
                                  # Allow Lambda to re-activate CloudTrail logging.
                                  iam.PolicyStatement(resources=[trail.trail_arn],
                                                      actions=['cloudtrail:DescribeTrails',
                                                               'cloudtrail:GetTrailStatus',
                                                               'cloudtrail:StartLogging'],
                                                      effect=iam.Effect.ALLOW),
                                  # Allow Lambda to attach policies to user.
                                  iam.PolicyStatement(resources=[user.user_arn],
                                                      actions=['iam:AttachUserPolicy'],
                                                      effect=iam.Effect.ALLOW,
                                                      conditions={'ArnEquals': {"iam:PolicyARN": "arn:aws:iam::aws:policy/AWSDenyAll"}})
                              ])

        topic = sns.Topic(self, 'CloudTrailLoggingStateTransition')
        topic.add_subscription(subs.EmailSubscription('enter_your_mail_address@here.com'))
        topic.grant_publish(fn)

        fn.add_environment('SNS_ARN', topic.topic_arn)

        # Event Pattern that defines the CloudTrail events that should trigger
        # the Lambda.
        event_pattern = events.EventPattern(source=['aws.cloudtrail'],
                                            detail={'eventName':   ['StopLogging',
                                                                    'DeleteTrail',
                                                                    'UpdateTrail',
                                                                    'RemoveTags',
                                                                    'AddTags',
                                                                    'CreateTrail',
                                                                    'StartLogging',
                                                                    'PutEventSelectors'],
                                                    'eventSource': ['cloudtrail.amazonaws.com']})
        trail.on_cloud_trail_event('CloudTrailStateChange',
                                   description='Detects CloudTrail log state changes',
                                   target=events_targets.LambdaFunction(fn),
                                   event_pattern=event_pattern)

```

You can now run

```bash
poetry run cdk deploy cloudtrail-protection
```

confirm the IAM changes and then lean back and watch CDK do its deployment magic. Somewhere during the deployment process you will receive an email containing a link to confirm your subscription to the SNS Topic. Hit that link and you're ready to proceed to testing the entire thing.

## Test the automated CloudTrail log reactivation

Now that everything is set up, we can finally check out what we've created. First step is to log in as the user we've created. I have to remind you again, that it is not advisable to use your regular Admin login, because after we're done you will end up with effectively no permissions! So after you've logged in, please head over to CloudTrail and select the trail we created via CDK. It should be named something along the lines of `cloudtrail-protection-s3accountactivity<HASH>`. In the upper right corner you will see the toggle button that turns off the CloudTrail log. The moment has come to finally hit that button (and get punished for it ^^)

![Deactivate your CloudTrail log by hitting the toggle switch](/assets/deactivate_cloudtrail_switch_toggle.png "CloudTrail log deactivation toggle")

Shortly after deactivating the CloudTrail log you should receive an email with the subject `CloudTrail event "StopLogging" received. Initiating reactivation...` and a JSON containing the event details in the message body. Shortly after that you should receive another mail stating that the event `StartLogging` has been received. When you hit the refresh button in your browser you will then get the following error message:

![The error message that you get after your user has been stripped of all permissions](/assets/error_after_cloudtrail_deactivation.png "Error message after CloudTrail log deactivation")

That is because (you already guessed it) the Lambda correctly attached the `AWSDenyAll` policy to the issuing user, thus effectively leaving him without any permissions.

## Destroy the stack

Now that our fun little ride is over we need to destroy our stack. There is a simple CDK command for that, but unfortunately we have to do two things manually. This is because our CloudTrail log created an S3 bucket with log files in it and we have created a user with a password set. As soon as an S3 bucket is non-empty, CloudFormation won't be able to delete it (there are community-provided workarounds for CDK, for example this [S3 auto-deletable bucket construct](https://github.com/mobileposse/auto-delete-bucket "GitHub repo for S3 auto-deletable bucket construct")). Same goes for our delinquent user. Removing this resource automatically would be no problem if we hadn't set a password. But since we did, we have to remove the user manually as well.

So in your AWS web console please head over to the S3 service and remove the bucket called `cloudtrail-protection-s3accountactivity<HASH>`. Then head over to IAM and remove the user called `cloudtrail-protection-myuser<HASH>`.

After this is done, you can now destroy the rest of the stack by executing

```bash
poetry run cdk destroy cloudtrail-protection
```

CDK will again ask you for confirmation, which you can confirm with a clear conscience. After a few seconds of patience your stack should be successfully destroyed.

# Conclusion

In this tutorial you learned a few basics about how to build a security-relevant serverless application on AWS. In addition to that, you learned how to use the AWS CDK with Python and poetry. Building upon these concepts you can now build more complex and meaningful automation systems that protect your business infrastructure. Keep in mind however, that reactive systems like the one shown here should only be your second line of defense. Provided you have the required permissions, it is almost always better to protect your infrastructure by using Service Control Policies and IAM.

# Useful links

* Original [blog post](https://aws.amazon.com/blogs/mt/monitor-changes-and-auto-enable-logging-in-aws-cloudtrail/ "Original blog post by AWS") by AWS
* Blog post with a much more complex and closer-to-reality [example](https://aws.amazon.com/blogs/compute/orchestrating-a-security-incident-response-with-aws-step-functions/ "Orchestrating a security incident response with AWS Step Functions") of Serverless Security Automation (also available in Serverless Application Repository as 'Automated-IAM-policy-alerts-and-approvals')
* [AWS CDK Python Workshop](https://cdkworkshop.com/ or https://cdkworkshop.com/30-python.html "AWS CDK Python Workshop")
