---
layout: post
title: "How to Defend Against AWS Surprise Bills"
backgroundUrl: "https://images.unsplash.com/photo-1603777953913-d4bace4bf01c?auto=format&fit=crop&q=80"
description: "In this article we'll take a look at how billing works, and what you can do to prevent surprise bills."
---

**Short on time?** [Set up Budget Alerts in less than 2 minutes]('#2-budget-alerts).

**Got a surprise bill?** [Here's how you can contact AWS Support]('#contact-support').

Imagine you've been running a hobby project in the cloud for the last 6 months. Every month you
paid 20 cents. Not enough to really care about. However one morning you notice a surprisingly
large transaction of $2700.

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Good morning, $2700 AWS bill!<br><br>Holy shit...</p>&mdash; Chris Short @ KubeCon (@ChrisShort) <a href="https://twitter.com/ChrisShort/status/1279406322837082114?ref_src=twsrc%5Etfw">July 4, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

Cloud computing allows us to pay for storage, compute and other services as we use them. Instead of going to a
computer shop and buying a server rack, we can use services and get a bill at the end of the month. The downside
is however that we can use more than we might have money for. This can be especially tricky with serverless
solutions which automatically scale up with the traffic that comes in.

Accidentally leaving an expensive VM running, or having your Lambda functions spiral out of control, may lead
to a dreaded surprise bill.

{% raw %}
<blockquote class="twitter-tweet"><p lang="und" dir="ltr"><a href="https://t.co/tAqUqCoV9R">pic.twitter.com/tAqUqCoV9R</a></p>&mdash; Fernando (@fmc_sea) <a href="https://twitter.com/fmc_sea/status/1328510918855073793?ref_src=twsrc%5Etfw">November 17, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

In this article we'll take a look at how billing works, and what you can do to prevent surprise bills.

## Focus On Small Bills

This article **focuses on personal or small company accounts** with relatively small
bills. While a $3000 spike in cost might not be noticeable in a large corporate
bill, it can be devastating for a personal account that you run hobby projects on.

## There's No Perfect Solution

Unfortunately there's no perfect solution to prevent surprise bills. As [Corey Quinn explains on his podcast](https://www.lastweekinaws.com/podcast/aws-morning-brief/whiteboard-confessional-the-curious-case-of-the-9000-aws-bill-increase/)
the AWS billing system can take a couple hours to receive all data, in some cases up to 24 or 48 hours. As a result the Budget Alerts might trigger
hours or days after a significant spending happened. Budget alerts are still a great tool to prevent charges that take more than a day
or two to accrue, e.g. forgetting an expensive EC2 instance that you used to follow a machine learning workshop.

It's up to you how much time you want to invest to reduce the risk of surprise bills, but I highly recommend you to
**take 2 minutes to set up Budget Alerts**!

## Defense Mechanisms

There are multiple mechanisms that you can apply to defend against surprise bills. The ones we look into
include security, alerting, remediating actions and improved visibility.

### 1. Secure Your Account With Multi Factor Authentication

This is the **first thing you should set up** when creating a new AWS account.

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Hey! You! 👋 Do you own an AWS account?<br><br>🚨STOP SCROLLING AND CHECK THIS NOW- is MFA enabled on your root account? <br><br>Yes? Cool, carry on 🙋🏻‍♂️<br><br>No? ENABLE IT NOW! PLEASE! 🙏🏽<br><br>This reminder brought to you by an SA who had two customers with theirs account compromised in a week 🙈</p>&mdash; Karan (@somecloudguy) <a href="https://twitter.com/somecloudguy/status/1331288928096309249?ref_src=twsrc%5Etfw">November 24, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

[Follow this official guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa_enable_virtual.html) from AWS to set up multi factor authentication (MFA) for your account.
By activating MFA on your account, you add another barrier for malicious attackers.

### 2. Budget Alerts

This is the **second thing you should set up** when creating a new AWS account.

Budget Alerts are the most popular way to keep an eye on your spending. By creating a budget alert, you
will get a notification e.g. via e-mail which tells you that the threshold has been exceeded. You can further
customize notifications through [Amazon SNS](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/budgets-sns-policy.html)
or [AWS Chatbot](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/sns-alert-chime.html).

Here's a [short video](https://youtu.be/_YXSDIPFhTI) (52 seconds) that you can follow to create your first Budget Alert. [Ryan H Lewis made a longer video](https://www.youtube.com/watch?v=MKNtSOQXFrY) with some more context around Budget Alerts, and the many ways you can configure them.

If you're already using the CDK then the package [aws-budget-notifier](https://awscdk.io/packages/@stefanfreitag/aws-budget-notifier@0.1.5/#/) gets you started quickly.

**What amount should you start with?**

Start with an amount that's a bit above your current spending and that you're comfortable with. If you're just starting out, $10 is probably a good idea.
If you already have workloads running for a few months, then take your average spending and add 50% on top.

I also recommend to set up **multiple billing alerts at various thresholds**:

1. The comfortable alert: This is an amount that you're comfortable spending, but you want to look into the bill over the next days.
2. The dangerous alert: At this amount, you're not comfortable anymore, and want to shut down a service as soon as possible. If your comfortable amount is $10, this one might be $100.
3. The critical alert: At this amount, you want to nuke your account from orbit. With a comfortable amount of $10, this one might be $500. You can attach [Budget Actions](#3-budget-actions) or pager alerts to this alarm to automatically stop EC2 instances or wake you up at night.

As an addition to predefined thresholds, you can also try out [AWS Cost Anomaly Detection](https://aws.amazon.com/aws-cost-management/aws-cost-anomaly-detection/).

**DANGER - The Orbital Nuke Option**

As you can send notifications to SNS, you can trigger a Lambda function that
runs [aws-nuke](https://github.com/rebuy-de/aws-nuke) which will tear down all the infrastructure in your account.
Do not use this on any account that you have production data in. If you want to learn more
about this, [check out the GitHub repository](https://github.com/rebuy-de/aws-nuke).

### 3. Budget Actions

AWS [recently announced Budget Actions](https://aws.amazon.com/about-aws/whats-new/2020/10/announcing-aws-budgets-actions/).
This is an extension to Budget Alerts, where you can trigger actions when a budget exceeds
its threshold. In addition to sending e-mail notifications, you can now apply custom IAM policies
like "Deny EC2 Run Instances" or let AWS shut down EC2 and RDS instances for you as shown below.

![Budget Action to Shut Down an EC2 Instance](https://bahr.dev/pictures/2020/surprisebills/budget-action-shut-down-ec2.png)

### 4. Mobile App

The [AWS Console Mobile Application](https://aws.amazon.com/console/mobile/) puts the cost explorer just 3-5 taps away. This
way you can check in on your spending with minimal effort.

Below you can see two screens from the mobile app:

![Cost Explorer in Mobile App](https://bahr.dev/pictures/2020/surprisebills/mobile-app.png)

To use the app you should [set up a dedicated user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
that only [gets the permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html) that the
app needs to display your spending.

Here's an IAM policy that grants read access to the cost explorer as well as cloudwatch alarms.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ce:DescribeCostCategoryDefinition",
                "ce:GetRightsizingRecommendation",
                "ce:GetCostAndUsage",
                "ce:GetSavingsPlansUtilization",
                "ce:GetReservationPurchaseRecommendation",
                "ce:ListCostCategoryDefinitions",
                "ce:GetCostForecast",
                "ce:GetReservationUtilization",
                "ce:GetSavingsPlansPurchaseRecommendation",
                "ce:GetDimensionValues",
                "ce:GetSavingsPlansUtilizationDetails",
                "ce:GetCostAndUsageWithResources",
                "ce:GetReservationCoverage",
                "ce:GetSavingsPlansCoverage",
                "ce:GetTags",
                "ce:GetUsageForecast",
                "health:DescribeEventAggregates",
                "cloudwatch:DescribeAlarms",
                "aws-portal:ViewAccount",
                "aws-portal:ViewUsage",
                "aws-portal:ViewBilling"
            ],
            "Resource": "*"
        }
    ]
}
```

We can group the permissions into 3 sets:

1. Cost Explorer Read Access (everything that starts with `ce:`). These let us get detailed information about or current and forecasted spending.
2. CloudWatch Alarms Read Access (`cloudwatch:DescribeAlarms`). This allows you to see if there are any alarms,
 but doesn't let you get further than that.
3. General Access (permissions starting with `aws-portal:` and `health:`). These allow you to display the mobile dashboard properly.
As far as I understand and tested, without they don't give you access to the spending details, but without them you can't show the dashboards.

Please [let me know](https://twitter.com/bahrdev) if any of these permissions can be removed.

### 5. Secrets Manager

[If access keys get leaked through public repositories](https://www.reddit.com/r/aws/comments/9i2zzh/huge_unexpected_45k_bill_for_ec2_instances/), malicious actors can start expensive EC2 instances in your account and use it for example to mine Bitcoin.
There are also reports of instances hidden away in less frequently used regions, small enough that they don't get noticed in the bill summary.

To keep your code free from access keys or other secrets, you can use
the [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) to store the secrets which your code needs at runtime.

[Follow this AWS tutorial](https://aws.amazon.com/secrets-manager/) to create your first secret.
Once you've created one, replace the secret from your code base by using one of the official AWS clients ([boto3 for Python](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/secretsmanager.html))
to retrieve the secret.

```python
import boto3

client = boto3.client('secretsmanager')

response = client.get_secret_value(SecretId='replace-me')

secret = response['SecretString']
```

Please note that each secret will cost you $0.40 per month, as well as $0.05 per 10,000 API calls.

## Contact Support

If you experienced a surprise bill, stop the apps that cause the high spending, rotate your access keys if necessary and contact AWS support.

[Here's a 20 seconds video which guides you to the support case](https://youtu.be/oKNAxfmQMZM).

The steps to file a support ticket are:

1. In the top right click on Support and then select the Support Center
2. Press the orange button that says Create case
3. Select Account and billing support
4. As type select "Billing" and as category select "Payment issue"
5. Now fill out the details and submit

While there are folks who got their surprise bill reimbursed, please don't rely on this.

## Conclusion
1. Secure Your Account With Multi Factor Authentication
The first thing you should do is set up [MFA](#1-secure-your-account-with-multi-factor-authentication) and [Budget Alerts](#2-budget-alerts). After that you can look into more advanced
operations like Budget Actions to lock down your account if spending spikes.

If your applications use secrets or access keys, you can prevent them from accidentally ending up in your repositories
by [storing the secrets in the AWS Secrets Manager instead](#5-secrets-manager).

## Resources

* [The AWS bill heard around the world](https://chrisshort.net/the-aws-bill-heard-around-the-world/)
* [Last Week in AWS Podcast](https://www.lastweekinaws.com/podcast/aws-morning-brief/whiteboard-confessional-the-curious-case-of-the-9000-aws-bill-increase/)
* [AWS Checklist for avoiding unexpected charges](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/checklistforunwantedcharges.html)
* [AWS' best practices for budgets](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/budgets-best-practices.html)
* [How to Protect Yourself From Unexpectedly High AWS Bills](https://medium.com/better-programming/how-to-protect-yourself-from-unexpectedly-high-aws-bills-4ec91bbe66f4)
* [Never Get an Unexpected AWS Bill Again!](https://www.ceoraford.com/posts/never-get-an-unexpected-aws-bill-again!/)
* [YouTube: How to avoid Huge AWS Bills with AWS Budgets](https://www.youtube.com/watch?v=MKNtSOQXFrY)
* [YouTube: How to set up Budget Alerts](https://www.youtube.com/watch?v=FVwdlJ8lM0Q)
