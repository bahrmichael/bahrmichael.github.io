---
layout: post
title: Analysis of DynamoDB’s TTL delay
backgroundUrl: "https://images.unsplash.com/photo-1563371386-187140095137?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&q=80"
---

In the article “[Serverless scheduling of irregular invocations](https://medium.com/@michabahr/scheduling-irregular-aws-lambda-executions-through-dynamodb-ttl-attributes-acd397dfbad9)” we used the TTL attribute of DynamoDB to schedule irregular lambda executions. This follow up article takes a look at the delays between the specified TTL and the actual deletion.

## Setup

To measure the delay we specified an evenly distributed TTL on each database entry. Upon deletion that entry is pushed into a stream which is then consumed by a lambda function. That lambda function prints the delay between the old record’s specified TTL and the current time.

```python
def handle(event, context):
    print('Received %d records' % len(event['Records']))
    for record in event['Records']:

        # as we're using ttl to schedule executions, we do not care about inserts or updates,
        # only about removes which happen after the ttl is hit
        event_name = record['eventName']
        if event_name != 'REMOVE':
            print('Skipping %s' % event_name)
            continue

        # note that this image is in DynamoDB style, not a regular python object and needs to be converted accordingly
        old_image = record['dynamodb']['OldImage']
        print(old_image['payload'])
```

## Tests

We ran two tests: The first with 1.000 entries and the second with 100.000 entries.

{% gist 369dd06684a650e665a3705c51ad590d %}

As you can see in the small example, most events are triggered with a delay of 10.5 to 13 minutes. None were on time with the earliest being 8 minutes late.

{% gist 69707f51c3418e53512a1b7b2f2df4dd %}

The numbers rise for the large example with most events being triggered with a delay of 17 to 25 minutes.

As you may notice we didn’t wait for all entries to be deleted, but one can assume that the delay would decrease as the table shrinks.

## Summary

Entries in a table with 1.000 entries were usually deleted within 13 minutes whereas entries in tables with 100.000 entries took longer with a maximum delay of 28 minutes.

Officially DynamoDB deletes the entries [within a 48h](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/howitworks-ttl.html) after the TTL expires. The delay appears to correlate with the tables size as shown above.

As the results show you should monitor the delays to see if they remain acceptable as your workload grows.

Follow me on [Twitter](https://twitter.com/michabahr) for more experiments in and around cloud computing!
