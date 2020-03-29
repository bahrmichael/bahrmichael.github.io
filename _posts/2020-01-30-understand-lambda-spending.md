---
layout: post
title: AWS Cost Optimisation - Understanding Lambda Spending
---

Serverless is an amazing approach to building highly scalable and usually low cost systems. But the pay-per-use model combined with increased traffic and code that is not cost optimised may lead to higher bills than anticipated. In this series I will show and discuss my attempts to understand and fix my AWS spendings.

# Understanding Lambda Spending

Assumptions about the reader:
- You know how to use the [AWS Cost Explorer](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/ce-what-is.html) and have [activated its cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/custom-tags.html)

![Last Month's Lambda Spending](https://dev-to-uploads.s3.amazonaws.com/i/r5iurpnrnbxei8ya0lme.png)

As my lambda spending has decently increased over the last weeks, this post will dive into this service's cost sources.

## Lambda Pricing

Lambda is priced on a pay-per-use model. The [official pricing page](https://aws.amazon.com/lambda/pricing) is very detailed, but you can start by assuming that 100.000 seconds of Lambda runtime will cost ~1.7$ when using the default 1024MB of RAM. Depending on the type and size of your workloads you can lower your cost by decreasing the RAM. Increasing it [can also help](https://hackernoon.com/lower-your-aws-lambda-bill-by-increasing-memory-size-yep-e591ae499692), as with more RAM comes more CPU.

Lambda comes with a free tier of "1M free requests per month and 400,000 GB-seconds of compute time per month". The first two weeks in the above graph were probably due to the free tier.

## Cost Tagging

The first step to understanding AWS costs should always be to add tags. When you add tags to your AWS resources, you can then tell the Cost Explorer to [scan those tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/activating-tags.html) and include them in your reports.

![Lambda spending with tags](https://dev-to-uploads.s3.amazonaws.com/i/rksielpepv5q5jlaiom3.png)

As you can see above, I've been a bit sloppy with tagging my resources. If you have only a few projects, then I suggest to just open them and add tags. Tools like the serverless framework can be helpful with that. You can also add tags manually through the AWS Console.

Note that tagging does not retroactively mark spending. Only new spending will have the additional information. Therefore it is crucial that you add some basic tags like "project" or "name" to all of your new resources. Otherwise you have to write blog posts while you wait for new information to be collected ;)

## Identifying Untagged Lambda Spending

Let's open up CloudWatch which has some usage [metrics](https://console.aws.amazon.com/cloudwatch/home#metricsV2:) to offer. In there select the service Lambda and then filter "By Resource".

![Search for duration](https://dev-to-uploads.s3.amazonaws.com/i/5g38o5am23svd2iq6vfc.png)

Then search for duration and select all. This will include all duration metrics on the graph above. To make more sense of this information in comparison with our spending, we will adjust the graph a little further.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/qdql7hxmhvp0fdepcrle.png)

In the tab Graphed Metrics, use the top right settings to set the overall statistic to Sum and period to 1 Day. Your graph has probably vanished if it is still showing only the last three hours. We will change the period next.

![Adjust graph period](https://dev-to-uploads.s3.amazonaws.com/i/m2ryr98o165nxmdhrt2x.png)

In this post's first picture we can see that Lambda spending started to ramp up on January 14th. Therefore we can set our graph's period to start at January 14th until yesterday (in this case January 29th). We exclude today, because it's still a day in progress.

![Graph with lambda durations](https://dev-to-uploads.s3.amazonaws.com/i/fqa1m59m96l0hnbxrz9h.png)

Now we can see the total duration that each lambda function was running per day. You can see one function going up above 40.000 seconds (11 hours), while all others stay at or below 10.000 seconds (2.8 hours). The function name tells me which function was running for so long, and lets me understand that this was from a recent data migration. The migration has completed and therefore I can exclude this function and the others which were part of the migration. 

![Exclude resources from the graph](https://dev-to-uploads.s3.amazonaws.com/i/dsg5z0y91lvpwysz5xo1.png)

You can exclude certain functions from the graph by clicking their color label. Let's see how the graph looks now.

![Cleaned graph with lambda durations](https://dev-to-uploads.s3.amazonaws.com/i/r8m2sol071esxks5qz73.png)

When you hover over the graph you can see which resources had the biggest duration on a given day.

![Top function durations](https://dev-to-uploads.s3.amazonaws.com/i/dcctzpquq9m17mxtvx2t.png)

As you can see there are the three projects aws-scheduler, contracts-appraisal and market-watch. With this information we can make sure that those projects are tagged properly and we get better information over the next days.

When I checked the projects I noticed that the projects aws-scheduler and contracts-appraisal weren't tagged at all. Three lines in each `serverless.yml` and a `sls deploy` should be enough.

```
provider:
  [...]
  tags:
    department: research
    project: aws-scheduler
```

Check back soon when we use the newly gained information to optimise the expensive lambda functions.

Please note that the cost of a lambda execution consists of both the runtime as well as the memory. For simplicity we excluded the latter in this chapter. Do you know how to combine the duration and memory usage in CloudWatch? Please share it!

## Feedback

Did you find this interesting, wrong or did you miss something? Please let me know here or on [twitter](https://twitter.com/michabahr).