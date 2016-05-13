+++
date = "2016-05-12T09:00:00+03:00"

image = "images/post/webhook-service/architecture.png"
comments = false

title = "Building webhooks with lambdas"
slug = "building-webhooks-with-lambdas"
author = "mikk"
tags = ["lambdahype", "dynamodb", "webhook"]
+++

Recently we implemented a new inner service called webhook service. In this post we are going to discuss how and why we implemented that service as we did. 

<!--more-->

### What are webhooks?

> A webhook in web development is a method of augmenting or altering the behavior of a web page, or web application, with custom callbacks. These callbacks may be maintained, modified, and managed by third-party users and developers who may not necessarily be affiliated with the originating website or application. The term "webhook" was coined by Jeff Lindsay in 2007 from the computer programming term Hook. [wikipedia][wikipedia]

**TL;DR** We could say webhooks are user-defined HTTP callbacks that are triggered by a specific event.

### How are we going to use the given hooks?

For us, [Testlio][testlio], webhooks are a integral part of our infrastructure. We use them for variety of different reasons, for example:

- integrations with third party systems, such as syncing issues to and from clients issue trackers
- reacting to different service defined events, such as creating a new project
- etc

### Requirements for webhook service

As said earlier then when we are talking about the webhook service we are talking about one of core services that will be used by multitude of other services. Thus we have to keep a few key things in mind:

