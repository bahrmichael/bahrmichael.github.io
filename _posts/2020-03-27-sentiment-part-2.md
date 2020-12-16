---
layout: post
title: "Stock Sentiment Analysis - Part 2: Analysing the sentiment"
backgroundUrl: "https://images.unsplash.com/photo-1535320903710-d993d3d77d29?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&q=80"
---

In this two part article I will show you how to build an app, that collects people's opinions about companies and how to turn that into sentiments. Disclaimer: Trade at your own risk!

[Part 1](https://bahr.dev/2020/03/27/sentiment-part-1/)

---

**Warning**: Only deploy this if you have set up budget alarms and understand the spending potential of this solution! Check the section Cost Analysis for more details.

---

We left off with collecting raw tweets in our DynamoDB table. Now it's time to understand if the tweets about a given company are rather positive or negative. We will add another Lambda which is invoked when we find new tweets. This Lambda then asks AWS Comprehend for the tweet's sentiment, which is a score of how positive, neutral or negative the text was.

Here's an example tweet I collected. Do you think the text is rather negative or positive?


> I wish I were as positive about anything in life as Ross is about $TSLA.


## Streams

The [serverless framework](https://serverless.com) has a simple way to attach a lambda to a [DynamoDB stream](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html).

```yaml
functions:
  tweetAnalyzer:
    handler: tweetAnalyzer.handle
    events:
      - stream:
          arn: arn:aws:dynamodb:REGION:ACCOUNT_ID:table/Tweets/stream/DATE
          batchSize: 25
```

With this snippet of code the function `tweetAnalyzer` will be invoked when we add new tweets to the table. The `batchSize` of 25 allows us to process multiple tweets at once. I chose 25 because that's the maximum amount of texts we can pass to AWS Comprehend per request ([BatchSizeLimitExceededException](https://docs.aws.amazon.com/comprehend/latest/dg/API_BatchDetectSentiment.html)).

Our function is invoked when an entry in our DynamoDB table changes. As I've defined the stream to only contain the key of the entry, we first have to load the entry, then send it to AWS Comprehend and finally update the entry in our table. You can find the source code on [GitHub](https://github.com/bahrmichael/twitter-sentiment-analyzer).

The event passed from the stream to our function contains a list of `Records` which each hold the entry's key in `item['dynamodb']['Keys']['id']['N']`. If your key is not a number, you have to adjust the `['id']['N']`.

```python
import boto3
table = boto3.resource('dynamodb').Table('Tweets')

tweets = []
for item in event.get('Records', []):
    item_id = int(item['dynamodb']['Keys']['id']['N'])
    tweet = table.get_item(Key={'id': item_id}).get('Item', None)
    if 'sentiment_score' not in tweet:
        tweets.append(tweet)
```

This loop loads all the entries which don't have a `sentiment_score` yet. We need to filter because the events will fire when an entry in the table _changes_, which is also the case when we add the sentiment score. Let's continue with adding that score.

## Sentiments

The Comprehend API expects a list of texts. As we saved some more meta information along the tweets' text, we have to extract the raw text first. Don't change the tweets' order here as AWS Comprehend returns the results along with their `Index` from the input, which we will map back the results to the original tweets.

```python
text_list = list(map(lambda tweet: tweet['text'], tweets))
```

In the next step we send the texts to Comprehend. Note that the `text_list` must not have more than [25 entries](https://docs.aws.amazon.com/comprehend/latest/dg/API_BatchDetectSentiment.html). You must also specify a language. As we've previously told the Twitter API to only return English tweets (by best effort), we will use `en` here.

```python
import boto3
comprehend = boto3.client('comprehend')

comprehend_response = comprehend.batch_detect_sentiment(
    TextList=text_list,
    LanguageCode='en'
)
```

Each of the results contains an `Index` which is the index of the item in the `text_list`. We use that information to map the result back to our DynamoDB entries.

```python
from decimal import Decimal

for entry in comprehend_response.get('ResultList', []):
    tweet = tweets[entry['Index']]
    tweet['sentiment_score'] = json.loads(json.dumps(entry['SentimentScore']), parse_float=Decimal)
    table.put_item(Item=tweet)
```

Note that because DynamoDB doesn't like float parameters, we have to convert the floats to Decimals.

Let's see how Comprehend rated our example from above. Do you think that Comprehend catched the sarcasm?

```json
{
    "neutral": 0.07,
    "positive": 0.88,
    "negative": 0.03,
    "mixed": 0.00
}
```

While the score may be wrong for a couple of tweets, the overall scoring over thousands and millions of tweets will average out the wrong ones.

The app now collects sentiment insights for you. Don't stop reading here, as this might become expensive!

## Cost Analysis

Each tweet will require one sentiment analysis and one DynamoDB write operation. Again the time span is a whole year of operation.

With [on-demand pricing](https://aws.amazon.com/dynamodb/pricing/on-demand/) DynamoDB charges $1.25 per million write request units and $0.25 per million read request units. With the worst case of 100 new tweets every 10 minutes, we're looking at a total of 100x6x24x365 = 5256000 WCUs or $6.57 per year.

The [cost of AWS Comprehend](https://aws.amazon.com/comprehend/pricing/) is a bit more intense.

Tweets vary in length, but for the worst case we assume that each tweet uses up the full 280 characters. If we collect 100 tweets every 10 minutes, we are going collect 100x6x24x365 = 5,256,000 tweets per year. Over the last year I did however only collect 1,500,000 tweets.

> "Amazon Comprehend requests are measured in units of 100 characters, with a 3 unit (300 character) minimum charge per request."

Sentiment analysis is priced at $0.0001 per unit. The price to analyze 5,256,000 tweets with 280 characters or 3 units each is 5,256,000 x $0.0001 x 3 = **$1,576.8**. This is expensive for a hobby project, so please make sure you tag your resources appropriately and think twice before you run this as a hobby project.

There is a free tier for Comprehend, which for sentiment analysis will "cover only the main analysis [...]. But after you analyze the text, the system automatically calls different APIs [...]. These automatic calls [...] are not covered by the Free Tier [...]" (Source: AWS Support). Because of this I started seeing Comprehend charges before the full 50k free units were used up. The team of Comprehend has already received the feedback so they can improve the website.

## Further development

If you want to take this approach and push it forward, I suggest to start building a container/VM based solution to pull tweets from Twitter. [AWS Fargate](https://aws.amazon.com/fargate/) lets you run containers without managing the servers below.

Know any other sentiment APIs? How do they compare in pricing? Try to attach one of them instead.

## Conclusion

Analyzing the sentiment can be achieved with one function, but only scales if your workload is small or your credit card big. Please make sure you have budget alarms set up if you want to deploy this yourself, this is not cheap!
