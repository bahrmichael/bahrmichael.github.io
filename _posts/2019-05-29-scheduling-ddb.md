---
layout: post
title: Scheduling irregular AWS Lambda executions through DynamoDB TTL attributes
---

## Introduction

This article describes a serverless approach to schedule AWS Lambda invocations through the usage of AWS DynamoDB TTL attributes and streams. At the time of writing there was no way to schedule an irregular point of time execution of a lambda execution (e.g. “run this function once in 2 days and 10 hours and then again in 4 days”) without [abusing CloudWatch crons](https://forums.aws.amazon.com/thread.jspa?messageID=902484) (see Alternatives for more info).

<iframe src="https://medium.com/media/eec9da0dbe595735d91b5a9caa2c175e" frameborder=0></iframe>

This approach scores with its clean architecture and maintainability. It only requires a function to insert events into a scheduling-table and a function that processes events that hit reach the scheduled point of time. As we build everything on serverless technology, we don’t have to run software upgrades, maintain firewalls and pay for idle time. At low usage it’s practically free and even with higher usage we only really start paying once we schedule **hundreds of thousands of events per day**. Read more in this follow up article:
[**Cost Analysis: Serverless scheduling of irregular invocations**
*In the article “Serverless scheduling of irregular invocations” we used the TTL attribute of DynamoDB to schedule…*medium.com](https://medium.com/@michabahr/cost-analysis-serverless-scheduling-of-irregular-invocations-a1c044957588)

While this approach allows one to schedule an execution for a certain time, it falls short on accuracy. [In our tests](https://medium.com/@michabahr/cost-analysis-serverless-scheduling-of-irregular-invocations-a1c044957588) with a scheduling table holding 100.000 entries, the events appeared in the DynamoDB stream with a delay of up to 28 minutes. According to the [docs](https://docs.aws.amazon.com/de_de/amazondynamodb/latest/developerguide/howitworks-ttl.html) it may take up to 48 hours for especially large workloads. Therefore this approach does not fit, if you require the function to be executed at a certain hour, minute or second. Potential use cases are status updates which run every couple hours or days or non time critical reminders.

The source code for the lambda functions is available at [GitHub](https://github.com/bahrmichael/lambda-scheduling-with-dynamodb).

## Approach

Our approach is to use a lambda function to create scheduling entries. These entries contain the payload for later execution and a time to live attribute (TTL). We then configure the DynamoDB table to delete items after their [TTL has expired](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html) and push those changes, including the old entry, to a stream. The stream is then connected to a lambda function which processes the changes and executes the desired logic. The functions are written with Python 3.7.

The executor function may reschedule events by performing the same logic as the scheduler function.

In this guide we will 
1. setup the table and stream, 
2. the executor which consumes the stream, 
3. a little function to schedule events and 
4. deploy it to AWS with the [serverless framework](https://serverless.com) (version 1.32.0).

You can change the order of the steps, but should keep in mind that the executor requires the stream’s [ARN](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html).

## Table Setup

Start by creating a new DynamoDB table.

![](https://cdn-images-1.medium.com/max/2000/0*Ao5j_LbjG57uLVhD)

If you expect to exceed the [free-tier](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=categories%23alwaysfree&awsf.Free%20Tier%20Categories=productcategories%23database), we recommend switching to on-demand. Please not that exceeding a provisioned capacity may lead to DB operations being rejected, while on-demand is not limited.

![](https://cdn-images-1.medium.com/max/2000/0*CJA4gL70nX8KVwbl)

This will open a dialog where you specify the TTL attribute and activate DynamoDB Streams. We will use the attribute name “ttl”, but you may choose whatever name you like that is not [reserved by DynamoDB](https://docs.aws.amazon.com/de_de/amazondynamodb/latest/developerguide/ReservedWords.html). Click on Continue to create enable the TTL.

![](https://cdn-images-1.medium.com/max/2000/0*rBCGrPkSwz_Yezpj)

Once DynamoDB has created the TTL and stream, you will see the stream details on the table overview. We will later need the “Latest stream ARN” to hook it up to our executor function.

![](https://cdn-images-1.medium.com/max/3200/0*0sEMe2PTOWYC_jUf)

## Executor

Next we write an executor function in Python which consumes the stream events.

<iframe src="https://medium.com/media/906020a4fcc3894cf329c19a32467823" frameborder=0></iframe>

A few things to keep in mind here:

1. Line 2–3: You may receive more than one record. Therefore your function must finish before the [configurable](https://serverless.com/framework/docs/providers/aws/guide/functions/#configuration) [lambda timeout](https://docs.aws.amazon.com/lambda/latest/dg/limits.html) is reached.

1. Line 7–10: You will receive events that are unrelated to the TTL deletion. E.g. when an event is inserted into the table, the *event_name* will be *INSERT*.

1. Line 13: The [record’s structure](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_Record.html) differs from the entry we wrote to the DynamoDB table. You need to access *Records*, then *dynamodb* and finally *OldImage* to get the database entry as it was before deletion. Note that the payload follows DynamoDB’s [AttributeValue](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_streams_AttributeValue.html) schema shown below:

![](https://cdn-images-1.medium.com/max/2504/1*qxR6M44jtLY3rW37XdVW0A.png)

## Scheduler

You may create new entries manually through the DynamoDB management console or through scripts. In this example we will write an AWS Lambda function in Python which creates a new entry.

<iframe src="https://medium.com/media/3c3018ecb3774f3527179ba1ce46e198" frameborder=0></iframe>

Please check that line 8 of the *scheduler* has the table name you specified during the table setup.

All that this function does, is to create a database entry with an id, a payload and a TTL attribute. The payload may be a dict, array or plain value. The ttl is measured in [seconds since the epoch](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html). We use the function *delay()

The *put_item* from line 24 will cause an *INSERT* event to be pushed to the stream. According to line 8 of the *executor* we ignore *INSERT* events.

## Deployment

To deploy the functions, we use the [serverless framework](https://serverless.com). Here is the *serverless.yml*, which specifies the desired resources:

<iframe src="https://medium.com/media/593af6c0e5da00565829f71e3b580935" frameborder=0></iframe>

From line 3 to 12 we specify the provider (AWS), the runtime (python3.7) and grant permissions to our lambda functions. Here we only need write and read access for the scheduling table. You may [extend the roles](https://serverless.com/framework/docs/providers/aws/guide/iam/) depending on what your *scheduler* and *executor* do.

Lines 15 to 16 set up the *scheduler*. This function is not available publicly, but only through the lambda management console. You may extend it to [run regularly](https://serverless.com/framework/docs/providers/aws/events/schedule/) or be available through an [AWS ApiGateway endpoint](https://serverless.com/framework/docs/providers/aws/events/apigateway/).

Lines 17 to 21 set up the *executor*. Through events we specify that the *executor* should be invoked when an event is pushed to the stream. Here you should replace the example with the name of your stream from the table setup.

Once you adjusted the *serverless.yml* to your needs, deploy it by running *serverless deploy* from the folder where the *serverless.yml* and functions are located. This may take a few minutes to complete.

![](https://cdn-images-1.medium.com/max/2164/1*-CBNPJi0Zob528dMvnqz-g.png)

## Test

Once serverless completes the deployment, head over to the [lambda management console](https://console.aws.amazon.com/lambda/home?region=us-east-1#/applications/lambda-scheduler-dev) and click on *ScheduleLambdaFunction*.

![](https://cdn-images-1.medium.com/max/4444/1*d0CpAQ_MDDwjjb2-_HkFFw.png)

In the top right corner you can then configure a test event.

![](https://cdn-images-1.medium.com/max/2996/1*WAZXz6fKvch1n_1mQeDwHQ.png)

As we don’t evaluate the input of the scheduler function the default *Hello World* event is sufficient.

![](https://cdn-images-1.medium.com/max/3200/1*LCRPw4-EK3EOrd9EODiOyA.png)

Create the test event and then execute it by clicking on Test.

![](https://cdn-images-1.medium.com/max/5156/1*F_M-jZjclb5X9vUlxlKEHQ.png)

Now head over to the [DynamoDB management console](https://console.aws.amazon.com/dynamodb/home#tables:) and open the items of your table. When you hover over the TTL attribute you will see an overlay which tells you when the entry is supposed to be deleted (remember that the deletion may be delayed).

![](https://cdn-images-1.medium.com/max/3680/1*_JEWnU8xCte7MiLVIrimBw.png)

Now we have to wait until DynamoDB deletes the entry. As DynamoDB will probably take a few minutes we are in no rush here. Switch back to the console where you deployed the serverless application and run the command *serverless logs -f executor -t* to listen for new logs of the *executor* function. Get yourself a drink and a snack as this will probably take 10 to 15 minutes for a small table. If you created 10.000.000.000 entries, then you might have to wait longer.

![](https://cdn-images-1.medium.com/max/2504/1*qxR6M44jtLY3rW37XdVW0A.png)

That’s it! We invoked a single lambda execution without the use of rate or cron.

## What’s next?

From here you can play around with the [example](https://github.com/bahrmichael/lambda-scheduling-with-dynamodb) and try a couple things:

1. Modify the delay e.g. by using Python’s *random.randint()

1. If you are using this approach to schedule a lot of executions (read: more than 100 per day), you should look into optimising the resources of your functions away from the default of 1GB RAM. You can do this by glancing over the logs and looking for the RAM usage, then estimating a number that will fit for all those executions or by using the [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) tool to estimate the perfect fit.

1. Let the executor reschedule the event by adding another database entry for a future execution. My project does status updates and then reschedules the execution if the desired status did not match yet.

1. Extend the approach for multiple executors, by specifying another field with ARN of the desired executor, and then writing a new delegating executor which invokes the desired ARN.

## Housekeeping

Should you decide to use this approach for your project, make sure to check off a few housekeeping tasks:

1. Monitor the delay between the defined TTL and the actual execution. The example code prints a log entry that you can use to visualize the delays.

1. Optimize the granularity of the permissions in the *serverless.yml* file. Aim for the least amount of privileges.

1. Define a [log retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) for the generated CloudWatch log groups. You probably don’t want to rake up costs as time passes and log storage increases.

1. [Tag the resources for cost analysis](https://aws.amazon.com/premiumsupport/knowledge-center/tags-billing-cost-center-project/) so you are later able to identify where major cost producers come from.

1. Set up an [AWS Budget](https://aws.amazon.com/aws-cost-management/aws-budgets/) to prevent billing surprises.

## Alternatives

The approach above is not the only way to schedule irregular executions, but feels the cleanest to me if you need to pass a payload to the lambda function.

### CloudWatch PutRule API

If you have a limited amount of scheduled invocations, you can use the [CloudWatch PutRule API](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) to [create a one time cron execution](https://forums.aws.amazon.com/thread.jspa?messageID=902484).

    cron(23 59 31 12 ? 2019)

This cron will execute on the 31st of December 2019 at 23:59.

The downside of this approach is that you can’t create more than [100 rules per account](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/cloudwatch_limits_cwe.html) and have to clean up expired rules.

### EC2 Scheduler

You can spin up an EC2 that runs a scheduler software, but this requires additional costs for the EC2 instance and does not fit the serverless goal of this article.

## Summary

In this article we looked at an approach to schedule irregular lambda invocations and learned about its implementation and limitations. If you use this approach, then beware of the delays and monitor them.

What are your thoughts? Do you have a better approach or know how to achieve this on a different cloud provider? Share it!

Follow me on [Twitter](https://twitter.com/michabahr) for more experiments in and around cloud computing!

## Further Reading

* [Analysis of DynamoDB’s TTL delay](https://medium.com/@michabahr/cost-analysis-serverless-scheduling-of-irregular-invocations-a1c044957588)

* [Cost Analysis: Serverless scheduling of irregular invocations](https://medium.com/@michabahr/cost-analysis-serverless-scheduling-of-irregular-invocations-a1c044957588)

* Yan Cui’s take on [DynamoDB TTL as an ad-hoc scheduling mechanism](https://medium.com/theburningmonk-com/dynamodb-ttl-as-an-ad-hoc-scheduling-mechanism-bda119116887)

* [More suggestions on scheduling mechanism](https://medium.com/@zaccharles/there-is-more-than-one-way-to-schedule-a-task-398b4cdc2a75) by Zac Charles

* [Scheduling with State Machines](https://aws.amazon.com/getting-started/tutorials/scheduling-a-serverless-workflow-step-functions-cloudwatch-events/)

* [Using AWS Lambda with CloudWatch Events](https://docs.aws.amazon.com/lambda/latest/dg/with-scheduled-events.html)

* [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)
