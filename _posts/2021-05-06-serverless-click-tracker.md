---
layout: post
title: "How To Build Your Own Serverless Click Tracker"
backgroundUrl: "https://images.unsplash.com/photo-1588600878108-578307a3cc9d?auto=format&fit=crop&q=80"
description: "This article shows you how to track clicks on your website with serverless technology."
---

What if I told you that you don't need any analytics tracker, cookies (or biscuits if you're British) or JS libraries at all to understand how visitors of your website move around? And you can do this serverless, so you only pay for what you use.

With the `ping` attribute of the HTML element `<a>` you can **enable hyperlink auditing** for any link on your website. This means that you can connect any link on your website with a self-made analytics service that you fully control. This does not apply to links from other websites to yours though.

```
<a href="/about" ping="/ping">About Me</a>
```

If you don't want to build and maintain a tracker yourself while maintaining your viewers' privacy, you should check
out Fathom Analytics by following this [referral link](https://usefathom.com/ref/I1EFJ1). I'm a happy customer
and [the statistics of my blog are public](https://app.usefathom.com/share/gvlddgwj/bahr.dev).

## Prerequisites

To get the most out of this article, you should be familiar with using AWS Lambda.

The code snippets are in TypeScript, and use `aws-sdk` V2.

## What's the ping attribute?

[The ping attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a#ping) takes a list of URLs. When
you click on the link, your browser will send a `PING` body, along with some insightful headers. Within your Lambda function you can see much more than the information below,
but this information already lets us create a hash that's individual per IP.

```json
{
  "requestContext": {
    "accountId": "YOUR_AWS_ACCOUNT_ID",
    "identity": {
      "sourceIp": "YOUR_IP_ADDRESS",
      "userAgent": "YOUR_USER_AGENT"
    }
  }
}
```

It can help you understand how viewers move within your website, or where they leave to. On my blog however about 80% of the viewers only view one page and then leave again. So it might not be too useful with high [bounce rates](https://support.google.com/analytics/answer/1009409?hl=en).

The ping attribute is [supported by all major browsers](https://caniuse.com/ping). Firefox however disables it by default. On my blog 17% of the viewers use Firefox.

I like the `ping` attribute because it's more transparent than JS/iframe based approaches, it can be controlled by the
user, and does not require any cookies. If one of the browsers doesn't allow you to disable hyperlink auditing (grr
Chrome), you can use extensions like [uBlock Origin](https://github.com/gorhill/uBlock) to prevent this traffic. This is
a lot harder to prevent with JS libraries.

## Build it

In this section I will show you how to set up a simple analytics service, where we record and output a user journey. I'm
going to show you a serverless solution, where we have a couple Lambda functions which serve HTML pages, and one that
our browser will send the ping requests to. I'm going to user the [serverless framework](https://www.serverless.com/) with TypeScript.

[The whole source code is available on GitHub](https://github.com/bahrmichael/hyperlink-auditing).

### Architecture

Our application consists of three APIs for hyperlink auditing, showing the user journey, and serving a website with a
couple links. It is backed by a DynamoDB table and Lambda functions.

![Architecture](https://bahr.dev/pictures/auditing-architecture.png)

### Serve a Website

This function returns an HTML snippet that the browser renders as a website. It's not stylish, but it gets the job done. You can replace this with websites hosted on any other service, like AWS Amplify. The only important thing is that the `/ping` address lives on the same domain so that you can avoid CORS problems.

```typescript
import {htmlResponse} from '@libs/apiGateway';

export const main = async (event) => {
  return htmlResponse(`
        <h1>Welcome To hyperlink auditingng</h1>
        <p>Current Page: <pre>${event.path}</pre></p>
        <p>Follow one of the links below to generate data.</p>

        <p><a href="./" ping="./ping">Home</a></p>
        <p><a href="./about-me" ping="./ping">About Me</a></p>
        <p><a href="https://bahr.dev" ping="./ping">Blog</a></p>
        <p><a href="./no-audit">Unaudited Page</a></p>

        <hr/>
        <p><a href="./analytics">See the results</a></p>
    `);
}
```

Check out the [source code on GitHub](https://github.com/bahrmichael/hyperlink-auditing) for the function `htmlResponse`.

### Audit the Hyperlink

This function accepts a `POST` request, ignores the body, and uses the headers to identify from where to where a user
went. It also uses the headers to generate a hash of the user's session. This is not to identify a user, but to match
multiple requests of a journey together. We also include the date into this hash so that we're not able to see a user's
movement over more than one day.

```typescript
import {DynamoDB} from 'aws-sdk';

const {TABLE_NAME} = process.env;
const ddb = new DynamoDB.DocumentClient();

export const main = async (event) => {

  const item = {
    id: 'some-user-hash',
    date: new Date().toISOString(),
    from: event.headers["ping-from"],
    to: event.headers["ping-to"],
  };

  await ddb.put({
    TableName: TABLE_NAME,
    Item: item
  }).promise();

  return {
    statusCode: 200,
    body: ''
  }
}
```

The function above accepts any request, creates a hashed ID for the user, and stores the `from` and `to` URLs in a
DynamoDB table. In the next function we will query this data to show the user their history.

### View the Results

This function return your recent journey, based on the same hashing mechanism as we used for auditing hyperlinks.

```typescript
import {htmlResponse} from '@libs/apiGateway';
import {DynamoDB} from 'aws-sdk';

const {TABLE_NAME} = process.env;
const ddb = new DynamoDB.DocumentClient();

export const main = async (event) => {

  const items = (await ddb.query({
    TableName: TABLE_NAME,
    KeyConditionExpression: 'id = :id',
    ExpressionAttributeValues: {
      ':id': 'some-user-hash'
    },
    ScanIndexForward: false
  }).promise()).Items;

  const rows = items.map((i) => {
    return `
        <tr>
            <td>${i.date}</td>
            <td>${i.from}</td>
            <td>${i.to}</td>
        </tr>
        `;
  });

  return htmlResponse(`
        <h1>Welcome To Analytics</h1>
        <p><a href="./">Home</a></p>
        <table>
            <tr>
                <th>Date</th>
                <th>From</th>
                <th>To</th>
            </tr>
            ${rows.join('')}
        </table>
    `);
}
```

## Test It

You can test this yourself by deploying the application with
the [source code on GitHub](https://github.com/bahrmichael/hyperlink-auditing).

Open the website endpoint, and click around a bit. Once you've clicked a couple links, follow the link to the analytics.

![Analytics](https://bahr.dev/pictures/analytics-result.png)

Here you should see which links you followed, if hyperlink auditing is enabled on your browser, and not blocked by any
extension.

## Conclusion

hyperlink auditing is a lightweight and transparent approach to understanding your viewers' journeys on your website.
With just one function, you can start recording movements without giving your viewers' data to anyone who will monetise
them.

If you only want some simple analytics, while maintaining your viewers' privacy, you should check out Fathom Analytics
by following this [referral link](https://usefathom.com/ref/I1EFJ1).
