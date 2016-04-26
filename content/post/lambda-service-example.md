+++
date = "2016-04-28T09:00:00+03:00"
share = false
draft = true

image = "images/post/lambda-service-example/cover.png"
comments = false

title = "Lambda Backed Microservices (Part 4)"
slug = "lambda-backed-microservices-part-4"
author = "henri"
tags = ["lambdahype"]
+++

In this part, we will combine all of what we learned in previous parts to build a simple example service. We will go through the process of setting up the service, making sure its configuration is correct and we'll finish off by deploying the service to AWS.

<!--more-->

This part is going to be a longer one, so sit back, grab a notepad or a cup of coffee and let's get started. For those who want to play along, make sure that you have configured your PC to work with AWS CLI/SDK, as well as installing Node and npm (anything above 5.0.0 and 3.0 is sufficient).

_This is the fourth part in our series of posts about AWS Lambda backed microservices. Make sure to check out [Part 1]({{< ref "lambda-getting-started.md" >}}), [Part 2]({{< ref "lambda-tools-introduction.md" >}}) and [Part 3]({{< ref "lambda-foundation-introduction.md" >}})._

## Background

The service we are going to build in this post and all of its source code is available on [GitHub](). You can try out the service by directly talking to [AWS API Gateway]() or by using the front-end app hosted at [demo.testlio.com](https://demo.testlio.com).

The service we are going to build is a straightforward one, consisting of two models - questions and answers. The service will have two endpoints, one for getting a random question and the other for answering a question. When answering a question, the service also allows specifying a callback URL, which will be called when the answer has been stored. The last part, although functionally serving little purpose, demonstrates how we can attach Lambda functions to DynamoDB streams.

The architecture of our small service can be visualised as follows (_diagram courtesy of [CloudCraft](https://cloudcraft.co)_):

![Service architecture](/images/post/lambda-service-example/architecture.png)

## Setting up our workspace

Before we start writing our service, let's set up a workspace, including tools we introduced in this series.

_For the purposes of this post, we assume all of the following commands are run on a command line, in a new directory that will serve as the root of the service. We also assume that you already have both Node.js and NPM installed._

As with all other posts in this series, we use Node.js as our reference implementation and npm as our dependency manager.

First, let's initialise the service package, install Yeoman and the generators.

```sh
$ npm init
name: (demo-service) demo-service
version: (1.0.0)
description: Demo for Lambda Tools, Lambda Foundation and their service architecture
entry point: (index.js)
test command:
git repository:
keywords:
author:
license: (ISC)
About to write to /Users/henrinormak/Work/Services/demo-service/package.json:

{
  "name": "demo-service",
  "version": "1.0.0",
  "description": "Demo for Lambda Tools, Lambda Foundation and their service architecture",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC"
}


Is this ok? (yes)

$ npm install -g yo generator-lambda-tools
```

Once these steps are done, we can run the main generator and bootstrap our service.

```sh
$ yo lambda-tools
? Service name demo-service
? Service description Demo for Lambda Tools, Lambda Foundation and their service architecture
? License (API) ISC
? Author email you@yourdomain.com
? Author name That is You
? Install dependencies lambda-tools, lambda-foundation
 conflict package.json
? Overwrite package.json? overwrite
    force package.json
   create cf.json
   create api.json
   create .lambda-tools-rc.json

$ npm install --save joi needle promiscuous
```

This generates the stubs for all of our service configuration files, as well as installing our libraries. We also install a couple of libraries that we'll need down the line. Once we have these in place, we can move on to fleshing out our service.

## Building the service

### Questions

First thing we'll do is define our Questions model. Once this is done, we can code up a Lambda function to sit behind the `/questions/random` endpoint. This endpoint will fetch a random question from our DB and return it to the caller.

Let's start by defining our model, using Lambda Foundation for this. Create a file in `lib/models/questions.js` and paste the following code into it:

```js
'use strict';

const Joi = require('joi');
const model = require('lambda-foundation').model;

module.exports = model('Questions', {
  hashKey: 'guid',
  timestamps: true,
  schema: {
    guid: model.types.uuid(),
    question: Joi.string()
  }
});
```

This code defines a simple Question model, which has two properties - `guid` and `question`. We also enable timestamps on rows (which means `createdAt` and `updatedAt` values are automatically added). Once we have defined our model code, we need to make sure we include an appropriate DynamoDB table in our CloudFormation stack.

In order to do this, we can simply run the `dynamo-table` generator:

```sh
$ yo lambda-tools:dynamo-table
? Table name questions
? CloudFormation Resource name QuestionsDynamoDB
? Key schema type Hash Key
? Hash key attribute name guid
? Hash key attribute type String
? Include in Lambda access policies Yes
 conflict cf.json
? Overwrite cf.json? overwrite
    force cf.json
   create lambda_policies.json
```

Now we have everything to build out our first Lambda function. This function will sit behind API Gateway at `GET /questions/random` and return a random question from our DynamoDB.

We can use another generator to create the starting point for our endpoint:

```sh
$ yo lambda-tools:endpoint
? Path for the endpoint /questions/random
? HTTP Method GET
? Lambda function name questions-random-get
? Map HTTP headers? No
 conflict api.json
? Overwrite api.json? overwrite
    force api.json
   create lambdas/questions-random-get/index.js
   create lambdas/questions-random-get/event.json
```

Opening up `lambdas/questions-random-get/index.js` we are greeted with the familiar _"Hello, World!"_ Lamba function we saw in [Part 1]({{< ref "lambda-getting-started.md" >}}). Modify the code to the following, the code includes comments explaining the different parts, so we won't go into too much detail about the exact functionality. In general, the code uses the model we just defined, grabbing all questions and then picking one at random. _This code is not the best approach, but for example purposes it is good enough._

```js
'use strict';

const Questions = require('../../lib/models/questions.js');
const LambdaError = require('lambda-foundation').error;

exports.handler = function(event, context) {
    // Lambda Foundation model is a wrapper around
    // Vogels, promisifying the API
    Questions.scan().exec().then(function(questions) {
        // Pick a random question, if no questions were found
        // then fail with error 404
        if (!questions || questions.length === 0) {
            throw new LambdaError(404, 'No questions found');
        }

        const idx = Math.floor(Math.random() * questions.length);
        return questions[idx];
    })
    .then(context.succeed)
    .catch(context.fail);
};

```

Notice that we fail with a custom error type, `LambdaError`. This error type makes sure that the description of the error always puts a HTTP status code as the first thing. This is needed to make sure API Gateway can properly map the error to a HTTP status code. In order to nicely throw a HTTP 404 error, we also need to run the `endpoint-response` generator:

```sh
$ yo lambda-tools:endpoint-response
? Add a response to path /questions/random
? HTTP Method GET
? Status code 404
? Response name/pattern 404.*
? Response description Not Found
? Response template MIME type application/json
? Create response template? Yes
? Response template {"message": "Not Found"}
? Include any HTTP headers? No
 conflict api.json
? Overwrite api.json? overwrite
    force api.json
```

#### Testing it out locally

Let's now try out our one endpoint service. We can use Lambda Tools for this, as explained in [Part 2]({{< ref "lambda-tools-introduction.md" >}}), by simply running `$(npm bin)/lambda run`. However, as we use a DynamoDB table in our code, we must first deploy the service as otherwise Lambda Foundation will not be able to connect to the table (as it doesn't exist). After running `$(npm bin)/lambda deploy`, we can either try out our service by going to the API Gateway console and trying it out there, or by simply running `$(npm bin)/lambda run`.

The latter starts a web server that exposes port `3000` on `localhost` for our service. We can then issue requests against our service to try out our service. In the example, we are using [httpie](https://github.com/jkbrzt/httpie), which is a nicer way of using curl.

```http
$ http localhost:3000/questions/random
HTTP/1.1 404 Not Found
Connection: keep-alive
Content-Length: 26
Content-Type: application/json; charset=utf-8
Date: Tue, 26 Apr 2016 08:05:41 GMT

{
    "message": "Not Found"
}
```

As expected, we receive a 404, as we haven't added any questions to our DynamoDB table. For the purposes of this example, you can use the AWS console to [add a few questions](http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/AddUpdateDeleteItems.html#AddItemUsingConsole). Once we have done that, we can re-execute our curl. This time we should get back a single question.

```http
$ http localhost:3000/questions/random
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 111
Content-Type: application/json; charset=utf-8
Date: Tue, 26 Apr 2016 08:02:40 GMT

{
    "guid": "6f3ecb65-86b2-4ff8-abb4-77f60d6ee420",
    "question": "What is \"callback hell\" and how can we avoid it?"
}
```

Perfect, we can now retrieve questions from our DynamoDB table. Let's move on to the next part, which is adding answers to said questions.

### Answers

As with questions, we are going to start from the model, building up to the endpoint. Let's create another file under `/lib/models` called `answers.js`, with this inside:

```js
'use strict';

const Joi = require('joi');
const model = require('lambda-foundation').model;

module.exports = model('DemoQuestionsAnswers', {
    hashKey: 'email',
    timestamps: true,
    schema: {
        questionGuid: Joi.string().guid().required(),
        answer: Joi.string().required(),
        callback: Joi.string().uri({
            scheme: [
                'http',
                'https'
            ]
        })
    }
});
```

As with the questions model, the answers model is also fairly straight forward. We store a reference to the question, the answer given by the user and a callback href that will get notified by the stream later on. Based on this definition, we can create our DynamoDB table.

```sh
$ yo lambda-tools:dynamo-table
? Table name answers
? CloudFormation Resource name AnswersDynamoDB
? Key schema type Hash Key
? Hash key attribute name guid
? Hash key attribute type String
? Include in Lambda access policies Yes
 conflict cf.json
? Overwrite cf.json? overwrite
    force cf.json
 conflict lambda_policies.json
? Overwrite lambda_policies.json? overwrite
    force lambda_policies.json
```

Similarly, we use the same rinse and repeat strategy when creating our endpoint. First run the generator:

```sh
$ yo lambda-tools:endpoint
? Path for the endpoint /questions/{guid}/answers
? HTTP Method POST
? Lambda function name questions-answers-post
? Map request body to event property (leave blank to skip) payload
? Is path parameter 'guid' required? Yes
? Map HTTP headers? No
 conflict api.json
? Overwrite api.json? overwrite
    force api.json
   create lambdas/questions-answers-post/index.js
   create lambdas/questions-answers-post/event.json
```

Notice that this time, we included a parameter in our URL, this is a handy way of creating a much nicer API, while also grabbing some required parameters directly from the URLs. Similarly to last time, we'll need a 404 response, so once again we can run the `endpoint-response` generator, just for the new endpoint. For sake of clarity, I won't reproduce that code here.

Once we have done that, we can open up our new Lambda function in `lambdas/questions-answers-post/index.js` and write in the code to handle 2 things:

1. First we need to look up that the question exists, if it doesn't then we must fail with a 404
2. If the question does exist, we need to store the answer

```js
'use strict';

const FoundationError = require('lambda-foundation').error;
const Questions = require('../../lib/models/questions.js');
const Answers = require('../../lib/models/answers.js');

exports.handler = function(event, context) {
    // Find the question
    return Questions.find(event.guid).then(function(question) {
        if (!question) throw new FoundationError(404, 'No question found');
        return question
    })
    .then(function(question) {
        // Try to create the answer, this will throw an error if the payload
        // is invalid, if it succeeds we can finish our processing with the
        // new answer
        return Answers.create({
            questionGuid: event.guid,
            answer: event.payload.answer,
            callback: event.payload.callback
        });
    })
    .then(context.succeed).catch(context.fail);
};
```

#### Trying it out locally

As previously, we can once again run the service locally and try out our new endpoint for adding answers. As with questions, we also need to deploy first, so that our dev stage has a DynamoDB table for answers. We will look into how this can be done locally in the future, make sure to subscribe to the blog for that.

We can look up a GUID from the response of `/questions/random`, and use that to send a request to `/questions/{guid}/answers`. In my specific case this is a POST request to `/questions/6f3ecb65-86b2-4ff8-abb4-77f60d6ee420/answers`:

```http
$ http POST localhost:3000/questions/6f3ecb65-86b2-4ff8-abb4-77f60d6ee420/answers answer="It is a nightmare" callback=http://example.com/test

HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 206
Content-Type: application/json; charset=utf-8
Date: Tue, 26 Apr 2016 08:07:31 GMT

{
    "answer": "It is a nightmare",
    "callback": "http://example.com/test",
    "createdAt": "2016-04-26T08:07:30.957Z",
    "guid": "b2da8b7f-a275-4d45-9f8f-91d654179af1",
    "questionGuid": "6f3ecb65-86b2-4ff8-abb4-77f60d6ee420"
}
```

Excellent, we can now both fetch a random question as well as submit a new answer. In a production service, this would be the point where we would add tests to both of these endpoints, as well as any additional CRUD endpoints we might need.

As an excercise you can try adding the complementary GET for a specific answer, an endpoint that sits on `/questions/{questionGuid}/answers/{answerGuid}` for example, where you can validate both the answer GUID as well as the question GUID and return the answer if one is found.

## Stream

By now, we have a service, which has two HTTP endpoints, backed by two Lambda functions. As additional resources we also have two DynamoDB tables that store our questions and answers. And all this in a relatively short period of time, pretty cool. The last piece we will add is purely to demonstrate the power of AWS Lambda when combined with event sources such as DynamoDB.

We will create a new Lambda function, that sits on the stream of our answers table, reacting every time a new answer is added. The Lambda function will look whether the answer has a callback defined, and if so, trigger a request to said callback with the contents of the DynamoDB item. In broad strokes, this is a way to implement webhooks that are triggered when something gets added or updated in our DB.

First, let's start by running yet another generator, this time the `dynamo-stream` one. This will create a new Lambda function and hook it up to the stream of one our DynamoDB tables.

```sh
$ yo lambda-tools:dynamo-stream
? DynamoDB resource to attach the stream to AnswersDynamoDB
? Stream resource name AnswersDynamoDBStream
? Lambda function name answers-webhook
? Batch size 1
? Enable the stream Yes
? Starting position TRIM_HORIZON
? Stream view type for the table NEW_IMAGE
 conflict cf.json
? Overwrite cf.json? overwrite
    force cf.json
 conflict lambda_policies.json
? Overwrite lambda_policies.json? overwrite
    force lambda_policies.json
   create lambdas/answers-webhook/index.js
```

The newly created Lambda function at `lambdas/answers-webhook/index.js` is slightly different from the ones we've seen thus far. While the boilerplate code in it is the same, the way it is triggered is different. This Lambda function is not tied to any endpoint in our API Gateway. Instead, it is tied to the stream of the answers DynamoDB table.

As such, we can't test it out locally by invoking an HTTP endpoint. Instead, we have to leverage another script in Lambda Tools, namely, `lambda execute`. This script is used for executing a single Lambda function, with a predefined event that can be read from a file. This is perfect for setting up Lambda functions that don't sit behind API Gateway for local testing.

First, let's configure the event file at `lambdas/answers-webhook/event.json`. This is the event our Lambda function will receive when executed locally. In AWS, this event will be generated by DynamoDB, so we can look up the structure by investigating [the documentation](http://docs.aws.amazon.com/lambda/latest/dg/with-dynamodb-create-function.html).

A modified event, which would correspond to our answers table would look something like this:

```json
{
    "Records":[
        {
            "eventID":"1",
            "eventName":"INSERT",
            "eventVersion":"1.0",
            "eventSource":"aws:dynamodb",
            "awsRegion":"us-east-1",
            "dynamodb":{
                "Keys":{
                    "guid":{
                        "S":"b2da8b7f-a275-4d45-9f8f-91d654179af1"
                    }
                },
                "NewImage":{
                    "guid":{
                        "S":"b2da8b7f-a275-4d45-9f8f-91d654179af1"
                    },
                    "answer": {
                        "S":"It is a nightmare"
                    },
                    "questionGuid": {
                        "S":"6f3ecb65-86b2-4ff8-abb4-77f60d6ee420"
                    },
                    "createdAt": {
                        "S":"2016-04-26T08:07:30.957Z"
                    },
                    "callback": {
                        "S":"http://example.com/test"
                    }
                },
                "SequenceNumber":"111",
                "SizeBytes":70,
                "StreamViewType":"NEW_IMAGE"
            },
            "eventSourceARN":"stream-ARN"
        }
    ]
}
```

Once we have stored this to `event.json`, we can run the following from the root of our service:

```sh
$ $(npm bin)/lambda execute answers-webhook

Executing: /Users/henrinormak/Work/Services/demo-service/lambdas/answers-webhook/index.js
	--
    With event:
	{
		"Records": [
			{
				"eventID": "1",
				"eventName": "INSERT",
				"eventVersion": "1.0",
				"eventSource": "aws:dynamodb",
				"awsRegion": "us-east-1",
				"dynamodb": {
					"Keys": {
						"guid": {
							"S": "b2da8b7f-a275-4d45-9f8f-91d654179af1"
						}
					},
					"NewImage": {
						"guid": {
							"S": "b2da8b7f-a275-4d45-9f8f-91d654179af1"
						},
						"answer": {
							"S": "It is a nightmare"
						},
						"questionGuid": {
							"S": "6f3ecb65-86b2-4ff8-abb4-77f60d6ee420"
						},
						"createdAt": {
							"S": "2016-04-26T08:07:30.957Z"
						},
						"callback": {
							"S": "http://example.com/test"
						}
					},
					"SequenceNumber": "111",
					"SizeBytes": 70,
					"StreamViewType": "NEW_IMAGE"
				},
				"eventSourceARN": "stream-ARN"
			}
		]
	}
    --

	--
	Result '"Hello!"'

Executing: /Users/henrinormak/Work/Services/demo-service/lambdas/answers-webhook/index.js ✔

```

As we can see, the event gets properly ingested and passed to the Lambda function. Now all that remains is to implement the Lambda function to handle the event.

```js
'use strict';

const needle = require('needle');
const Promise = require('promiscuous');

// Helper function for sending out a request via needle
function sendRequest(href, answer, questionGuid, guid) {
    return new Promise(function(resolve, reject) {
        const data = {
            answer: answer,
            questionGuid: questionGuid,
            guid: guid
        };

        needle.post(href, data, { json: true }, function(err, response) {
            if (err) return reject(err);
            resolve({
                statusCode: response.statusCode,
                message: response.statusMessage,
                body: response.body
            });
        });
    });
}

exports.handler = function(event, context) {
    // Grab all records from the event and filter out only those
    // that are new items in the DB
    const records = [].concat(event.Records).filter(function(record) {
        return record.eventName === 'INSERT';
    });

    // No events, then we can exit early
    if (!records) {
        return context.succeed('No events to handle');
    }

    // Map all records to requests that need to be sent out
    const requests = records.map(function(record) {
        const obj = (record.dynamodb || {}).NewImage;
        const answer = obj.answer.S;
        const questionGuid = obj.questionGuid.S;
        const guid = obj.guid.S;

        // Callback was optional
        const href = obj.callback ? obj.callback.S : undefined;

        if (!href) {
            return Promise.resolve();
        } else {
            return sendRequest(href, answer, questionGuid, guid);
        }
    });

    // Send out all the requests, if any fails, we fail
    // otherwise just succeed with the responses
    Promise.all(requests).then(context.succeed).catch(context.fail);
};
```

While this Lambda function isn't as succinct as the other functions were, the code is still straightforward to understand and should read quite easily. We unpack the records from the event and then trigger a separate POST request for each answer that had a callback defined.

Once we deploy this, we can try it out by using something like [RequestBin](http://requestb.in/) for our callback and submitting a new answer. For example, after we have deployed we can execute something like this locally (make sure to change the callback to your RequestBin address):

```sh
$ http POST localhost:3000/questions/6f3ecb65-86b2-4ff8-abb4-77f60d6ee420/answers answer="It is a nightmare" callback=http://requestb.in/181ogam1
...
```

And once we've done this, we can open up our RequestBin and see that the callback has indeed arrived!

![RequestBin Contents](/images/post/lambda-service-example/request-bin.png)

## Conclusion

In this, the final post of our first [#lambdahype](/tags/lambdahype) series, we combined everything we learned in the previous part to build our first Lambda backed microservice. The service had two endpoints, two DynamoDB tables and made use of DynamoDB streams to trigger a callback whenever a new answer was added.

As mentioned before, you can try out the service at [demo.testlio.com](https://demo.testlio.com) and go through the source code of the service in [GitHub](). There are some nuances that we didn't cover in the post, such as CORS support for the service. The published source code includes all these, and the general gist of adding it involves executing another generator for a couple of times.

All in all, we are very eager to see what you think of our approach to serverless microservices. Feel free to submit issues and pull requests to any of our [open-source](https://github.com/testlio/lambda-tools) [repositories](https://github.com/testlio/lambda-foundation) for [Lambda](https://github.com/testlio/generator-lambda-tools). Long live **#lambdahype**!
