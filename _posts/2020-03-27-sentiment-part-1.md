---
layout: post
title: "Stock Sentiment Analysis - Part 1: Collecting opinions"
---

In this two part article I will show you how to build an app that collects people's opinions about companies and how to turn that into sentiments. Disclaimer: Trade at your own risk!

As for technologies my current go to stack is AWS serverless tech and deployment with the Serverless Framework. This article assumes that you are familiar with both.

## Collecting opinions

Many platforms have APIs that let us collect opinions. The prime example is probably Twitter where everyone screams into the forest. We will start by setting up an app and collecting the raw data.

In order to collect data from Twitter, you have to create [a developer app](https://developer.twitter.com/apps) and generate [oauth1 keys](https://developer.twitter.com/en/docs/basics/authentication/oauth-1-0a). You can do all of that through the browser. Store the details in the `config.<STAGE>.json` file. The value for `<STAGE>` is either `dev` or whatever you provided with the `--stage` parameter.

![Twitter Access Keys](https://dev-to-uploads.s3.amazonaws.com/i/ekekl9klmpkiqt4ibcgh.png)

Next we set up a serverless app. With serverless technologies like AWS Lambda we don't need to worry about creating and maintaining serves, and it's also super cheap. You can find the source code [on github](https://github.com/bahrmichael/twitter-sentiment-analyzer) or learn how to create a serverless app at [serverless.com](https://serverless.com).

Our first function will run in a regular interval. I chose 10 minutes because that was usually enough time for for 10-50 new tweets to appear. Learn more about the scheduling options by visiting the [CloudWatch docs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html).

```yaml
functions:
  tweetCollector:
    handler: tweetCollector.handle
    events:
      - schedule: rate(10 minutes)
```

The function needs to authenticate with Twitter, load new tweets and then store these in our table for later processing.

The first step is pretty simple with [Tweepy](https://tweepy.org). In the following snippet we load the oauth1 keys from environment variables and then authenticate. Our script will abort if the authentication fails. You can find the full source code on [Github](https://github.com/bahrmichael/twitter-sentiment-analyzer).

```python
import tweepy

auth = tweepy.OAuthHandler(os.environ['CONSUMER_KEY'], os.environ['CONSUMER_SECRET'])
auth.set_access_token(os.environ['ACCESS_TOKEN'], os.environ['ACCESS_TOKEN_SECRET'])

api = tweepy.API(auth)
if not api:
    print("Can't Authenticate")
    return
```

As I'm running this on AWS Lambda, search queries are a better fit than the Twitter streaming API. The following search query allows us to define a search term, how many tweets we want to load per query as well as a max_id and since_id for pagination.

```
new_tweets = api.search(q=search_query, lang='en', count=tweets_per_query, max_id=str(max_id - 1), since_id=since_id)
```

To store the data I'm using a [DynamoDB table](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SampleData.CreateTables.html). As this query may return tweets that we already have, we check if a tweet already exists before writing it. We could just overwrite the tweets, but this would leader to higher DynamoDB spending. As a rule of thumb [a read costs](https://aws.amazon.com/dynamodb/pricing/on-demand/) 1/5th of a write operation.

```python
for tweet in new_tweets:
    existing_tweet = table.get_item(Key={'id': tweet._json['id']}).get('Item', None)
    if existing_tweet is None:
        table.put_item(
            Item={
                    'id': tweet._json['id'],
                    'created_at': tweet._json['created_at'],
                    'text': tweet._json['text'],
                    'query': search_query
                }
            )
        count += 1
```

Before you deploy, make sure to fill in the configuration. The config is documented in the [readme](https://github.com/bahrmichael/twitter-sentiment-analyzer).

Finally deploy the app with `sls deploy` and let the tweet collection begin. Keeping in mind that the schedule only fires every 10 minutes, check the logs for errors if no tweets arrive. The most likely errors are missing AWS permissions or bad Twitter keys. The first new tweet in our table shows that our collector is up and running!

```
START RequestId: 6aad9d8e-aa2c-422c-a471-6c7f9254c919 Version: $LATEST
Downloaded 100 tweets
Saved tweets: 17
END RequestId: 6aad9d8e-aa2c-422c-a471-6c7f9254c919
REPORT RequestId: 6aad9d8e-aa2c-422c-a471-6c7f9254c919	Duration: 1121.61 ms	Billed Duration: 1200 ms	Memory Size: 1024 MB	Max Memory Used: 98 MB
```

## Cost Analysis

To understand the **yearly** cost of this stack, we will look at two parts: The Lambda function and the DynamoDB table.

### Lambda

The collector function runs once every 10 minutes. This results in 6x24x365 = 52560 invocations per year. [Lambda charges](https://aws.amazon.com/lambda/pricing/) $0.20 per 1M requests, so we're looking at $0.1 per year. Additionally Lambda charges $0.0000166667 for every GB-second. A GB-second is a Lambda with 1024MB RAM running for one second. When our function runs for 2 seconds each 10 minutes, it will use 2x6x24x365 = 105120 GB-seconds. That's another $1.75 per year.

The [free tier](https://aws.amazon.com/free/) is likely to cover all of that.

### DynamoDB

With [on-demand pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/) DynamoDB charges $1.25 per million write request units and $0.25 per million read request units. This means that the worst case for spending is all new tweets and none that we can skip. With 100 new tweets every 10 minutes, we're looking at a total of 100x6x24x365 = 5256000 WCUs or $6.57 per year.

If you exceed the free 25GB per month, then DynamoDB will charge you $0.25 for every additional GB. My table with 1.5m tweets weighs ~270MB.

You can additionally lower the cost by switching to provisioned mode, where DynamoDB offers [25 WCUs and 25 RCUs for free](https://aws.amazon.com/dynamodb/pricing/provisioned/).

### Total

In total that's $0.1 + $1.75 + $6.57 = $8.42 per year.

Note that CloudWatch will charge you too, should you exceed the free tier.

## Conclusion

Collecting data is fairly simple and very cheap. But will it be the same if we monitor 100 companies? Feel free to test that by using the source code on [GitHub](https://github.com/bahrmichael/twitter-sentiment-analyzer).

In the [part 2](https://dev.to/michabahr/stock-sentiment-analysis-part-2-analysing-the-sentiment-28ig) of this article we will use sentiment analysis to understand if a tweet is positive, neutral or negative.