- we **MUST** have a way to turn off triggering for a specific service (e.g. in case it's secret is exposed)
- we **MAY** have unlimited different consumers for one event
- each consumers **MAY** have a specified secret key with what we sign all the requests to that consumer
- if provided with a secret key, then that secret **MUST** be encrypted before we store it
- if provided with a secret key, then we **MUST** encrypt the sent request with that key
- at all times we **MUST** keep the clients secret encrypted

### Architecture of the service

When building this service we decided to use the following technologies:

- [Dynamodb][dynamodb] tables for holding our information
- [Api gateway][apigateway] to expose public endpoints
- [KMS][kms] for encrypting and decrypting secrets provided by consumers

The complete architecture of the service looks like this:

![webhook architecture](/images/post/webhook-service/architecture.png)

Here we can see two different main parts of our service:  

1. hook registering that deals with registering consumers.
2. event triggering that deals with making the actual requests.

As you can see then we decided to link those parts together by using dynamodb streams and lambdas. There are a few reasons behind that:  

- Constant speed for our gateway requests. When using an approach we can always say that endpoints that sit behind our api gateway have pretty much constant response time.   
When triggering an event then they only have to worry about writing ONE entry to one table. The only real exception is the endpoint that will list out all the event listeners.
- Separation of concerns. When building a service around dynamodb streams then we can easily and elegantly separate functionality into smallest functional pieces possible. This gives us a huge benefit in terms of understanding whats going and fixing bugs.
- Visibility. Since we built our service out of small "blocks" we can easily see what each of those blocks are doing. This also makes the potential bug tracking much easier.
- Scalability. We can set read and write throughput on each stream separately. We can also set scaling triggers for each api endpoint separately. This means that we can scale things more precisely, which also means that its cheaper.
- Stability. If something happens with the streams we can be sure that when they are restored then the action continues where it left off. Sure, we might lose a minute or so, but if the stream is back up then we always know that the events will be sent in the correct order.

Now lets take a closer look on two two main different part of our systems.

#### Hook registering

![webhook architecture hook registering](/images/post/webhook-service/architecture-hook-registering.png)

Its important to note here that when we say **consumer** then we mean another Testlio service.

Hook registering contains of two major steps:  

1. Generating a unique hash about the event. For that we take the given event to register and generate a unique hash based on the information about the consumer (api-key among other things) and the event itself.  
This step is **essential** in order to protect our consumers from potential vulnerabilities where the attacker could just guess the event names.
2. Saving the generating hash with given callbacks and encrypted secret key to a specified dynamodb table.

Those two steps are two endpoints:

1. endpoint that calculates event hashes based on the information about the consumer and the event itself.
2. endpoint that takes in the given calculated hash and saves it as a consumer trigger event.

The main reasoning behind that is that a service that consumes webhooks **MUST NOT** save event hashes.  
Instead if a service wants to trigger an event it **MUST** always regenerate a new hash of an event.  

This is done so for security purposes. For instance if an service access key is exposed for some reason we can quickly change it and since the api-key is a part of the hash then those old events can never be triggered again.

For saving the actual event and consumer data to dynamodb we also use KMS to provide encryption for the clients key (if provided).  
We wouldn't want to save unencrypted keys now, would we?!

So to recap:  

- Hook endpoint that is tied to [api gateway][apigateway]
- [Lambda function][lambda] for hashing the request 
- [KMS][kms] for encrypting clients secret key (IF PROVIDED)
- [Lambda function][lambda] for saving the event and consumer to dynamo table
- [Dynamodb][dynamodb] table for saving the consumers

That is the first step of our service - registering hook listeners.

#### Event triggering

![webhook architecture event triggering](/images/post/webhook-service/architecture-event-triggering.png)

Now for the tricky part. Event triggering logic.  
Event triggering consist of multiple different steps:  

1. Receiving the given event from the requester.
2. Saving the received event to a dynamodb table. For transparency sake we save all the events that we receive. This gives us potential *replay* possibilities as well as a really easy way to trace actual bugs.
3. Fetching events out of the aforementioned and checking if there are any listeners for the given event. In case there are listeners to the received event, then we sign the event and save the request object to the requests table.
4. Fetching saved events out of the requests table and actually making the given HTTP requests. 
5. Saving request responses back to the requests table.


For those steps we need the following this:  

- Hook endpoint that is tied to [api gateway][apigateway]
- [Dynamodb][dynamodb] table with saved consumers
- [Lambda function][lambda] that saves the received event to events dynamodb table
- [Lambda function][lambda] that sits on events table stream and will generate actual request objects from that given event
- [KMS][kms] library that decrypts our previously encrypted clients secret and encrypt the request with that 
- [Dynamodb][dynamodb] table where request objects are saved to
- [Lambda function][lambda] that sits on requests table and will actually send out the given requests and save the responses back to the given table

Most of those steps are quite plain and simple. They are just lambda functions that get triggered by either an api-gateway call or a dynamodb stream.  
However as with all good things there are exceptions and the next chapter will briefly go over that.

##### Request sending lambda

Most of our lambda functions are quite straight forward, but there is one that I would like to talk a bit more about - the request sending lambda.
For a quick recap:

Request sending lambda is the lambda that sits on requests table, listens to new item adding to the table and will trigger the same amount of requests. It is also responsible about tracking the requests responses.

Now here things get hairy and there are a few reasons behind that:

1. this lambda sits on stream
2. this lambda triggers requests
3. request can fail
4. request responses must be saved regardless of the response status

The most important bit from that list is the third point "request can fail". This means that we have to dabble around with our architecture a bit, otherwise we might shoot ourselves in the foot.  
Main reason for that is the fact that there is a *small* undocumented feature with dynamodb streams:

> If a lambda that sits on a dynamodb stream **SHOULD** exit with an error then the given lambda will be tried again.

This means that we have to be a bit more careful, otherwise we could write ourselves in a deadlock, where one patch event will get sent over and over again because of the failing lambda. 

For us the solution for that problem was quite a *cool* one (if I may say so). The solution was: 

- lambda that sits on the stream will receive as many new entities from dynamodb table as possible. 
- it will then check how many different images it received
- if the amount of images was higher than 1 then it would trigger a new lambda function N times (where N is the amount of entities received)
- after triggering the new lambda functions this lambda will ALWAYS successfully exit

With this solution we can always be assured that we trigger as many requests that we can in the shortest amount of time possible.  
Also we can be sure that the new patch of requests comes in each and every time.

### Conclusion

For us it seems that using things like lambdas and event based request triggering is the best possible solution for building something like webhooks. We are really - really pleased to see how the current stack behaves and how easy it is to actually implement the said feature. 

Really dont know what to write here though...

[lambda]: http://docs.aws.amazon.com/lambda/latest/dg/welcome.html
[dynamodb]: https://aws.amazon.com/documentation/dynamodb/
[testlio]: https://testlio.com
[apigateway]: https://aws.amazon.com/api-gateway
[wikipedia]: https://en.wikipedia.org/wiki/Webhook
[kms]: https://aws.amazon.com/kms/
