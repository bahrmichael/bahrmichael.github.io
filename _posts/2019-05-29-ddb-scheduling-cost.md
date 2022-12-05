---
layout: post
title: "Cost Analysis: Serverless scheduling of irregular invocations"
backgroundUrl: "https://images.unsplash.com/photo-1513596846216-48ae70153834?ixlib=rb-1.2.1&auto=format&q=80&fit=crop"
---

> Update: In November 2022 AWS has released the EventBridge Scheduler. It does what I expect from a serverless scheduler, and has a free tier of 14 million invocations per month.

In the article [Serverless scheduling of irregular invocations](https://bahr.dev/2019/05/29/scheduling-ddb/) we used the TTL attribute of DynamoDB to schedule irregular lambda executions. This follow up article takes a look at the operational cost of that approach.

## Service Prices

In the mentioned article we used only DynamoDB and Lambda. All costs here are taken from us-east-1 (North Virginia). Below you can see how active the scheduling was over a 24h period.

![Activity of the scheduler in 5 minute intervals](https://bahr.dev/pictures/ddb-scheduling-cost-graph-1.png)

### DynamoDB

The relevant pricing aspects of DynamoDB for our example are
1. read requests,
2. write requests,
3. data storage and
4. streams.

In our example we used on-demand pricing which charges $1.25 per million write requests and $0.25 per million read requests. Let’s take a look at read requests first.

![Consumed Read Capacity Units](https://bahr.dev/pictures/ddb-scheduling-cost-graph-2.png)

The spikes in the picture above were from table size scans. During regular operation there are no read requests, therefore we don’t have to calculate read capacity, only write capacity. We do however pay for the DynamoDB stream which we will look into later.

The cost for write capacity depends on how many entries you write and how big those are.
> DynamoDB charges one write request unit for each write (up to 1 KB)

Measure the size of your scheduling entries and adjust your calculations accordingly.

![Consumer Write Capacity Units](https://bahr.dev/pictures/ddb-scheduling-cost-graph-3.png)

In our 24 hour example we scheduled ~280.000 entries each consisting of a uuid, the TTL attribute and a target id (number). This results in the 278.000 write capacity units shown above.
> $1,25 / 1.000.000 * 278.000 = $0,3475

If this amount is scheduled every day it will lead to a monthly cost of ~$10.

Looking at a smaller example of 200 scheduling entries every half our we only reach 9.600 write capacity units per day. Following the formula above the resulting cost per month would be $0,36.

### Data storage

We are not taking a deep dive on this as the first 25 GB stored per month are free and after that each GB only costs $0,25. The scheduling table is not supposed to be your business warehouse, but should only hold enough information for the triggered function to do its job.

### Streams

DynamoDB starts to charge $0,02 per 100.000 read requests *after* exceeding the free first 2.500.000 read requests of the free tier. One read request may contain up to 4 KB. To estimate the costs we can look at how many lambda functions were executed during a given timespan.

In our example, during the first 24 hours ~300.000 entries reached their TTL and were pushed to the stream. Over the course of a month that would result in a maximum of 7.200.000 stream reads. Subtracting the free tier of 2.500.000 read requests we have to pay for 4.700.000. At $0,02 per 100.000 read requests this results in a cost of $9,4.

Looking at the smaller example of 200 scheduling entries every half our and assuming they have to be rescheduled two times, we end up with the 576.000 read requests. This is within the free tier of 2.500.000.

Note that the costs for stream reads might be even lower as each read request may return multiple records.

### Lambda

You can analyse the logs to get a understanding of your lambdas’ resource usage. Another option is to look into the [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/), drilling down the costs with tags.

`aws logs filter-log-events --log-group-name /aws/lambda/my-log-group --filter-pattern “Billed Duration” --start-time 1559050096000 | grep “Billed Duration” > resource_usage.txt`

In this section we are looking 180.000 consumer executions which stayed within the minimum of 128 MB RAM and showed the following distribution of billing durations:

{% gist 943c59a4e19b2b69d6cd0e2c24186e9c %}

*No plot here because Google Docs crashed :(*

AWS Lambda offers the first 1.000.000 requests and the first 400.000 GB seconds for free. After that it charges [$0.0000166667 for every GB second](https://aws.amazon.com/lambda/pricing/). A GB second is calculated as one GB of RAM for one second. This means that when we only use 128 MB of RAM we actually have 3.200.000 free tier seconds per month. At 128 MB RAM and a median billed duration of 200 ms we have 16.000.000 executions before we start paying. This results in $0 for our 180.000 executions.

### Optional: Traffic

Traffic pricing only applies if you request resources outside of the current region. [As of 2010](https://aws.amazon.com/blogs/aws/aws-data-transfer-prices-reduced/) you pay $0.15 per GB.

### Optional: API Gateway

API Gateway can be a major cost driver if used at scale. While our example didn’t make use of API Gateway, you may decide to use it to as an endpoint for scheduling. It [starts with 1.000.000 free requests](https://aws.amazon.com/api-gateway/pricing/) and then rapidly takes off with $3.5 per 1.000.000 requests. Once you exceed 333 million requests (or $1.165,5) costs are lowered to $2.8 per 1.000.000 requests. If you don’t use any of the fancy AWS API Gateway features and only need to route requests to a lambda function you should consider [replacing API Gateway with an Application Load Balancer](https://serverless-training.com/articles/save-money-by-replacing-api-gateway-with-application-load-balancer/).

## Summary

At moderate usage it is possible to stay within the “basically free” area. Significant costs only appear once you schedule more than 100.000 executions per day, process each entry many times per day or have resource intensive lambda functions. Even then you should be able to stay within $50 for moderate enterprise workloads.

As always make sure to monitor your costs and set up budgets so that you get notifications if costs rise unexpectedly.

## Further Reading

* [Replacing AWS Gateway with AWS ALB](https://serverless-training.com/articles/save-money-by-replacing-api-gateway-with-application-load-balancer/)

* [AWS “Not So” Simple Monthly Calculator](https://calculator.s3.amazonaws.com/index.html)

* [AWS Cost Management](https://aws.amazon.com/aws-cost-management/)

* [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)

* [AWS DynamoDB Pricing](https://aws.amazon.com/dynamodb/pricing/)

* [AWS Traffic Pricing](https://aws.amazon.com/blogs/aws/aws-data-transfer-prices-reduced/)

* [AWS API Gateway Pricing](https://aws.amazon.com/api-gateway/pricing/)
