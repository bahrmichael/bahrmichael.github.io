---
layout: post
title: "How To Paginate DynamoDB Tables With The AWS SDK For TypeScript on NodeJS"
backgroundUrl: "https://images.unsplash.com/photo-1564669722947-c89159202d19?auto=format&fit=crop&q=80"
description: "This article has DynamoDB pagination examples in TypeScript on NodeJS that you can copy and paste."
---

[Python library boto3 has built in paginators](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/dynamodb.html#paginators),
but this doesn't seem to be the case for AWS SDK for JavaScript. This article has code examples that you can copy and paste
to your code.

The table setup and examples are [available on GitHub](https://github.com/bahrmichael/ddb-paginators).

## Prerequisites

This guide is for **TypeScript** and `aws-sdk` (article written with version `2.958.0`).

The examples are intended for the `DynamoDB.DocumentClient`. If you don't have to use this client, consider Jeremy Daly's [DynamoDB Toolbox](https://github.com/jeremydaly/dynamodb-toolbox).

I assume that you are familiar with DynamoDB table structures, especially partition and sort keys.

## Async Generators

You can use
[async generators](https://javascript.info/async-iterators-generators#async-generators-finally)
as a wrapper for any AWS SDK request where
[continuation tokens](https://phauer.com/2017/web-api-pagination-continuation-token/#:~:text=A%20%E2%80%9CContinuation%20Token%E2%80%9D%20solves%20this,to%20he%20has%20to%20continue.)
are used. The example below is based on [Tamás Sallai's article](https://advancedweb.hu/how-to-paginate-the-aws-js-sdk-using-async-generators/).

```typescript
export const getPaginatedResults = async(fn) => {
  const EMPTY = Symbol("empty");
  const res = [];
  for await (const lf of (async function* () {
    let NextMarker = EMPTY;
    let count = 0;
    while (NextMarker || NextMarker === EMPTY) {
      const {marker, results, count: ct} =
        await fn(NextMarker !== EMPTY ? NextMarker : undefined, count);

      yield* results;

      // if there's no marker, then we reached the end
      if (!marker) {
        break;
      }

      NextMarker = marker;
      count = ct;
    }
  })()) {
    res.push(lf);
  }

  return res;
};
```

## Usage Example

Copy the function above to your source code, and use it as shown below.

```typescript
import { DynamoDB } from 'aws-sdk';
const ddb = new DynamoDB.DocumentClient();

/*
The table in this example has a partition key 'pk'
and a sort key 'sk'. While not shown in the query
below, the sort key is required to store multiple
items in a partition.
 */
const ddbQueryParams = {
  TableName: TABLE_NAME,
  KeyConditionExpression: 'pk = :pk',
  FilterExpression: 'randomValue > :r',
  ExpressionAttributeValues: {
    ':pk': 'my-partition',
    ':r': 0.5,
  },
};

const records = await getPaginatedResults(async (ExclusiveStartKey) => {
  const queryResponse = await ddb
    .query({ExclusiveStartKey, ...ddbQueryParams })
    .promise();

  return {
    marker: queryResponse.LastEvaluatedKey,
    results: queryResponse.Items,
  };
});

console.log(records);
// [record1, record2, record3, ...]
```

The result is one array with all the records from the paginated requests.

This is the cleanest solution I've seen so far, and doesn't require you to perform any logic with the
result or continuation token, if all that you care about is the full result of the query.

## Adding Offset And Pagination

If your query targets a partition that has a sort key, we can add offset and pagination to the example. Note that this
approach may load more items from DynamoDB that `pageSize` specifies, but the function will return at most `pageSize` records.
This is because DynamoDB operations can read a maximum of 1 MB per request, and adding a `Limit` parameter would lead
to many more requests than necessary.

If our sort key is numerical, or lexicographically ascending sorted, we achieve an offset by specifying the first sort key that
the query shall look at.

To limit the page size, we add an early break. The example below can return a page size that's larger than the expected
page. It requires the calling code to discard additional records, and pick the appropriate start key for the next request.

```typescript
import { DynamoDB } from 'aws-sdk';
const ddb = new DynamoDB.DocumentClient();

const offset = 10;
const pageSize = 50;

const ddbQueryParams = {
  TableName: TABLE_NAME,
  KeyConditionExpression: 'pk = :pk and sk > :sk',
  ExpressionAttributeValues: {
    ':pk': 'my-partition',
    ':sk': offset,
  },
};

const records = await getPaginatedResults(async (ExclusiveStartKey, count: number) => {
  const queryResponse = await ddb
    .query({ExclusiveStartKey, ...ddbQueryParams })
    .promise();

  // stop the pagination when we reach the pageSize
  if (count + queryResponse.Count >= pageSize) {
    return {
        // manually trim to the item count to max pageSize records
      results: queryResponse.Items.slice(0, pageSize - count),
      // return an empty marker to indicate that we have
      // no further results to process
      marker: null,
    };
  }

  return {
    marker: queryResponse.LastEvaluatedKey,
    results: queryResponse.Items,
    count: count + queryResponse.Count,
  };
});

console.log(records);
// [record1, record2, record3, ...]
```

I hope that this helps you to add pagination quicker when you need it.

Found a better way to use this code? Please share it or [create a pull](https://github.com/bahrmichael/ddb-paginators) request so that I can improve this article :)

## Resources

* [Repository with Examples and Table Setup](https://github.com/bahrmichael/ddb-paginators)
* [Async Generators](https://javascript.info/async-iterators-generators#async-generators-finally)
* [Continuation Tokens](https://phauer.com/2017/web-api-pagination-continuation-token/#:~:text=A%20%E2%80%9CContinuation%20Token%E2%80%9D%20solves%20this,to%20he%20has%20to%20continue.)
* [How to paginate the AWS JS SDK using async generators](https://advancedweb.hu/how-to-paginate-the-aws-js-sdk-using-async-generators/)
