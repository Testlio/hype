+++
date = "2016-04-16T09:10:00+03:00"
draft = true
title = "Lambda Backed Microservices (Part 2)"
slug = "lambda-backed-microservices-part-2"
author = "henri"

image = "images/post/lambda-tools-introduction/cover.png"
share = false
comments = false
tags = ["lambdahype"]
+++

_This is the second part in our series of posts about AWS Lambda backed microservices. Make sure to check out [Part 1]({{< ref "lambda-getting-started.md" >}})._

In the first post, we introduced the building blocks for our services. We also proposed an architecture for our services. The architecture would keep the different layers separated, allowing us to treat each layer individually and offer abstractions tailored specifically for each.

In this part, we will introduce the tools and libraries that we believe help us enforce the proposed architecture. These are [Lambda Tools](https://github.com/testlio/lambda-tools) and [Generator Lambda Tools](https://github.com/testlio/generator-lambda-tools).

## Lambda Tools

The first toolset we will look at is [Lambda Tools](https://github.com/testlio/lambda-tools). The aim of these tools is to manage two phases of a service's lifecycle - running/testing it locally, and deploying to the AWS cloud. The library itself consists of 5 command-line scripts, written in Node.js that allow running Lambda backed services locally (with some exceptions) and deploying said services.

Both of these features solve concrete problems in our workflow. Let's talk in detail about both. It is worth noting that Lambda Tools assumes a specific structure to a project, which is documented in its GitHub repository, and as such will not be discussed in this post.

### Local Execution

Due to its' nature, running Lambda backed services locally requires more work than simply starting a web-server. As we want our API Gateway definition to be loosely coupled to our Lambda functions, there is quite a bit that needs to be done in order to simulate the API locally. Furthermore, API Gateway allows rather complex mapping logic, which also needs to be simulated locally. All of this logic is condensed into the `lambda run` command.

This command takes the following steps:

1. Searches the service for an API definition, usually at `./api.json`
2. Goes over the API definition, building a set of routes that it can serve, including all of the API Gateway mapping logic (request -> Lambda -> response)
3. Sets up a web server to serve said routes, defaulting to port 3000
4. Watches for changes to `api.json`, live-reloading it if there are any changes

When any of the endpoints is triggered, the corresponding Lambda function is executed. The code for the Lambda function is not cached, which both mimics the worst case of API Gateway, as well as serves as a means to live-reload Lambdas as they change.

This allows for a work-flow that looks roughly like this:

1. Start the service - `lambda run`
2. Execute an endpoint via `curl` or similar
3. Modify Lambda code or API definition and repeat step 2

This covers running Lambda functions that are exposed via the API, however, there are often also Lambda functions that are tied to events on AWS services and thus have no endpoint for us to trigger locally. For these functions, Lambda Tools has the `lambda execute` command.

This command can be used to execute a single Lambda function, with a specific `event` loaded from a file. The execute command can also be beneficial when working with Lambda functions that are not part of any microservice.

For example, given a service with the following structure:

```sh
.
├── README.md
├── api.json
├── cf.json
├── event.json
├── lambdas
│   ├── bar
│   │   ├── index.js
│   │   └── event.json
│   └── foo
│       └── index.js
└── package.json
```

We can run `lambda execute bar` from the root of our service to run the `bar` Lambda function. The script is smart enough to look for a `event.json` file next to the Lambda function, and load it as the event if one exists. Alternatively, we can also load an event from elsewhere by using the `-e` option:

```sh
lambda execute bar -e ./event.json
```

This will load the event file that is located at the root of the service. This approach allows us to reuse events from a central location, as events more often than not are similar.

In short, the combination of `lambda run` and `lambda execute` allows us to quickly try out changes to our functions or the service setup locally. Granted, neither of these running environments is identical to that of the AWS cloud, however, they allow for a quick prototype to be tried out.

It is worth noting that neither of these commands does anything with the CloudFormation stack. This means none of the resources that the code needs are created or run locally. This decision is a practical one, as Lambda Tools can not possibly know ahead of time all of the resources the services may want to use.

### Deployment

Once we are satisfied with our service performs locally, we can deploy it to the cloud by running the `lambda deploy` command.

This will kick-start the deployment process, taking several steps, which completed result in the service being deployed as a CloudFormation stack. For the purposes of this post, the description of each of the stages is kept to a minimum, you can read more about the steps in [GitHub](https://github.com/testlio/lambda-tools#deploy).

1. Sets up a local staging area, used as a cache for subsequent deployments
2. Sets up a remote S3 bucket, which is used as a storage for the CloudFormation stack and its assets
3. Processes all Lambda functions, minifying and transpiling them, embedding requested environment variables
4. Uploads all of the Lambda code, the API definition and the CloudFormation template to S3
5. Trigger CloudFormation, wait for it to successfully complete

Generally speaking, for a relatively small service (5-6 Lambda functions) the deployment process takes about 1-2 minutes. However, the deployment time can vary significantly depending on what resources the stack uses or how.

Also, deployments without the local cache can also take longer due to the expensive transpiling process that is run on the Lambda functions.As a reminder, this process is needed as the Node.js runtime used by AWS Lambda is an older one that may not support all of the features we are used to.

The deployment script also allows deploying to a different stage _(default being `dev`)_, or a different availability-zone on AWS _(default being `us-east-1`)_. This means we can quickly redeploy our service in case of region-wide outages or to a different stage if we want to try out a new feature. Lambda Tools does not currently offer any kind of _unpublishing_ feature, if you believe this is important, please let us know via [an issue](https://github.com/testlio/lambda-tools/issues).

The deployment script can also be used directly from a CI (assuming appropriate IAM permissions) by simply including a line similar to this one.

```
lambda deploy -s production -e NODE_ENV=production
```

Notice that the deployment script also allows defining environment variables to be embedded into the Lambda functions. As Lambda has no way of defining environment variables natively, Lambda Tools uses [envify](https://github.com/hughsk/envify) to replace the specified environment variables with corresponding values during the processing step.

## Generator Lambda Tools

While Lambda Tools allows managing the life-cycle of a service, it makes a few critical assumptions:

1. The service structure on disk is fixed
2. The API definition is a JSON Swagger file with [API Gateway extensions](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions.html)
3. The CloudFormation template is valid and includes all required resources

Making sure these assumptions hold, and do so without forcing engineers to learn all of the nitty-gritty details of the different JSON formats, is the job of [Generator Lambda Tools](https://github.com/testlio/generator-lambda-tools). These are a set of [Yeoman](https://yeoman.io) generators, that help bootstrap new services or modify existing ones by adding new resources/Lambda functions.

With the help of the generators, setting up a new service is as easy as running `yo lambda-tools` and answering a few questions.

```
yo lambda-tools
? Service name example-service
? Service description Demonstrating the generators
? License (API) ISC
? Author email dev@testlio.com
? Author name Testlio, Inc.
? Install dependencies lambda-tools, lambda-foundation
   create api.json
   create cf.json
```

We can then add endpoints and various other resources to our service by running some of the other generators. For example, we can simply run `yo lambda-tools:endpoint` to add a new HTTP endpoint backed by a Lambda function. The full list of generators can be seen on [GitHub](https://github.com/Testlio/generator-lambda-tools#generators).

This means for most of the common scenarios, we never need to manually edit the `api.json` or `cf.json` files, instead running generators that do that for us. This reduces errors in the configuration files, while also allowing us to focus more on the contents of the Lambda functions, rather than their setup.

In short, by using the generators, Lambda Tools can safely make assumptions about the accuracy and structure of our service scaffolding. Furthermore, the generators can be tested and improved over time, introducing new features without having to re-teach engineers about the underlying specifics. This is a critical abstraction that makes working with these types of services easier and faster.

The generators also leave the option and control over the resulting files to us, which is important for removing any notion of _magic_ from the service setup. This is critical as it means the engineers working on the service are in control, even if the code is generated for them. It also means that for those few cases where generators are not good enough, we can dive directly into the configuration files and make the changes ourselves.

Such a workflow should result in more and more generators being written over time, as they certain new processes become common enough. While other generators can be updated to accommodate any changes AWS may make, or even removed as they become unnecessary.

## Conclusion

In this part, we looked at the tools we can use to manage our services. [Lambda Tools](https://github.com/testlio/lambda-tools) helps us with running and deploying the service, giving us the convenience of not having to interact with CloudFormation or API Gateway directly, instead allowing us to focus on writing the Lambda functions. The toolchain also enforces the service architecture we devised in [Part 1]({{< ref "lambda-getting-started.md" >}}).

The second component we introduced is [Generator Lambda Tools](https://github.com/testlio/generator-lambda-tools), a set of Yeoman generators that add an additional layer of abstraction over the `api.json` and `cf.json` files. The generators allow us to quickly bootstrap services and resources, without having to know the custom JSON formats ourselves. This let's us focus even more on the Lambda functions, and generally not worry about the nitty-gritty of service configuration.

In the next part, we will dive deeper into how we can make our Lambda functions more consistent, while also making them more [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).
