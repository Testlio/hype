+++
date = "2016-04-16T09:08:13+03:00"
draft = true
title = "Lambda Backed Microservices (Part 1)"
slug = "lambda-backed-microservices-part-1"
author = "henri"

image = "images/post/lambda-getting-started/cover.jpg"
share = false
comments = false
tags = ["lambdahype"]
+++

In this, the first part of our Lambda posts, we will look at the AWS technologies that make building microservices on Lambda possible - AWS Lambda, AWS API Gateway and AWS CloudFormation<!--more-->

_It is worth noting that throughout this series, we use Node.js as our reference implementation. AWS Lambda does also natively support Python and Java, which some of the concepts we discuss can be translated to._

## AWS Lambda

> Run code without thinking about servers.
> Pay for only the compute time you consume.

That's how AWS [itself positions Lambda](https://aws.amazon.com/lambda/). We like to think of Lambda as a way to focus on business logic and have someone else handle the infrastructure behind it. In reality this involves writing small functions, that are triggered by some **event**, do some **processing** and produce some **output**.

The real power comes from how AWS runs these functions. Every time a suitable event is triggered, AWS loads the correct Lambda function, allocates some compute time to it and runs it. This means we only ever pay for the time our functions run, not for the time they spent waiting for the next event.

The anatomy of a Lambda function is straightforward, a simple "Hello, World!" function can be reduced down to 3 lines of code:

```js
exports.handler = function(event, context) {
    context.succeed('Hello, World!');
};
```

Let's walk through this structure. A Lambda function running on Node.js is a [module](https://nodejs.org/api/modules.html) that has a function `handler`. This function is the main entry point for the Lambda function, taking two parameters  - `event` and `context`. _(We won't discuss the new callback based handlers added for Node.js 4.3 runtime on Lambda)_

The `event` parameter is the event that triggered the function, it often contains input parameters/data that the Lambda is expected to handle. The `context` parameter contains an object that handles the lifetime of the Lambda function.This way we can run asynchronous code in our handler and then call `succeed` or `fail` on the `context` object to finish the Lambda function execution.

It is important to understand that Lambda functions, as any function in general, can have multiple outcomes, that can be generalised to two categories - success or failure. Failures typically include throwing exceptions/errors or returning a value that was not expected, while success in general is optionally returning a result of the computation. These two categories of outcomes are reflected on the `context` object.

