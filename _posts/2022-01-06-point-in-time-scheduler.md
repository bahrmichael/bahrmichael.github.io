---
layout: post
title: "Announcing the Point In Time Scheduler Public Preview"
description: "With the Point In Time Scheduler developers can specify when and how their endpoint should be invoked. They can do so for any time into the future, be it minutes, weeks, or years. You only pay for what you use, and don't have to waste engineering time operating a scheduler."
backgroundUrl: "https://images.unsplash.com/photo-1558469070-b0bb906830a2?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&q=80"
---

Back in 2019 we've seen a lot of articles covering mechanisms for serverless ad-hoc scheduling. It's clear that developers
want to schedule events while benefiting from serverless principles like pay-per-usage, scaling to zero, and not having to run yet
another cluster yourself.

We use the term ad hoc scheduling for irregular point in time invocations, e.g. one in 32 hours and another one in 4 days.

![Irregular events on a timeline](https://bahr.dev/pictures/serverless-scheduler-1.png)

Yan Cui wrote about [DynamoDB's TTL](https://theburningmonk.com/2019/03/dynamodb-ttl-as-an-ad-hoc-scheduling-mechanism/),
[StepFunctions](https://theburningmonk.com/2019/06/step-functions-as-an-ad-hoc-scheduling-mechanism/) and
[CloudWatch](https://theburningmonk.com/2019/05/using-cloudwatch-and-lambda-to-implement-ad-hoc-scheduling/).
Paul Swail gave an overview of [AWS scheduling primitives](https://serverlessfirst.com/aws-scheduling-primitives/),
and Zac Charles [showed many ways one can schedule events](https://zaccharles.medium.com/there-is-more-than-one-way-to-schedule-a-task-398b4cdc2a75).
I joined the party with the [Serverless Scheduler](https://acloudguru.com/blog/engineering/serverless-scheduler).

Some people use ad hoc scheduling to move workloads into the night, others for an exponential backoff that can span
hours, days or weeks. You can probably come up with many more use cases.

There are some properties of existing AWS services that may help you:

* If you only need up to 15 minutes of delay, use SQS' DelaySeconds.
* If you need up to a year and are okay with StepFunctions' higher price, use StepFunction's Wait task.
* If you're okay with a delay of up to 48 hours, use DynamoDB's time to live mechanism.

Didn't find a match or don't want to operate it yourself? Then this announcement is for you!

**Today I'm proud to announce the public preview of the [Point In Time Scheduler](https://point-in-time-scheduler.com).**

![Scheduler high level](https://bahr.dev/pictures/scheduler-high-level.png)

With the Point In Time Scheduler developers can specify when and how their endpoint should be invoked. They can do so for any time into the future, be it minutes, weeks, or years. Messages come back with latency of no more than 5 seconds. You only pay for what you use, and don't have to waste engineering time operating a scheduler.

For feedback, I invite you to use [ZipMessage](https://zipmessage.com/gwcyvrb1). You can send text, audio, video and share your screen.
You can also reach me via [Twitter](https://twitter.com/bahrdev).

## Getting Started

To start using the Point In Time Scheduler, head over to [the dashboard](https://app.point-in-time-scheduler.com) and sign in with your GitHub account.

There are four main steps:

1. Create an endpoint where the scheduler will send messages to
2. Register an application with that endpoint and get an API key
3. Schedule a message
4. Receive a message

If you don't have an endpoint for callbacks yet, [there's a demo for you](https://github.com/bahrmichael/scheduler-demo-rest).
It's built with the Serverless Framework and you can deploy within minutes, if you already have an AWS account.
The demo has functions for sending and receiving messages, so you only have to deploy it, configure it, and wait for metrics to arrive in CloudWatch.

### Register an Application

Once [signed in](https://app.point-in-time-scheduler.com) you can register an application.
An application gives the scheduler information about where to send messages, and how to authorize itself.
The authorization that you specify here is the one that your endpoint requires.

In the first step choose the integration type REST. Then click on next.

![choose the integration type REST](https://bahr.dev/pictures/scheduler-register-2.png)

Currently, the scheduler only supports REST endpoints. SQS, EventBridge and SNS will be become available in the next weeks. Let me know what I should add first!

In the next step enter the URL of the endpoint for receiving messages. If you have an API with an endpoint `https://example.com/callback`,
that would be the URL you put in when registering an application.

![enter the URL of the endpoint](https://bahr.dev/pictures/scheduler-register-3.png)

In the following step you specify the authentication as a header name and header value. You can use an `Authorization` header,
an `x-api-key` header, or anything else.

![specify the authentication](https://bahr.dev/pictures/scheduler-register-4.png)

Give your app a name and description, something like "Test App" is enough to later remember what this was for. The
description is optional.

![Give your app a name and description](https://bahr.dev/pictures/scheduler-register-5.png)

In the last step, you will see an Application ID and an API key. You will need those later to send messages to the scheduler. For this article lets assume we have the Application ID "123" and the API key "S3crEt!".

![Generate api key](https://bahr.dev/pictures/scheduler-register-6.png)

### Schedule a Message

The scheduler has a REST API with which you can schedule messages:

```
POST
https://api.point-in-time-scheduler.com/messages

Content-Type: application/json
Authorization: Basic MTIzOlMzY3JFdCE=

{
  "payload": "123",
  "sendAt": "2022-01-05T18:32:42.124Z"
}
```

To schedule a message, you need a string `payload` (max. 1 KB), a `sendAt` timestamp (ISO 8601), and an `Authorization` header.

The `payload` can for example be an ID that maps to a workload in your application. Please don't send sensitive data.

The timestamp `sendAt` must be an ISO formatted string (ISO 8601). If you use Node.JS, you can use `new Date().toISOString()`.

To build the `Authorization` header, we need three steps. First, take the app ID and the api key, and concatenate them with a colon `:`. For example `123` and `S3crEt!` become `123:S3crEt!`.
Then base64 encode the concatenated string. For example `123:S3crEt!` becomes `MTIzOlMzY3JFdCE=`.
Finally, set the encoded value as a Basic Authorization header. In our example this would look like the following:

```
headers: {
  Authorization: "Basic MTIzOlMzY3JFdCE="
}
```

Everything ready? Now combine that into a JSON request to the scheduler's API.

Below are some examples. Remember to use your own Application ID and API key.

**curl**

```
curl -X POST -H "Authorization: Basic MTIzOlMzY3JFdCE=" -d '{"payload": "test", "sendAt": "2022-01-05T18:32:42.124Z"}' https://api.point-in-time-scheduler.com/messages
```

**TypeScript with axios**

```typescript
import axios from 'axios';

await _axios.post(`https://api.point-in-time-scheduler.com/messages`, {
    payload: 'test',
    sendAt: new Date().toISOString(),
  }, {
    headers: {
      Authorization: 'Basic MTIzOlMzY3JFdCE='
    }
});
```

### Receive the Message

Open the logs of your endpoint and wait until the timestamp specified above passes.
Within a few seconds of the timestamp the message should arrive at your endpoint.
Received your message? Congratulations! It's that simple!

If the message doesn't arrive, please make sure your endpoint is publicly accessible, and that you specified the right authorization parameters.
Feel free to [reach out to me](https://zipmessage.com/gwcyvrb1) to debug issues as well.

In the dashboard of the scheduler you can also navigate to the application, and then view its messages.
It will tell you if a message is pending, successful, or has failed.

If a message is pending, you can abort it. If a message failed, you can redrive it by clicking the according button.

These operations will soon be available as APIs.

## Service Guarantees

The scheduler guarantees at-least-once delivery. Due to the nature of distributed systems, messages can arrive more than once in rare cases.
Please make sure that your service uses an idempotent mechanism if necessary.

If your API does not return a 200 OK status code, the scheduler will retry up to 2 times with an exponential delay before marking a message as failed.
Once a message has failed, you can redrive it.

You may abort messages up until the last moment. Please note that this is on a best effort basis, and messages may already be in flight
if you abort it within the last seconds (REST) or minutes (SQS).

The scheduler is currently located in AWS' us-east-1, and will soon be available in other AWS regions as well.
In case of an outage, you may not be able to schedule new messages for a while.
Already scheduled messages will keep the at-least-once delivery guarantee. They will arrive at your endpoint as things recover.

## Service Limits

There are soft and hard limits. If you want to raise a soft limit, [please send a request](https://zipmessage.com/gwcyvrb1).
If you think a hard limit is too limiting, [please share your thoughts](https://zipmessage.com/gwcyvrb1)!

### Soft Limits

Each account can register up to 10 applications. This is a soft limit and [can be raised](https://zipmessage.com/gwcyvrb1).

The scheduler API has a default throttling limit of 100 messages per second, and a quota 10,000 messages per day.
This is a soft limit and [can be raised](https://zipmessage.com/gwcyvrb1).

### Hard Limits

Each payload must be no more than 1 KB large. This is currently a hard limit, but may be changed based on [your feedback](https://zipmessage.com/gwcyvrb1).

Your API must finish the request within 2 seconds. The scheduler will not wait longer, and mark the message as failed.
Please consider adding your own queuing mechanism (e.g. with SQS) if you expect tasks to take longer.

## Pricing

The public preview's purpose is to collect feedback, and get more data on usage and cost.
You're not charged at the moment, but once the Point In Time Scheduler reaches general availability, there will be a usage based pricing.

## Shouldn't cloud providers offer this serverless primitive?

Yes, absolutely! I hope that I can eventually retire this service when there's a cheaper and better solution out there.
Until then, I'll try my best to help you with this offering.

## Share your feedback!

Please share your feedback on [Twitter](https://twitter.com/bahrdev) or use [ZipMessage to start a conversation](https://zipmessage.com/gwcyvrb1).

I'm looking forward to hearing from you!
