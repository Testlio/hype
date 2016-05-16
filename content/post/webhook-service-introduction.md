+++
date = "2016-05-16T12:00:00+03:00"

image = "images/post/webhook-service/architecture.png"
comments = false

title = "Building webhooks with Lambda"
slug = "building-webhooks-with-lambdas"
author = "mikk"
tags = ["lambdahype", "dynamodb", "webhooks"]
+++

While [last time]({{< ref "lambda-service-example.md" >}}) we focused on the architecture of a service by presenting a simple made-up example, this time we'll look at a real-world service we recently built to handle webhooks. In this post we are going to discuss why and how we did this.

<!--more-->

### What are webhooks?

First, a bit of background, according to [Wikipedia][wikipedia], webhooks can be described as

> A webhook in web development is a method of augmenting or altering the behavior of a web page, or web application, with custom callbacks. These callbacks may be maintained, modified, and managed by third-party users and developers who may not necessarily be affiliated with the originating website or application. The term "webhook" was coined by Jeff Lindsay in 2007 from the computer programming term Hook.

In short, we could say webhooks are user-defined HTTP callbacks that are triggered by a specific event. In a way, webhooks are like push notifications, used between servers to avoid polling and keep data up to date as it is changed by various events.

### How are we going to use webhooks?

At [Testlio][testlio], webhooks are an integral part of our infrastructure. We use them for variety of different use cases, for example:

- Integrations with third party systems, such as syncing issues to and from clients issue trackers
- Reacting to different service defined events, such as creating a new project
- Connecting services while avoiding [tight coupling][coupling]
- etc.

### Requirements for webhook service

As said earlier, when talking about a webhook service, we talk about one of our core services. One that will be used by multitude of other services. Thus we have to keep a few key requirements in mind:

