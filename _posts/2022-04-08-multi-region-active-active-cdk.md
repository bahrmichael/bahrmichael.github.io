---
layout: post
title: "How to go multi-region with AWS Serverless and CDK"
description: "Join me in deploying a serverless application to multiple regions, with just about 100 lines of CDK code!"
backgroundUrl: "https://images.unsplash.com/photo-1451187580459-43490279c0fa?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&q=80"
---

Building globally distributed applications with automatic failover sounds like a challenging task that requires deep networking knowledge and many weeks of engineering effort.
What if I told you that this is possible with just about 100 lines of CDK code and a surface level understanding of networking?
Join me on a journey where we turn a serverless application global!

At [Stedi](https://stedi.com) we build a cloud platform for business integrations. Customers rely on us for exchanging data with their trading partners.
Because [Electronic Data Interchange (EDI) is already difficult enough](https://www.stedi.com/blog/what-makes-edi-so-hard),
we don't want our customers to also deal with high latency and [regional outages that can
last many hours](https://aws.amazon.com/message/12721/).

How can your application continue working when an AWS region is down? You deploy your application to multiple regions and make it global.
This is also known as [multi-region active-active](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/).
An application that's multi-region active-active should basically do what it does in one region,
but in the closest region to a customer, and fail over if one region is degraded. For this article, let's call an application that is multi-region active-active a global application.

There are many tools for routing global traffic in the arsenal that AWS and others offer: Application Load Balancer (ALB), Elastic Load Balancing (ELB),
Route53, and software you can install and maintain yourself on EC2 instances.

We are building serverless applications, which I define as something that uses managed services as much as possible.
I don't want to patch the OS myself, if experts can do that for everyone at once.

Following that principle we are more likely to use services like Lambda, API Gateway and Route53.
Services that require more operational effort are less likely to be used, esp. software that we have to operate ourselves and is not part of our core business.

In this article we will learn how to turn this

![Customer calls fixed region](https://bahr.dev/pictures/active-active-high-level-before.png)

into this

![Customer calls closest healthy region](https://bahr.dev/pictures/active-active-high-level-after.png)

## 1. Requirements

- You have an AWS account.
- You have experience using CDK with TypeScript. If not, you'll have to translate the examples to your language, or Infrastructure as Code (IaC) tooling.
- You already have a DynamoDB Global Table, or can create a new one and migrate data. Unfortunately normal tables can't be changed to global tables.
- You have a domain and [created a Route53 hosted zone for it](https://bahr.dev/2020/09/01/multiple-frontends/).
- You are using AWS API Gateway. This example doesn't apply to AWS AppSync.
- Your API exposes a `/health` endpoint that responds with a `200` status code.

## 2. What does your standard Serverless application look like?

Before getting started, our architecture looked like this:

![Single region architecture](https://bahr.dev/pictures/active-active-architecture-before.png)

Customers and Route53 are already global, but the rest is not. Let's figure out how to change that, starting with DynamoDB.

## 3. How do we turn serverless data "global"?

Our data lives in a DynamoDB table in one region, e.g. `us-east-1`. If we only wanted to extend our application along the US,
we could get away with keeping the data there, and doing cross-region calls from other US regions.
The further you get away from that region, the more the latency will increase. Doing a request from Australia to the US is probably not feasible for most customers.

So should we write our own replication? No! There's [DynamoDB Global tables](https://aws.amazon.com/about-aws/whats-new/2017/11/aws-launches-amazon-dynamodb-global-tables/).
You create a table in one region, and specify replication regions.
DynamoDB takes care of the rest. The replication latency is usually around one second, and can be monitored through the CloudWatch metric `ReplicationLatency`.
AWS does not seem to offer a guarantee on how quickly items will be replicated.

Please note that because of the bi-directional replication, DynamoDB applies a "last write wins strategy". Keep this in
mind when writing your application, because it may lead to unexpected data inconsistencies. It is even more important to
carefully roll out changes to your data model through multiple deployments so that the current state does not break.

There's one catch though! You can't transform existing regular tables to global tables. So if you have data already,
you may have to create a new application version, and then migrate data over.

We will cover the code for Global Tables in section 7. Let's continue with picking the best region.

## 4. How does the customer find the closest healthy region?

There are two aspects to this question: **Latency and availability**.

To solve latency alone, we might offer multiple isolated instances of the same service in different regions.
A customer who uses a datacenter near Frankfurt, Germany could then choose `eu-central-1`. That's about as close as it gets,
and latency should be very low. Most AWS services follow this principle.

But what when AWS' `eu-central-1` has a major outage? [We've seen](https://aws.amazon.com/message/12721/) that AWS services can fail so much
that a region becomes unusable for us. Everything fails all the time; most of the time we are just fortunate enough to not notice it.
There probably won't be buildings going up in fire and years of data being lost, but an unfortunate bug in DynamoDB could prevent us from accessing tables for hours.

To the rescue come health checks, which have been around in the industry for a long time.
If you have more than one host, you need to distribute traffic. Load balancers can ask the hosts "are you healthy?"
If they respond with "I'm healthy" then everything is fine. If they don't, then the load balancer stops sending traffic there until the host recovers.
[Watch Bart Castle covering how load balancers work](https://www.youtube.com/watch?v=79ZGHUSlvFM).

I'm deliberately excluding larger events where many regions fail, or a malicious actor cuts undersea cables. There may be
solutions for this, but they are out of scope for this article.

Route53 is a fully managed service that can handle this networking concern. Over the next chapters we'll figure out how to use it.

## 5. The new architecture

Here is the new architecture that will make our application ready for global failover. The boxes in orange are globally
deployed, the box in blue is deployed in multiple regions.

![Global architecture](https://bahr.dev/pictures/active-active-architecture-after.png)

In the next chapters we take an existing single-region CDK application and turn it into a global application. We will cover data, deployment, and DNS routing.

## 6. Infrastructure as Code with CDK

Infrastructure as Code is key with serverless applications! There are many more moving parts than when you only deploy a JAR file to a VM.
Another benefit is that developers can usually deploy the production application to their developer account by running a single command.

There are many great frameworks out there like Serverless Framework, SAM, and the Cloud Development Kit (CDK).
Because we use the CDK at Stedi, I will stick with it for the scope of this article.

## 7. Global Data

In our serverless application we already use the CDK. Because of that we access tables via ARNs, but how do we access a global table?

A global table still has an ARN, which we can either construct ourselves per region, or use the CDK function `Table.formTableName` to resolve the ARN based on the table's name.
As long as you pass in the table name via an environment variable, your Lambda code doesn't have to change. We only need to update our infrastructure code.
Continue reading to learn more why we're not using an auto-generated table name.

The infrastructure differs in that we create the table in one region, and let DynamoDB replicate it to other regions.
This means that we must not create the table again in secondary regions.

Below you see a CDK example of how I create a table in the main region, and then reference it in secondary regions.

```typescript
function createTable({region, tableName, replicationRegions}: CreateTableProps) {
    if (region === MAIN_REGION) {
        return new Table(this, "Table", {
            tableName,
            billingMode: BillingMode.PAY_PER_REQUEST,
            partitionKey: {
                name: "pk",
                type: AttributeType.STRING,
            },
            replicationRegions,
        });
    } else {
        return Table.fromTableName(this, "Table", tableName);
    }
}
```

[Check out the full source code on GitHub](https://github.com/bahrmichael/multi-region-active-active-cdk).

Here are some tricky things I stumbled upon:

1. Using an **auto-generated table name** was more difficult than expected, because we cannot use CloudFormation's Fn::ImportValue with cross-region stacks. Instead, we hardcoded the table's name. If you need to isolate dev stacks, you can add a suffix to the table name.
2. Do you have data **migration scripts** that run automatically? Update them to only run for your main region. All the data is available there, and changes will be replicated to other tables.
3. You don't need to include all `replicationRegions` initially, but can expand that array over time. Due to Route53 HealthChecks this solution however is [limited to certain regions](https://docs.aws.amazon.com/Route53/latest/APIReference/API_HealthCheckConfig.html).

## 8. Deployment

As you've seen in the previous section, our application has to be region-aware to understand if it should create a new table, or use a replicated one.

So how do we tell our application which region it lives in? There are intrinsic functions which let CloudFormation resolve the region, but that doesn't always work.
Some CDK constructs can't deal with tokens that will be resolved later, and therefore we have to pass in the region through an environment variable.
Upon deployment, we use the `AWS_REGION` environment variable, and pass its value into our CDK stack.

```typescript
const app = new cdk.App();
new GlobalApplication(app, 'GlobalApplication', {
  env: {
    region: process.env.AWS_REGION ?? process.env.CDK_DEFAULT_REGION,
  }
});
```

You can see that we use the environment variable `AWS_REGION` if it's available, and fall back to whatever CDK provides us.
The target region is now available in the file `stack.ts` by accessing the variable `props.env.region`.

Update your remaining code to also use the `region` property where necessary.
In the [proof of concept](https://github.com/bahrmichael/multi-region-active-active-cdk), I store the region with every new item in DynamoDB
so that we can later see what region our request ended up in.

Now let's deploy our application to multiple regions by adding deployment steps for each region like below:

```bash
export AWS_REGION="us-east-1"
cdk bootstrap
cdk deploy
```

Expect this to take 5-10 minutes per region.

Why do we need `cdk boostrap`? For some steps, CDK needs to upload files to an S3 bucket.
If you want to read more about the reasons, [check out this document](https://github.com/aws/aws-cdk/blob/master/design/cdk-bootstrap.md).

In a CI environment you may run `cdk deploy --require-approval never` to automatically accept all questions.

Congratulations! Your application now lives in multiple regions. Now let's get to the most challenging part: DNS.

## 10. DNS Routing

Our application now lives in multiple regions, and when we look at the CloudFormation stacks, there are
API Gateways that have their own [execute-api domain](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-call-api.html).
But we neither want to expose that domain to our customers, nor require customers to pick the right location, as well as also handling outages on our side themselves.
So how do we let Route53 find the best region? **Latency based routing with health checks**!

For health checks to work we need both a domain, and a health endpoint on our service. The health endpoint can simply respond to a request at `/health` with `200 OK`. That's all that Route53 needs.

DNS can be tricky, and every time I think I understand it, I get proven wrong. There's even a Haiku about the challenges with DNS:

> It’s not DNS
>
> There’s no way it’s DNS
>
> It was DNS

Let's start carefully by looking at Route53's health checks in the AWS Management Console, and figure out the required pieces step by step.
Go to Route53, pick the hosted zone for your domain, and click on "Create Record".

In the first step of the wizard, we choose latency as the routing policy.

![AWS Console Routing Policy](https://bahr.dev/pictures/active-active-routing-policy.png)

In the second step of the wizard, we add a latency record, add the API Gateway for one of the regions we deployed to.
Then we pick the region again (not sure why, but let's roll with it), and select a health check.

![AWS Console Latency Based Routing](https://bahr.dev/pictures/active-active-latency-config.png)

You can skip the health check if you don't have one yet, but I highly recommend adding one.
Doing that through the console requires a lot of clicking, and that's why using the CDK is a great idea here.

Speaking of CDK, let's turn the above into code.

### 10.1. Add parameters for domain name and hosted zone

We started with an existing hosted zone for our domain. How do we tell CDK which hosted zone it should use?

First we **add two more parameters** to the file `bin/app.ts`:

```typescript
const app = new cdk.App();
new GlobalApplication(app, 'GlobalApplication', {
  env: {
    region: process.env.AWS_REGION ?? process.env.CDK_DEFAULT_REGION,
  },
  hostedZoneId: process.env.HOSTED_ZONE_ID!,
  domainName: process.env.DOMAIN_NAME!,
});
```

Then we extend the parameters of the constructor in `stack.ts` to accept the new parameters:

```typescript
export class GlobalApplication extends Stack {
    constructor(scope: Construct, id: string, props: StackProps & {hostedZoneId: string, domainName: string}) {
        super(scope, id, props);
        // ...
```

### 10.2. Each region's API Gateway gets their own certificate

To connect API Gateways with our domain in Route53, we need custom domain names.
For that we first create certificates, then connect them with API Gateway and finally add an API Gateway Domain Name.

Below you see the required code. We'll step through the individual pieces right away.

```typescript
const hostedZone = HostedZone.fromHostedZoneAttributes(this, "HostedZone", {
    hostedZoneId,
    zoneName: domainName
});

const certificate = new DnsValidatedCertificate(this, `${region}Certificate`, {
    domainName,
    hostedZone,
    region,
});

const apigwDomainName = new DomainName(this, `${region}DomainName`, {
    domainName,
    certificate,
    securityPolicy: SecurityPolicy.TLS_1_2,
});

apigwDomainName.addBasePathMapping(restApi);
```

First, we resolve the hosted zone from the ID and domain name we added in the previous step.
We need the hosted zone ID to issue a DnsValidatedCertificate. The nice thing about this construct is that the validation happens automatically (if the DNS config is right).

Because some resources' names have to be globally unique, and because we want to isolate our stacks from one another, we create dedicated certificates per region.

After issuing a certificate, we use it to create a DomainName which tells API Gateway to map requests for our domain to the API Gateway for that region.
Note that this only tells API Gateway how to handle requests that reach it, but we still need to add DNS records so that requests actually reach API Gateway.

With the last line and the function call `addBasePathMapping` we connect our REST API with the domain name.

### 10.3. Create a health check

Route53 has to know which APIs are healthy, and how fast they respond. To provide that information, we create a HealthCheck.

```typescript
const executeApiDomainName = Fn.join(".", [
    restApi.restApiId,
    "execute-api",
    region,
    Fn.ref("AWS::URLSuffix")
]);

const healthCheck = new CfnHealthCheck(this, `${region}HealthCheck`, {
    healthCheckConfig: {
        type: "HTTPS",
        fullyQualifiedDomainName: executeApiDomainName,
        port: 443,
        resourcePath: `/${restApi.deploymentStage.stageName}/health`,
    }
});
```

Starting from the first line of the snippet, we build a URL like `https://{restapi_id}.execute-api.{region}.amazonaws.com/{stage_name}/`.
This URL directly targets the API Gateway instance of the region we deploy this stack to.
Once we have the `executeApiDomainName`, we pass it to the `CfnHealthCheck` construct,
which is an [L1 CDK construct that directly represents a CloudFormation resource](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html#constructs_l1_using).\
Since 2019 there's an [open issue](https://github.com/aws/aws-cdk/issues/4391) to make this available as an L2 construct.
You can learn more about the health check on the [CloudFormation documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-route53-healthcheck.html).

The property `resourcePath` tells the health check which endpoint to use. If your health endpoint is different to `/health`, you can change it here.
[Here's my functionless health check from the proof of concept](https://github.com/bahrmichael/multi-region-active-active-cdk/blob/main/lib/api.ts#L136-L149).

By default, the health check runs every 30 seconds. You can add the parameter `requestInterval` to change that.

The health check can only check to the regions `us-east-1 | us-west-1 | us-west-2 | eu-west-1 | ap-southeast-1 | ap-southeast-2 | ap-northeast-1 | sa-east-1` ([Source](https://docs.aws.amazon.com/Route53/latest/APIReference/API_HealthCheckConfig.html)).

### 10.4. Add an ARecord to Route53

This is the last step, before our application is multi-region capable. We now add an ARecord that uses the health check to
help Route53 figure out what region it should route to:

```typescript
const dnsRecord = new ARecord(this, `${region}`, {
    zone: hostedZone,
    target: RecordTarget.fromAlias(new ApiGatewayDomain(apigwDomainName))
});

const recordSet = dnsRecord.node.defaultChild as CfnRecordSet;
recordSet.region = region;
recordSet.healthCheckId = healthCheck.attrHealthCheckId;
recordSet.setIdentifier = `${region}Api`;
```

The ARecord will be placed in our HostedZone, and target the API Gateway we deployed into that region.
There's no high level support in CDK for attaching health checks, so we use the syntax `dnsRecord.node.defaultChild` to get the configuration object.

Then we configure the region and pass a reference to the health check that shall be used for this ARecord.
You can read more about the record set on the [CloudFormation documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-route53-recordset.html).

### 10.5. Deploy and verify

Now deploy the application, and check the newly created record in Route53. If all went well, it should look similar to the picture below:

![Route53 Record for latency routing](https://bahr.dev/pictures/active-active-config.png)

That's it, your application is now global. Congratulations!

## 11. Test it

If you deployed the proof of concept, you can create new records with a `POST` request to `your-domain.com/data/`.

![Create a new record](https://bahr.dev/pictures/active-active-post-data.png)

This will return an ID, which you can use in another `GET` request to `your-domain.com/data/<ID>`.

![Get the new record](https://bahr.dev/pictures/active-active-get-data.png)

Here we see that Route53 picked the closest healthy region. For me currently sitting in Berlin (Germany), this is `eu-west-1` (Ireland).

## 12. Conclusion

Thank you for joining me on this journey! I hope that multi-region active-active is not a scary term anymore, but one
where you now have confidence and enough information to achieve it yourself.

We learned about the concepts of a multi-region active-active application, and developed the code to make
a serverless application global with CDK. Don't worry if DNS doesn't work on the first attempt, [there are many things that can go wrong](https://bahr.dev/2020/09/01/multiple-frontends/#troubleshooting).

If you need help, then check out the [proof of concept on GitHub](https://github.com/bahrmichael/multi-region-active-active-cdk), or reach out to me [via Twitter](https://twitter.com/bahrdev).

I'd like to give a thank you to my coworkers, and especially [Rafal Wilinski](https://twitter.com/rafalwilinski) with whom I worked on this recently.
It took us less than one week of engineering effort to turn one of our applications global.

Did you find something that can be optimized, or know guides for other services like AppSync? Please let me know!

## 13. Resources

- [Disaster Recovery (DR) Architecture on AWS, Part IV: Multi-site Active/Active](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/)
- [Convert Your Single-Region Amazon DynamoDB Tables to Global Tables](https://aws.amazon.com/blogs/aws/new-convert-your-single-region-amazon-dynamodb-tables-to-global-tables/)
- [CloudFormation for Route53](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_Route53.html)
- [Summary of the AWS Service Event in the Northern Virginia (US-EAST-1) Region](https://aws.amazon.com/message/12721/)
- [AWS CDK Tutorial for Beginners](https://bobbyhadz.com/blog/aws-cdk-tutorial-typescript)
- [AWS CDK: Use Lambda with Application Load Balancer](https://sbstjn.com/blog/aws-cdk-lambda-loadbalancer-vpc-certificate/)
- [CDK and Route 53 Failover](https://repost.aws/questions/QUfRU5dvaoQRC97y4F9A9UwA/cdk-and-route-53-failover)