Lambda functions can be tied to various different event sources, from [DynamoDB tables](https://aws.amazon.com/dynamodb/) to [S3 buckets](https://aws.amazon.com/s3/) to [AWS Kinesis Streams](https://aws.amazon.com/kinesis/). The rule always being that there is an event that triggers the Lambda function.

So, how would we go about running one of these Lambda functions on a HTTP request? Enter [API Gateway](https://aws.amazon.com/api-gateway/).

## AWS API Gateway

API Gateway acts as the front-door for AWS services, allowing mapping HTTP requests to calls to services such as EC2 or even another HTTP API. It also has native support for AWS Lambda, making it ideal for creating an HTTP API that is completely backed by Lambda.

As we discussed, Lambdas are always executed in response to some event, with the `event` object containing input data. API Gateway
handles mapping a HTTP request to said event, and then also mapping the result of the Lambda function back to a suitable HTTP response.

API Gateway can be set up via the AWS console, however, the more interesting/scalable approach is to use [Swagger](http://swagger.io/specification/) (now also known as OpenAPI) to define our API and tie endpoints to Lambda functions.

The Swagger specification allows extending it, which API Gateway does, by introducing a couple of new keys for HTTP methods on paths - `x-amazon-apigateway-integration` and `x-amazon-apigateway-auth`.

For example, if we wanted to expose our little "Hello, World!" Lambda function over a suitably named `/hello` endpoint on our API, we would need to include something like this under the `paths` key in our JSON Swagger file.

```json
"/hello": {
    "get": {
        "responses": {
            "200": {
                "description": "Hello",
                "schema": {
                    "type": "string"
                },
                "headers": {}
            }
        },
        "parameters": [],
        "x-amazon-apigateway-auth": {
            "type": "none"
        },
        "x-amazon-apigateway-integration": {
            "type": "aws",
            "uri": "our-lambda-function-arn",
            "credentials": "iam-role-arn-for-executing-lambda",
            "httpMethod": "POST",
            "requestParameters": { },
            "requestTemplates": {
                "application/json": "{\"path\": \"$context.resourcePath\"}"
            },
            "responses": {
                "default": {
                    "statusCode": "200",
                    "responseTemplates": {
                        "application/json": ""
                    },
                    "responseParameters": {}
                }
            }
        }
    }
}
```

As you may have noticed, the configuration is excruciatingly explicit, we have to declare all possible HTTP response status codes, response body schemas and headers. We also have to provide a custom mapping from HTTP request to Lambda function event, as can be seen in the `requestTemplates` key, this is then complemented by the `responseTemplates` key that contains a mapping from Lambda function return value to HTTP response.

While the explicit nature of this format is good for making sure API Gateway does exactly what we want it to, it comes with a **steep learning curve** and is **error-prone** when typed up manually.

However, having our entire API in the Swagger format allows us to import it into API Gateway wholesale, reducing the repeated setup cost. Furthermore, the same file can also be used to generate documentation for our API, which comes in handy with any service that wants to expose a public API.

In terms of our goal of not having to deal with servers or their configuration, API Gateway helps us out by quite a lot. Most importantly, it allows us to determine the operating conditions for the API (rate-limiting, caching, authentication), while also taking care of logging by pushing access logs to [AWS CloudWatch](https://aws.amazon.com/cloudwatch/).

Having written a set of Lambda functions, put together an API in a Swagger file, we are now ready to push everything live so that we could actually use the service. Here's where the last piece of the puzzle comes in, CloudFormation, which will take care of raising a stack.

## AWS CloudFormation

> Infrastructure as parameterised code that's declarative and flexible

While AWS allows configuring all of it's services by hand, either through the CLI, the numerous SDKs or the web console, it is often error-prone and tedious to do so. For example, documenting what specific setup a service needs, and then following that documentation every time a deployment occurs, takes far too much valuable engineering time.

This is the problem CloudFormation aims to solve, allowing us to declare our stack in a template file, that can be used to automate the process of deploying and configuring the resources. In our case, these resources involve the Lambda functions, the API Gateway instance and any other AWS services we want to use as part of our service.

The template file itself is another JSON file, the format for which is [exhaustively documented](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-reference.html). _It is worth noting here, as with anything that's documented to the letter, this template is a superb candidate for automation._

As with the Swagger file, the stack template can also end up being several hundred lines long, so I've omitted an example from this post. I would, however, like to point out a trend here - with each level in our stack we have gone from more abstract to more concrete.

Going from the 3 line Lambda function, to the 20 something line Swagger configuration, to a potentially hundred lines of CloudFormation stack template. With each successive step, we have taken more control over our infrastructure, gained more configurability, at the cost of having to understand it and ensuring its' correctness.

We should keep this in mind going forward, as understanding each part individually, allows us to make individual abstractions and tooling for each. Similarly, keeping the levels apart gives us more modularity in the system.

## Proposed Architecture

We have 3 building blocks, levels, from AWS that we can use to design the architecture of our service. The blocks themselves are well-defined by AWS, however, the way they are combined is left up to us.

This brings us to the our proposed service architecture - a layered cake, where each layer knows what's above it, but not what's below it.

Lambda sits blissfully on top, worrying only about functional logic. Below it we have API Gateway, which gives structure to the Lambda functions that are exposed over HTTP. And at the base, we have CloudFormation, which makes sure other components have a solid footing to stand on and all the pipes are correctly connected.

With this in mind, we can imagine a service codebase, that can contains the following parts:

1. A set of Lambda functions
2. An API definition that exposes some or all of said Lambda functions through API Gateway
3. A CloudFormation stack definition, that contains all of the resources the service needs

There are several properties such an architecture has that are worth noting:

1. **Everything is code**. Engineers love code, as it can be _diffed_ and reviewed, it can be kept under source control, it can be trusted.
2. **It is generic**. It doesn't matter whether the service in question deals with users, photos or comments, the structure can always be enforced.
3. **It can benefit from automation**. As mentioned earlier, we can automate the tedious parts, such as writing the API definition or CloudFormation stack template.
4. **It follows separation-of-concerns**. If 3. is applied correctly, engineer dealing with the Lambda functions does not need to know about the guts of the CloudFormation stack.

## Conclusion

In this post, we introduced the AWS technologies that make up the building blocks of our proposed microservice - Lambda, API Gateway and CloudFormation. By investigating each individually, we gained some context to how they work, and what aspects of them could benefit from further abstraction.

Based on our notes, we proposed a service architecture, that would allow us to build a _serverless_ microservice. One that has full control over its own infrastructure, without having to worry about the nitty-gritty of running it.

In the next part, we will discuss implementing this structure in practice and the tools that make it easier and faster to develop such services.
