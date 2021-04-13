---
layout: post
title: "How To Remove DynamoDB Local And Test With AWS Managed DynamoDB"
backgroundUrl: "https://images.unsplash.com/photo-1566878466493-939ba590835f?auto=format&fit=crop&q=80"
description: "This article shows you how we switched from testing with DynamoDB Local to testing with managed DynamoDB on AWS."
---

Should we test directly against databases in the cloud like DynamoDB, or run local emulations? That's still a debated topic,
with [some](https://github.com/localstack/localstack) [for](https://medium.com/swlh/how-to-mock-aws-services-in-local-development-e231852e8a0f)
[testing](https://betterprogramming.pub/dont-be-intimidated-learn-how-to-run-aws-on-your-local-machine-with-localstack-2f3448462254)
[locally](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html), and
[some](https://medium.com/@chrisandrews7/real-world-integration-testing-in-serverless-c9b4b100309a)
[for](https://serverlessfirst.com/integration-e-2-e-tests/)
[testing](https://opensource.com/article/17/8/testing-production)
[with](https://copyconstruct.medium.com/testing-in-production-the-safe-way-18ca102d0ef1) the cloud.

While frequently mentioned in the community,
[the service differences](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html) were much
less a problem than the onboarding for new team members, as well as issues when
the local database's process stopped responding.

In this article, I'll show you how my team switched from testing against a local emulation to testing with managed DynamoDB on AWS.

## What Is DynamoDB Local?

[DynamoDB Local](https://dev.to/ara225/how-to-run-aws-dynamodb-locally-156i) is a DynamoDB
compatible API, backed by an SQLite database. You can run it locally on your own developer machine. This allows you to run
tests without the need for creating tables in your AWS account, and without requiring any network access or permissions
while running these tests.

My team used DynamoDB Local to verify our data model in unit tests. In addition to the reasons mentioned above, this also
allowed us to test out table changes quickly without introducing problems for others who might be using the same
database.

Some of us probably still remember the pain of traditional databases, that either had to be installed
locally with a couple of hacks trying to get the right system permissions, or using a shared instance that frequently
lead to conflicting changes. Using a local version of DynamoDB can be very compelling.

## Prerequisites

To get the most of this article, you should be familiar with AWS DynamoDB.

The examples in this article are based on TypeScript and the [AWS SDK for JavaScript](https://github.com/aws/aws-sdk-js).
We also use the test framework [Jest](https://jestjs.io/). It helps if you're familiar with those, but I tried to keep
the examples simple enough, so that they can be adapted to other languages.

## How Do We Structure Tests?

For simplicity, we're only looking at unit and integration tests here. The unit tests help us verify
that our classes and functions do what we expect from them. Sometimes we include multiple classes, but try to
stick with rather simple situations that include few moving parts. At the integration level, we deploy our application
to personal and staging accounts, and use well-defined entrypoints for testing. One of these entrypoints is API Gateway.

As part of the unit level, we also had tests that validated our data model against DynamoDB. Let's call them functional
tests. These functional tests ensured that the data model classes are compatible with DynamoDB, e.g. for key schemas or
indices. We used DynamoDB Local for the functional tests, which could be executed in various environments:

- The developer's notebook
- The developer's AWS account for a personal CICD pipeline
- The team's AWS account for running pull request builds
- The team's AWS account for the CICD pipeline

I listed the environment above, because we need to know where multiple test runs could use the same table. With
DynamoDB Local this wasn't an issue, as it created a new ephemeral table before each test run.

## What Had To Change For Migrating To Managed DynamoDB?

1. The developer must have an active AWS profile with valid credentials and the right permissions when running the tests.
2. The test executors in the AWS accounts must also have the right permissions. This applies to CICD and Pull Request builds.
3. Our tests must create tables on the fly.
4. The tests need to use random values, so that test executions don't conflict with each other.
5. When more than one developer runs tests in one account (e.g. for pull request builds), then the tests must use
   different tables. Otherwise, conflicting table changes would lead to confusing test failures.
6. Tests must use retry mechanisms to handle DynamoDB's eventual consistency.

## 1. Developer Permissions

In our team, every developer has their own AWS account. This allows everyone to test in an isolated environment, before
pushing changes to the shared account where they start to affect others. To let the functional tests use
the [AWS JavaScript SDK](https://aws.amazon.com/sdk-for-javascript/) for table setup and database queries, we need to have an active AWS profile and valid
credentials. [Check this guide on how to create named AWS profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html).

In our `package.json` we have a test script as shown below, which passes the `personal` profile to our test runner.

```
"test:functional": "AWS_PROFILE=personal jest --config test_config/jest.functional.config.js"
```

The jest config file also references further environment variables, like the table names for running tests.
Making the table names configurable is important, so that we can select unique table names when running
multiple test executors in the same account.

## 2. Text Executor Permissions

We need to grant similar permissions to the test executors which run in the CICD pipelines and on pull request builds.

In these environments there is already an active AWS profile with valid credentials, but the permissions might not be
sufficient yet. We're going to fix that, by attaching a new IAM policy to the test executor's role, as shown below.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "dynamodb:DescribeTable",
        "dynamodb:CreateTable",
        "dynamodb:DeleteTable",
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:dynamodb:MY_ACCOUNT_ID:MY_REGION:table/MyTestTable*",
        "arn:aws:dynamodb:MY_ACCOUNT_ID:MY_REGION:table/MyTestTable*/index/*"
      ]
    }
  ]
}
```

With the above IAM policy we allow the test executor to create and delete the table `MyTestTable*`. I'm using a wildcard suffix here to
allow for more tables to be added when we need to isolate concurrent test executions. The test executor is also
able to perform basic item operations, and query any index of that table.

[Check out the full list of DynamoDB permissions](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazondynamodb.html).
Please let me know if I missed any permission!

## 3. Replace DynamoDB Local With Managed DynamoDB Table Setup

Removing DynamoDB Local is pretty easy: Remove the dependencies, and any related setup that runs before executing the
tests.

Setting up a DynamoDB table requires a table configuration, verifying if the table exists, and creating the table
if necessary.

### 3.1 Write The Table Setup

The fastest way to create a new table is the `createTable` API of DynamoDB. CloudFormation or CDK may be a cleaner
solution, but take longer. By using the `createTable` API we create two tables within 20 seconds.

As you can see below, we have one typed config per table:

```typescript
import {DynamoDB} from 'aws-sdk';

export const MyTableConfig: DynamoDB.CreatTableInput = {
  TableName: process.env.TABLE_NAME,
  BillingMode: 'PAY_PER_REQUEST',
  KeySchema: {...},
  AttributeDefinitions: [...]
};
```

It's important to use an environment variable for the table name, so that you can isolate test runs which might modify
the table. We will see a solution for conflicting table changes later. Keep in mind that there's a [service
limit of 256 DynamoDB tables per account](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html#limits-tables).

### 3.2 Check If The Table Exists

Based on the table name in the config above, we can check if that table exists, and skip the setup if possible. This
allows our tests to run with nearly no delay.

```typescript
import {DynamoDB} from 'aws-sdk';
const dynamodb = new DynamoDB();

async function doesTableExist(tableName: string): Promise<boolean> {
  try {
    await dynamodb
      .describeTable({
        TableName: tableName,
      }).promise();
    return true;
  } catch (e) {
    // instead of a try-catch block you
    // can also use the ListTables API
    console.log(e);
    return false;
  }
}

const tableExists = await doesTableExist(process.env.TABLE_NAME);
if (!tableExists) {
    // create table
}
```

### 3.3 Create The Table

If we need to create the table, then we run the `createTable` command with the table config from earlier, and use
the `waitFor` command, to wait until the table becomes usable. The `createTable` call will complete before the table is
ready, so we need to wait a bit before our tests start running.

```typescript
import {DynamoDB} from 'aws-sdk';
const dynamodb = new DynamoDB();

async function createTable(tableConfig: DynamoDB.Types.CreateTableInput): Promise<void> {
    await dynamodb
      .createTable(tableConfig)
      .promise();

    await dynamodb
      .waitFor("tableExists", {
          TableName: tableConfig.TableName
      }).promise();
}
```

## 4. Tests Must Use Random Values

If we don't want to recreate the table before each test run, then our tests must use random values so that they
don't conflict with each other. With DynamoDB Local this was not problem, because it quickly recreated the tables
between test runs. On AWS however, recreating tables takes about 20 seconds or longer.

In our example, we have multiple tenants, each represented by a UUID. Before every test run, we generate a new UUID so that
our tests are able to run in parallel, and consecutive test runs don't interfere with each other.

Here's an example on how to do that with Jest and the npm package `uuid`:

```typescript
import {v4 as uuid} from 'uuid';

let tenantId;

beforeEach(() => {
  tenantId = uuid();
});

test('should do something', () => {
  doSomething(tenantId);
});
```

## 5. Enable Isolated Concurrent Tests Within The Same Account

We use a CodeCommit repository, and run builds on pull requests. This means that if two or more engineers create or
update a pull request at the same time, the tests could conflict with each other, especially when one of
the pull requests modifies the table in a non-backwards compatible way.

To solve this problem, we create tables with a commit hash suffix before the tests run, and tear them down
afterwards. This approach adds a bit of runtime to the pull request build, but I think it's okay for pull request
builds to take a minute or two longer.

Because we used an environment variable for the table name, we can quickly switch what table the tests use. Please note that
your application code must also use the environment variable for the table name.

```shell
TABLE_NAME=my_table_name_commit_${COMMIT_HASH} yarn test
```

## 6. Add Retries Or Strong Consistency

By default, [DynamoDB uses eventually consistent reads](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html).

> When you read data from a DynamoDB table, the response might not reflect the results of a recently completed write operation. The response might include some stale data. If you repeat your read request after a short time, the response should return the latest data.

DynamoDB Local however is so fast that it looks like it had strong read consistency. This means that your tests which succeeded with DynamoDB Local
might sometimes fail when running against DynamoDB on AWS.

There are two approaches to fix this: Using strongly consistent reads, or letting tests retry the get calls.

### 6.1 Strongly Consistent Reads (not recommended)

By using strongly consistent reads, DynamoDB will wait until the data has been written to all nodes.

```typescript
import {DynamoDB} from 'aws-sdk';
const dynamodb = new DynamoDB();

await dynamodb.getItem({
  TableName: process.env.TABLE_NAME,
  Key: {myKey: 'something'},
  ConsistentRead: true,
}).promise();
```

This approach however [has some disadvantages](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html).

> - A strongly consistent read might not be available if there is a network delay or outage. In this case, DynamoDB may return a server error (HTTP 500).
> - Strongly consistent reads may have higher latency than eventually consistent reads.
> - Strongly consistent reads are not supported on global secondary indexes.
> - Strongly consistent reads use more throughput capacity than eventually consistent reads. For details, see Read/Write Capacity Mode

[Strongly consistent reads are twice as expensive as eventually consistent reads.](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html#HowItWorks.requests)

I suggest using retries instead strong read consistency.

### 6.2 Retries For Tests (recommended)

Most large scale systems are eventually consistent. Therefore, it's okay to reflect that in our tests. We use the
library `async-retry` to exponentially retry a request, as shown below.

```typescript
import {v4 as uuid} from 'uuid';
import * as retry from 'async-retry';

test('should retry', async () => {
  const id = uuid();
  await writeData(id);

  await retry(
    async () => {
      // a request that should return the id
      const result = await getData(id);
      expect(result).toBe(id);
    },
    {
      retries: 3
    }
  )
});
```

## Conclusion

Our tests run faster, and so far we haven't had any problems yet that this article doesn't provide a solution for.
You might not be able to run the functional tests if your network connection drops, but unit tests are still available.

One unresolved problem we will encounter in the future is about configuration drift. We build our application's
infrastructure with CDK, but the test setup uses an SDK compatible config. Initial tests have shown that the error messages
are sufficient to point future engineers to the duplicate config, but it would be nice if we could synthesize the test
setup from our CDK code.

As our tables use the `PAY_PER_REQUEST` model and have a timeout of 20 seconds, we don't expect any
significant cost, even if tests go rogue.

I suggest that you try out using managed DynamoDB for any tests that interact with DynamoDB, instead of relying on a local
emulation. Do you have any feedback or concerns? Please [reach out to me on Twitter](https://twitter.com/bahrdev)!
I'd love to hear what other issues you discover.

## Resources

- [Integration and E2E tests are the primary confidence drivers for serverless apps](https://serverlessfirst.com/integration-e-2-e-tests/)
- [How to Run AWS DynamoDB Locally](https://dev.to/ara225/how-to-run-aws-dynamodb-locally-156i)
- [Setting Up DynamoDB Local](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBLocal.html)
- [Named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html)
- [DynamoDB Table limits](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Limits.html#limits-tables)
- [DynamoDB Strong Read Consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [AWS SDK for Javascript](https://aws.amazon.com/sdk-for-javascript/)
- [Jest](https://jestjs.io/)