- We **MUST** have a way to disable triggering for a specific agent (e.g. in case its' API key is leaked)
- We **MAY** have unlimited different listeners for one event
- Each listener **MAY** have specified a secret key, which we'll need to use when signing the request sent to said listener
- When provided with a secret key, it **MUST** be encrypted before storing it
- When provided with a secret key, it **MUST** be used for signing the sent request
- At all times, the listener secrets **MUST** be kept encrypted (e.g when stored in a database)

### Architecture of the service

In order to satisfy these requirements, whilst also keeping in line with the service architecture we have chosen, we decided to base our service on the following technologies:

- [DynamoDB][dynamodb] tables for storing data
- [API Gateway][apigateway] to expose a public API
- [KMS][kms] for encrypting and decrypting secrets provided by listeners (without having to store the private key in the service)

Diagrammatically, the complete architecture of the service looks like this:

![Webhook service architecture](/images/post/webhook-service/architecture.png)

We can clearly identify two separate flows in our service:

1. Registering a hook, which deals with registering new listeners
2. Triggering an event, which deals with making the actual requests

As you can see, the only bridging component between the two flows is a DynamoDB table that stores information about listeners. The two flows themselves are built upon DynamoDB streams and Lambda functions. This approach has several appealing factors:

- Our public API requests should offer constant response times. By using this approach, our event triggering API endpoint is independent of the request making procedure, meaning irregardless of the number of listeners for a given event, the response time for triggering an event should be the same
- We get implicit rate-limiting from our DynamoDB tables, meaning if someone attempts to send out too many events at once, they will be rate-limited by the DynamoDB table
- Separation of concerns - Building a service around DynamoDB streams means we can easily and elegantly separate functionality into the smallest possible functional piece. This gives a huge benefit in terms of understanding and analysing our service
- Visibility - Since our service is built out of small functional "blocks", we can easily see what each of those blocks are doing. This simplifies bug tracking by quite a bit
- Scalability - We can set the throughput on each stream separately. We can also set scaling triggers for each API endpoint separately. This means we can scale the service more precisely, which allows for more precise cost optimisation
- Stability - If something happens to the streams, we can be sure, once restored, the actions continue where the stream left off. Sure, we might lose a couple of minutes, but once the stream is back up, the events will still be sent out in the correct order

Now, let's dive in and take a closer look at the two main flows of our service.

#### Hook registering

![Webhook architecture - Hook registering](/images/post/webhook-service/architecture-hook-registering.png)

_It's important to note, when we speak about a **consumer**, we generally mean another Testlio backend microservice._

Hook registering contains two major steps:  

1. Generating a unique hash for a unique event. For this we take the event type the consumer wishes to expose, and combine it with information we have on the consumer (API key among other things) to generate a unique hash. This step is **essential**, in order to protect our consumers from potential vulnerabilities, where the attacker could just guess the event types.
2. Saving the generated hash with given callbacks and encrypted secret key to a DynamoDB table.

Those two steps are two endpoints:

1. Endpoint that generates the event hashes
2. Endpoint that takes in an event hash, information about the callback and saves it as a listener for a specific event

The main reason for separating these steps into separate endpoints is simple. The consumer **MUST NOT** store the event hashes. Instead, if a service wants to trigger an event, it **MUST** always regenerate a hash for the event.

This is again done for protection, by not storing the event type hash, it is easier for us to invalidate a specific consumer API key. It also means that even through changes to the hashing algorithm, we can maintain an accurate list of all existing listeners for a specific event. So, if for example, a service needs to revoke it's API key, we can simply generate a new one, which it can immediately use to fire off events that will still be delivered to all of its already existing listeners.

For saving the actual event and consumer data to DynamoDB we use KMS to provide encryption for the listener secret (if provided with any). We wouldn't want to save unencrypted keys now, would we?!

So, in short the hook registering flow consists of:

- A hook endpoint that is tied to [API Gateway][apigateway]
- A [Lambda function][lambda] that generates hashes for event types
- A listener registration endpoint in [API Gateway][apigateway]
- A [Lambda function][lambda] for saving the event and listener data to DynamoDB
- [KMS][kms] for encrypting listener secret key (if provided)
- A [DynamoDB][dynamodb] table for saving the listeners

That is the first step of our service - registering hook listeners.

#### Event triggering

![Webhook architecture - Event triggering](/images/post/webhook-service/architecture-event-triggering.png)

The second, more tricky, part of the service is the event triggering flow. Event triggering consists of multiple different steps:

1. Receiving the event from the consumer
2. Saving the received event to a DynamoDB table. For transparency's sake, we save all events that we receive. This gives us the possibility to *replay* events, which is another way to track down bugs
3. Fetching events out of the aforementioned DynamoDB table and checking if there are any listeners for them. If there are listeners for the received event, then we sign the event and save the request object to a separate requests table
4. Processing saved requests from the requests table and making the actual HTTP requests
5. Saving the HTTP request responses back to the requests table

For these steps we need the following:

- A triggering endpoint on [API Gateway][apigateway]
- A [DynamoDB][dynamodb] table with saved listeners
- A [Lambda function][lambda] that saves the received event to the events DynamoDB table
- A [Lambda function][lambda] that sits on the events table stream and generates the actual request objects for the event
- A [KMS][kms] library that decrypts our previously encrypted listener secret and signs the request with it
- A [DynamoDB][dynamodb] table for saving the outgoing requests
- A [Lambda function][lambda] that sits on the requests table stream and sends out the given requests, saving the responses back to the same table

Most of these steps are quite straightforward. They are just Lambda functions that get triggered by either an API Gateway endpoint or by a DynamoDB stream.
However, there is one exception, which we'll look into now.

##### Request sending lambda

Most of our lambda functions are quite straight forward, but there is one that I would like to talk a bit more about - the request sending lambda.

This Lambda function is in charge of executing the outgoing HTTP requests. It is connected to the requests DynamoDB table via its' stream, listening to new rows being added and triggering a request for each of those. It is also responsible for tracking the HTTP responses for said requests.

There are a few caveats to this:

1. The Lambda function sits on a DynamoDB stream
2. The Lambda function triggers HTTP requests
3. HTTP requests are known to fail
4. HTTP request responses must be saved irregardless of the response status

The most important bit from that list is the third point - _HTTP requests are known to fail_. This means, we have to dabble around with our architecture a bit, otherwise we might shoot ourselves in the foot quickly. The problematic part lies in an undocumented _feature_ of DynamoDB streams:

> If a Lambda function that sits on a DynamoDB stream **SHOULD** exit with an error, then the given Lambda function will be tried again.

This means the Lambda function that handles incoming events from the DynamoDB table stream should never fail, as otherwise we can end up in a deadlock, where one batch of events will get sent over and over again because of the failing Lambda function.

The solution we came up with is actually quite a *cool* one - _if I may so myself_.

- Lambda function sitting on the DynamoDB stream will receive as many new images from DynamoDB table as possible
- The function will then check how many different images it received and recursively trigger a new Lambda function N times _(where N is the amount of images received)_
- After triggering the new Lambda functions the stream Lambda function will **ALWAYS** successfully exit

With this approach we can rest assured that we trigger as many requests as we are asked, in the shortest possible time. Furthermore, we can also be sure that with each invocation of our stream, a new batch of requests comes in. Finally, the Lambda function that sends out the HTTP request is safe to fail, as in the worst case, we simply have no response written back to our requests DynamoDB table.

In the future we could build a periodic handler that can retry requests that seem to have not succeeded, or offer the option for the user to retrigger a particular request.

### Conclusion

In this post we looked at a real world example of a microservice built entirely on top of Lambdas and other AWS technologies. The domain for the service, webhooks, is a highly event driven one, which fits perfectly with our chosen approach.

We described ways to ensure certain performance characteristics, for example, how event triggering always appears to take near constant time. Furthermore, we analysed how parts of our service interact with one another, what safeguards can be put into place to ensure the stability of the service, as well as the integrity of both data and order of operations.

It is important to note that this service was one of the first to fully embrace the architecture we have proposed for serverless microservices - you can read more about that in [previous posts]({{< ref "lambda-getting-started.md" >}}).

We are really pleased with how easy it was for us to build something as complex as this service on top of the architecture, we are really looking forward to building even more complex ones next. Make sure to keep an eye on this blog for more posts on how we build things and as always, make sure to let us know what you think on [Twitter][twitter]!

[lambda]: http://docs.aws.amazon.com/lambda/latest/dg/welcome.html
[dynamodb]: https://aws.amazon.com/documentation/dynamodb/
[testlio]: https://testlio.com
[apigateway]: https://aws.amazon.com/api-gateway
[wikipedia]: https://en.wikipedia.org/wiki/Webhook
[kms]: https://aws.amazon.com/kms/
[coupling]: https://en.wikipedia.org/wiki/Coupling_(computer_programming)
[twitter]: https://twitter.com/testlio
