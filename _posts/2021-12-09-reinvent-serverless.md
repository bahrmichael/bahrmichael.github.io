---
layout: post
title: "My 3 favourite re:invent serverless announcements"
description: "Once a year AWS holds their re:invent conference where they announce new features and services. In this article I will go over my favourite serverless announcements."
backgroundUrl: "https://images.unsplash.com/photo-1540575467063-178a50c2df87?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&q=80"
---

Once a year AWS holds their re:invent conference where they announce new features and services. In this article I will go over my favourite serverless announcements.

## AWS Lambda Event Filters for DynamoDB Streams

[Link to announcement](https://aws.amazon.com/about-aws/whats-new/2021/11/aws-lambda-event-filtering-amazon-sqs-dynamodb-kinesis-sources/)

Gone are the days when we have 10 Lambda functions attached to a stream, with every one having filtering logic in the beginning.

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Anyone played with or put into prod the new filtered <a href="https://twitter.com/hashtag/DynamoDB?src=hash&amp;ref_src=twsrc%5Etfw">#DynamoDB</a> stream in Lambda Functions? If you have not created one, it’s really nice! <a href="https://twitter.com/hashtag/serverless?src=hash&amp;ref_src=twsrc%5Etfw">#serverless</a> <a href="https://twitter.com/hashtag/NoSQL?src=hash&amp;ref_src=twsrc%5Etfw">#NoSQL</a><br><br>Navigate to your Lambda Function -&gt; Configuration -&gt; Triggers and add new. Then go to additional settings.</p>&mdash; Kirk Kirkconnell (@NoSQLKnowHow) <a href="https://twitter.com/NoSQLKnowHow/status/1466099088764518402?ref_src=twsrc%5Etfw">December 1, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

We may now specify filters that Lambda applies before calling our function.

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Confirmed with <a href="https://twitter.com/ajaynairthinks?ref_src=twsrc%5Etfw">@ajaynairthinks</a> that the Lambda Service eats the polling cost, so you only get charged for what makes it into your function.</p>&mdash; Jeremy Daly (@jeremy_daly) <a href="https://twitter.com/jeremy_daly/status/1466171465733062657?ref_src=twsrc%5Etfw">December 1, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

In applications where there are many more Lambda functions attached to the same stream, this can be a major cost savings opportunity.
Imagine having 100 functions, and only one actually does something for each change. The cheapest code is the one that doesn't run.

If you're using the Serverless Framework, [there's already support for this filtering](https://dev.to/aws-builders/new-dynamodb-streams-filtering-in-serverless-framework-3lc5).

## New Storage Tiers for S3 and DynamoDB

I've written about archiving data to save on storage cost before: [Archive your AWS data to reduce storage cost](https://bahr.dev/2020/08/07/archiving-data/)

That article now needs an update, as there are more options, and more automations. Below you can find three major changes for long-lived data.

### S3 - Glacier

[Link to announcement](https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-s3-glacier-instant-retrieval-storage-class/)

There's a new S3 Glacier Instant Retrieval with "the same throughput and milliseconds access as the S3 Standard".
If you don't need fast retrieval and stick with the previous tier, you pay 10% less for storage.
The previous tier is now called S3 Glacier Flexible Retrieval and offers "retrieval in minutes or free bulk retrievals in 5-12 hours".

This is great if you have data like medical images or tax receipts. You rarely need to access them, but if so, you need them immediately.
This is now possible with Glacier which has a far lower cost than the standard S3 tiers.

S3 Glacier Deep Archive did not change and offers "data retrieval from 12-48 hours".

### S3 - Intelligent Tiering

[Link to announcement](https://aws.amazon.com/about-aws/whats-new/2021/11/s3-intelligent-tiering-archive-instant-access-tier/)

If you don't know the access patterns for your S3 data, then the Intelligent Tiering comes in handy.
Intelligent Tiering picks the best storage tier for your objects, and now also includes the archived tier.

This means that you can tell S3 to monitor access patterns. If it finds a better storage tier for an object, S3 will move the object.
This can lead to cost reduction for you, but keep in mind that you get charged for monitoring access patterns.

### DynamoDB

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Well, looky there. AWS just cut costs by 60% for infrequently accessed data on <a href="https://twitter.com/dynamodb?ref_src=twsrc%5Etfw">@dynamodb</a>.<br><br>That&#39;s great news for <a href="https://twitter.com/totogi?ref_src=twsrc%5Etfw">@totogi</a> who&#39;s built their charging system on <a href="https://twitter.com/dynamodb?ref_src=twsrc%5Etfw">@dynamodb</a>! WOO HOO!<a href="https://twitter.com/hashtag/singletabledesign?src=hash&amp;ref_src=twsrc%5Etfw">#singletabledesign</a> <a href="https://twitter.com/hashtag/reinvent?src=hash&amp;ref_src=twsrc%5Etfw">#reinvent</a><a href="https://t.co/zb7EAcIKrJ">https://t.co/zb7EAcIKrJ</a></p>&mdash; Danielle Royston (@TelcoDR) <a href="https://twitter.com/TelcoDR/status/1466104180813139968?ref_src=twsrc%5Etfw">December 1, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

I heard you like not paying much for data that you don't use often. DynamoDB joins the party with a the Infrequent Access table class.
This is available for On-Demand capacity only.

By switching from the default class to the infrequent access class, you get the following price changes:

* Write Capacity Units: +25% ($1.25 per million to 1.56$ per million)
* Read Capacity Units: +25% ($0.25 per million to $0.31 per million)
* GB per month: -60% ($0.25 per GB-month to $0.1 per GB-month)

There seems to be no constraint to switching table classes, so you can flip it back and forth if you're unsure if the new tier will save you money.

## Kinesis On-Demand

[Link to announcement](https://aws.amazon.com/about-aws/whats-new/2021/11/amazon-kinesis-data-streams-on-demand/)

Kinesis has a new On-Demand mode where you're charged by the hour and for throughput. Gone are the days of having to manage shards.

> In on-demand mode, pricing is based on the volume of data ingested and retrieved along with a per-hour charge for each data stream in your account [...]

But is it really serverless if you're charged by the hour, not only for usage?

{% raw %}
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Does it scale to zero?</p>&mdash; Jeremy Daly (@jeremy_daly) <a href="https://twitter.com/jeremy_daly/status/1465771902224322562?ref_src=twsrc%5Etfw">November 30, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

This announcement might not follow everyone's definition of serverless. However, I'm glad that I don't have to configure and manage stream shards.

> You do not need to specify [...] throughput [...]. Kinesis Data Streams instantly accommodates your workloads as they ramp up or down.

While an idle cost of $30 per month is not great for projects that run at nearly 0$ per month, I still welcome this new option.
So far the operational effort to manage throughput has been a major factor in deciding against Kinesis and for SQS. Now that blocker is gone.

## Conclusion

Not many new features this year, but continuous improvement to existing services.
This gives me the feeling that engineers can make great improvements to existing services (that also help with their promotions),
without [having to launch a new feature](https://www.lastweekinaws.com/blog/the-google-disease-afflicting-aws/).

To lower spending and better services!
