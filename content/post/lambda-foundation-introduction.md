+++
author = "henri"
comments = false
date = "2016-04-25T09:00:00+03:00"
title = "Lambda Backed Microservices (Part 3)"
draft = true
image = "images/post/lambda-foundation-introduction/cover.jpg"

slug = "lambda-backed-microservices-part-3"
tags = ["lambdahype"]
+++

In this part, we will look at [Lambda Foundation](https://github.com/testlio/lambda-foundation), a library, which can help us reduce the boilerplate code in our Lambda functions and make functions across services feel more consistent.

<!--more-->

_This is the third part in our series of posts about AWS Lambda backed microservices. Make sure to check out [Part 1]({{< ref "lambda-getting-started.md" >}}) and [Part 2]({{< ref "lambda-tools-introduction.md" >}})._

## Lambda Foundation

While in [Part 2]({{< ref "lambda-tools-introduction.md" >}}) we looked at Lambda Tools, a toolchain to help us manage deployment and local execution of our services, in this part we will drill down even further.

We will look at [Lambda Foundation](https://github.com/testlio/lambda-foundation), a library that consists of common code useful for Lambda functions across all services. This functionality includes configuration, error reporting, authentication, unit testing, model layer and more. For the purposes of this post, we will focus on 3 key features - the model layer, unit testing and configuration.

### Model Layer

One of the key functionalities for a Lambda backed service is interacting with a persistent model store, such as [DynamoDB](https://aws.amazon.com/dynamodb/). This functionality more than often includes all of the basic [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) operations, as well as validation and helper logic. In the case of Node.js, there are several libraries that offer a nice abstraction over DynamoDB - one of those being [Vogels](https://github.com/ryanfitz/vogels).

In Vogels you can quickly define a model schema that corresponds to a DynamoDB table

```js
var User = vogels.define('User', {
    hashKey: 'email',
    schema: {
        email: Joi.string().email(),
        name: Joi.string(),
        age: Joi.number()
    }
});
```

Once done, you can then perform all of the basic CRUD operations on the new resource:

```js
User.create({
    email: 'henri@testlio.com',
    name: 'Henri',
    age: 25
}, function(err, data) {
    console.log('New user', data.attrs);
});

User.get('henri@testlio.com', function(err, data) {
    console.log('Found user', data.attrs);
});
```

Lambda Foundation adds another layer of abstraction on top of Vogels, _Promisifying_ the API, as well as adding some conveniences. Creating a model that is based on Lambda Foundation is very similar to Vogels, in fact, it is almost identical:

```js
var User = foundation.model.define('User', {
    hashKey: 'email',
    schema: {
        email: Joi.string().email(),
        name: Joi.string(),
        age: Joi.number()
    }
});

// Use the better, promisified API
User.create({
    email: 'henri@testlio.com',
    name: 'Henri',
    age: 25
}).then(function(user) {
    // The value is immediately unpacked, no more .attrs needed
    console.log('New user', user);
});

User.find('henri@testlio.com').then(function(user) {
    console.log('Found user', user);
});
```

Using promises means we can structure our Lambda function code nicer, avoiding _callback hell_ and clearly separating functional parts of our Lambda function.

```js
exports.handler = function(event, context) {
    // Our event contains an email, let's look up the user
    User.find(event.email)
    .then(context.succeed, context.fail);
};
```

Using promises and chaining them gives our Lambda function code several nice qualities:

1. The code is flatter and the flow is uni-directional, making it easier to read
2. Code is more modular, as different processing steps can be separated into `.then` clauses on promises
3. The function encourages a single point of exit, helping to avoid the Lambda function incorrectly exiting without a `context.succeed` or `context.fail` call

Although we won't go into detail on error-reporting and authentication in Lambda Foundation, it is clear how these benefits come into play when adding those features to our Lambda function.

```js
var auth = require('lambda-foundation').authentication;
var Error = require('lambda-foundation').error;

exports.handler = function(event, context) {
    // Authenticate the event, if any error occurs, report it
    auth.authenticate(event.authorization)
    .then(function() {
        return User.find(event.email);
    })
    .then(context.succeed, function(err) {
        return Error.report(err).then(context.fail);
    });
};
```

### Configuration

Due to its nature, using configuration files in AWS Lambda can be quite tricky. A common trend in Node.js servers would be to have something like the [config](https://www.npmjs.com/package/config) package that allows swapping between configurations depending on the running environment. However, there are a few limitations that stop us from using the same approach in Lambda:

1. There is no way for us to modify the environment variables in Lambda, unless done so from the Lambda code itself
2. Ideally we want to bundle the Lambda function into a single file, removing any unused code, such as configuration options that are not used.

Here's where the configuration part of Lambda Foundation comes in. On the surface it feels very similar to the aforementioned _config_ package, however, there are some differences. First, during the bundling process, the configuration is flattened into the code, i.e all of the appropriate configuration files are loaded in. Furthermore, as the environment variables are not modifiable after a Lambda function is deployed, we can make use of this and drop any _unreachable_ configurations altogether.

In order to maintain some configurability, Lambda Tools allows us to define what the environment variables _with which_ the Lambda function code is bundled. Meaning we can still toggle between configurations in different deployments, however, we need to know beforehand.

For example, given the following service structure:

```sh
.
├── README.md
├── api.json
├── cf.json
├── config
│   ├── development.json
│   └── production.json
├── lambdas
│   ├── bar
│   │   └── index.js
│   └── foo
│       └── index.js
└── package.json
```

Deploying to the dev vs production stage would look something like this:

```sh
lambda deploy -s dev -e NODE_ENV=development
lambda deploy -s prod -e NODE_ENV=production
```

In either case, either `config/development.json` or `config/production.json` would be loaded. While not as dynamic as using other methods for configuring the service behavior, this approach is good enough for most cases. Conceptually, a configuration file could also be loaded from some remote location, such as S3. However, this might make the response-time of the Lambda function slower, acting more as an overhead.

### Unit Testing

While Lambda Tools has `lambda execute` and `lambda run`, allowing us to locally test the functionality of our Lambda code, it is always a good idea to cover the core parts of our service with unit-tests.

There are a huge variety of testing libraries out there for Node.js, choosing the _best_ one is an opinionated topic. In Lambda Foundation we looked at various libraries and finally settled on [Tape](https://npmjs.org/package/tape). Some of the reasons we decided to go with Tape are listed in this [excellent post by Eric Elliot](https://medium.com/javascript-scene/why-i-use-tape-instead-of-mocha-so-should-you-6aa105d8eaf4#.tu5dqox6v).

Combining the simplicity of Tape and the utility provided by Lambda Foundation, we can write a test case as:

```js
var foundation = require('lambda-foundation');
var context = foundation.test.context;
var Event = foundation.test.event;

var tape = require('tape');

tape.test('Hello, world', function(t) {
    // Load in the Lambda code
    var lambda = require('../lambdas/hello/');

    t.test('Should succeed', function(it) {
        var event = new Event({ property: true });

        // Assert that context.succeed is called, with an expected value
        var mockContext = context.assertSucceed(it, 'Hello, World!');

        // Execute the Lambda code with the mock context and a fake event
        lambda.handler(event, mockContext);
    });

    t.test('Should fail', function(it) {
        var event = new Event({ property: false });

        // Assert that context.fail is called, with an optional expected error
        var mockContext = context.assertFail(it);

        // Execute the Lambda code with the mock context and a fake event
        lambda.handler(event, mockContext);
    });
});
```

The key problem solved by Lambda Foundation here is what `event` and `context` values need to be sent to the Lambda function. Apart from helping out with creating mock event and context objects, the testing submodule also plays well with authorization aspect, allowing quickly adding authorization tests to a Lambda function.

The testing submodule also helps with sandboxing, which comes in handy when stubbing out service calls that our Lambda function may make to other AWS resources.

```js
var foundation = require('lambda-foundation');
var context = foundation.test.context;
var Event = foundation.test.event;
var tape = foundation.test.test;

tape.test('Hello, world', function(sandbox, t) {
    // Stub out something like S3 or DynamoDB
    sandbox.stub(awsModule, 'method', function(param) {
        // Return a stubbed value
        return { value: 'bar' };
    });

    // Load in the Lambda code
    var lambda = require('../lambdas/hello/');

    t.test('Should succeed', function(it) {
        var event = new Event({ property: true });

        // Assert that context.succeed is called, with an expected value
        var mockContext = context.assertSucceed(it, 'Hello, World!');

        // Execute the Lambda code with the mock context and a fake event
        lambda.handler(event, mockContext);
    });
});
```

The sandbox that the testing package provides is from [Sinon](http://npmjs.com/package/sinon) and is destroyed once all of the test cases are completed.

## Conclusion

In this part we looked at [Lambda Foundation](https://github.com/testlio/lambda-foundation), a library that helps us with common functionality in our Lambda functions. The library helps us with common aspects, such as configuration, interaction with the model layer, error reporting, authentication as well as unit-testing.

It also implicitly enforces some good patterns on the code, such as separating the different parts of the Lambda function by using nice promise-chains. This means that services are consistent on the function level, making it easier for engineers to go from one service/function to another.

In the next post, we will start building an example service, using all of the tools and background information we have gathered from parts [1]({{< ref "lambda-getting-started.md" >}}), [2]({{< ref "lambda-tools-introduction.md" >}}) and [3]({{< ref "lambda-foundation-introduction.md" >}}).
