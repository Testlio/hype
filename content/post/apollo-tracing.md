+++
author = "henri"
comments = false
date = "2018-02-12T09:00:00+02:00"
title = "Monitoring GraphQL services with Apollo tracing and Sumo Logic"
image = "images/post/graphql-tracing/cover.jpg"

share = true
slug = "monitoring-graphql-with-apollo-tracing"
tags = ["graphql", "sumologic", "monitoring"]
+++

Continuing on from our series on [serverless monitoring]({{< ref "serverless-monitoring.md" >}}) and [GraphQL testing]({{< ref "graphql-testing.md" >}}), this time we'll investigate how we can not only keep an eye on our GraphqQL service, but also use its logs as a source for analytics data.

<!--more-->

_Preface: This post assumes that we are using [Apollo](https://www.apollographql.com) as our GraphQL server and [Sumo Logic](https://www.sumologic.com) to manage our logs. Based on a quick web search, similar solutions can also be built on top of Elasticsearch/Logstash (by using [Kibana](https://www.elastic.co/products/kibana)) and other similar log management tools. The only real requirement is the ability to effectively filter logs and build dashboards on top of the results._

There are several well-written posts out there covering the benefits of using GraphQL (such as [this post](https://philsturgeon.uk/api/2017/01/24/graphql-vs-rest-overview/)), so we won't spend any time on that.

> ...gives clients the power to ask for exactly what they need and nothing more, makes it easier to evolve APIs over time... - [GraphQL website](http://graphql.org)

If you are anything like we are, then this sentence makes you think - how do we keep track of what parts of our API are actually being used? How do we know how said parts are performing? Can we keep track of not only what is being used, but also by whom? These are the questions we'll investigate in this post.

### Apollo Tracing

Few months ago, when we started to look deeper into GraphQL, we immediately stumbled upon a problem - how can we generate access logs in a generalised way without too much boilerplate in our server-side code?

After some investigation, Marko from our team submitted a detailed issue to one of [Apollo's GitHub repositories](https://github.com/apollographql/graphql-tools/issues/358). From there our investigation took us to [Apollo Tracing](https://github.com/apollographql/apollo-tracing), which seemed to be perfect for our needs.

So what is Apollo Tracing? In short, it is an extension built on top of the [GraphQL specification](http://facebook.github.io/graphql/October2016/#sec-Response-Format). Namely, the GraphQL specification allows additional metadata to be returned as part of the response. Metadata that could for example contain performance characteristics captured during the query resolution. This is exactly what Apollo Tracing does. It not only captures the start and end times of when the query was resolved, but also provides exact execution metrics for each individual resolver used.

For example, given the following query

```graphql
query {
  hero {
    name
    friends {
      name
    }
  }
}
```

The response from the service might look like this

```json
{
  "data": {
    "hero": {
      "name": "R2-D2",
      "friends": [
        {
          "name": "Luke Skywalker"
        }
      ]
    }
  },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2017-07-28T14:20:32.106Z",
      "endTime": "2017-07-28T14:20:32.109Z",
      "duration": 2694443,
      "parsing": {
        "startOffset": 34953,
        "duration": 351736,
      },
      "validation": {
        "startOffset": 412349,
        "duration": 670107,
      },
      "execution": {
        "resolvers": [
          {
            "path": [
              "hero"
            ],
            "parentType": "Query",
            "fieldName": "hero",
            "returnType": "Character",
            "startOffset": 1172456,
            "duration": 215657
          },
          {
            "path": [
              "hero",
              "name"
            ],
            "parentType": "Droid",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 1903307,
            "duration": 73098
          },
          {
            "path": [
              "hero",
              "friends"
            ],
            "parentType": "Droid",
            "fieldName": "friends",
            "returnType": "[Character]",
            "startOffset": 1992644,
            "duration": 522178
          },
          {
            "path": [
              "hero",
              "friends",
              0,
              "name"
            ],
            "parentType": "Human",
            "fieldName": "name",
            "returnType": "String!",
            "startOffset": 2445097,
            "duration": 18902
          }
        ]
      }
    }
  }
}
```

All the timing info is given in nanoseconds and as an offset from the original start time of the request. Nevertheless, with this information we can answer a lot of questions:

* How long did it take for the complete query to be resolved?
* Which resolvers were used and to what extent?
* How long resolving each of the properties took?

Enabling tracing is simple, when using Apollo Server, it is literally a single line. Once enabled, all of our requests start returning the tracing data in the response. For our needs, we did have to put in a bit more effort, as we didn't want to actually send the tracing data to the client, instead, we wanted to log the data out on the server-side.

```js
const { graphqlKoa } = require('apollo-server-koa');
const { TraceCollector, instrumentSchemaForTracing, formatTraceData } = require('apollo-tracing');

const graphqlHandler = graphqlKoa(request => ({
    schema: instrumentSchemaForTracing(schema),
    context: request
}));

const graphqlHandlerWithLogging = async(ctx, next) => {
    const traceCollector = new TraceCollector();

    traceCollector.requestDidStart();
    ctx._traceCollector = traceCollector;
    await graphqlHandler(ctx, next);
    traceCollector.requestDidEnd();

    const logData = {
        user: ctx.user,
        headers: ctx.request.headers,
        request: ctx.request.body,
        traceData: formatTraceData(traceCollector)
    };
    console.log(JSON.stringify(logData));

    await next();
};
```

Our route handled in Koa ended up looking something like the snippet above. We are using the [Koa flavoured Apollo Server](https://github.com/apollographql/apollo-server) and the generic [apollo-tracing](https://github.com/apollographql/apollo-tracing-js) package. Notice that we can simply log out the tracing data instead of embedding it in the query result - this keeps our responses small, whilst giving us a full access to the tracing data in the logs.

### Analysing the logs with Sumo Logic

Next step in our monitoring/analytics pipeline is to gather information across all the logs our service produces. We use [Sumo Logic](https://www.sumologic.com) as our log management tool, but this should be doable on other platforms as well.

Sumo Logic allows parsing and analysing logs with a relatively straightforward query language. For example, we can easily track the time it takes to resolve different properties in our GraphQL schema with the following query:

```json
json auto
| json "message.traceData.execution.resolvers" as resolvers
| parse regex field=resolvers "(?<resolver>\{[^\}]*\})" multi
| json field=resolver "fieldName", "parentType", "duration" as fieldName, parentType, duration nodrop
| concat(parentType, ".", fieldName) as resolverName
| fields - resolver, fieldName, parentType
| duration / 1000000 as duration
| pct(duration, 90) as duration_pct by resolverName
| sort by duration_pct asc
```

The query parses out the various resolvers and differentiates them by the type they refer to (important if you have the same property names on multiple types in your schema). It then does a nice 90% percentile based ordering of the resolvers, producing a nice looking bar graph of our service workings:

{{< figure src="/images/post/graphql-tracing/resolvers_by_time.png" caption="Graph of property resolvers, ordered by their execution time" >}}

We can then combine several of these graphs (adding graphs that focus on named queries, as those can be timed as well, or variables provided to the query that are also logged out), we can create efficient dashboards that give us an overview of our service.

{{< figure src="/images/post/graphql-tracing/dashboard_example.png" caption="An example of the kind of dashboards we can create for our GraphQL services" >}}

One of the things tracing is missing are errors (also discussed in the [issue mentioned above](https://github.com/apollographql/graphql-tools/issues/358)). We ended up also implementing a simple error middleware for our server that simply logs out the resolver name/title and the error that happened. Combining the two approaches allows us to also track the most problematic resolvers (a combination of slow and most erroneous resolvers).

### Summary

Keeping a keen eye on the life of our backend services is important. Not only is it an important "peace of mind" tool, it is also a handy way of extending the reach of our analytics tools.

With Apollo Tracing, we get a level of granularity for our logs that is almost impossible to get with a similar REST stack. This new level of detail allows prioritising engineering efforts, while gauging the return on various fixes and improvements.

What we covered in this post has allowed us to not only provide meaningful feedback on our tech-debt priorities, but also help provide an input to our product decisions, based on what parts of our apps get used the most. Of course, this is only the first step; as any data scientist would tell you, the real art is in separating the signal from the noise and making sense to the data we see.
