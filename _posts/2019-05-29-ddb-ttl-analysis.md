---
layout: post
title: Analysis of DynamoDB’s TTL delay
---

In the article “[Serverless scheduling of irregular invocations](https://medium.com/@michabahr/scheduling-irregular-aws-lambda-executions-through-dynamodb-ttl-attributes-acd397dfbad9)” we used the TTL attribute of DynamoDB to schedule irregular lambda executions. This follow up article takes a look at the delays between the specified TTL and the actual deletion.
[**Scheduling irregular AWS Lambda executions through DynamoDB TTL attributes**
*Introduction*medium.com](https://medium.com/@michabahr/scheduling-irregular-aws-lambda-executions-through-dynamodb-ttl-attributes-acd397dfbad9)

## Setup

To measure the delay we specified an evenly distributed TTL on each database entry. Upon deletion that entry is pushed into a stream which is then consumed by a lambda function. That lambda function prints the delay between the old record’s specified TTL and the current time.

<iframe src="https://medium.com/media/906020a4fcc3894cf329c19a32467823" frameborder=0></iframe>

## Tests

We ran two tests: The first with 1.000 entries and the second with 100.000 entries.

<iframe src="https://medium.com/media/e079d84279ac017f924882922c2829e5" frameborder=0></iframe>

As you can see in the small example, most events are triggered with a delay of 10.5 to 13 minutes. None were on time with the earliest being 8 minutes late.

<iframe src="https://medium.com/media/0a6da5861d5d4c8eb0b0ca82d5dbcf45" frameborder=0></iframe>

The numbers rise for the large example with most events being triggered with a delay of 17 to 25 minutes.

As you may notice we didn’t wait for all entries to be deleted, but one can assume that the delay would decrease as the table shrinks.

## Summary

Entries in a table with 1.000 entries were usually deleted within 13 minutes whereas entries in tables with 100.000 entries took longer with a maximum delay of 28 minutes.

Officially DynamoDB deletes the entries [within a 48h](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html) after the TTL expires. The delay appears to correlate with the tables size as shown above.

As the results show you should monitor the delays to see if they remain acceptable as your workload grows.

Follow me on [Twitter](https://twitter.com/michabahr) for more experiments in and around cloud computing!
