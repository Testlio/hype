+++
author = "henri"
comments = false
date = "2016-07-01T11:00:00+03:00"
title = "Taming logs with CloudWatch"
image = "images/post/cloudwatch-logs/cover.png"

share = true
slug = "taming-logs-with-cloudwatch"
tags = ["lambdahype", "awshype"]
+++

One of the requirements highlighted by the [Serverless Manifesto](http://serverlessmanifesto.com) is that _Metrics and logging are a universal right_. As we move towards having more and more serverless microservices, enforcing and supporting said universal right becomes increasingly more important.

<!--more-->

Luckily for us, AWS Lambda already does a lot towards enabling this, by sending all of its logs __automatically__ to [CloudWatch](https://aws.amazon.com/cloudwatch/). In this post, we'll look at how to interact with the logs and what kind of improvements we could make to our approach.

**Note about assumptions**

Throughout this post, we'll interact with CloudWatch under the assumption that it has some logs from any number of Lambda functions. If you don't have any Lambda functions set up, you can use one of the blueprints and simply add some logging statements to it, or you can refer to our example service we created in one of [our previous posts]({{< ref "lambda-service-example.md" >}}).

## Logs in AWS Console

One of the easiest ways of browsing the produced log output is via the [AWS Management Console](https://console.aws.amazon.com/cloudwatch/home#logs:). Here we are greeted by a list of __log groups__, which we can do some basic filtering on. In general every Lambda function will map to a single log group, where the group name follows the form `/aws/lambda/<lambda-function-name>`.

{{< figure src="/images/post/cloudwatch-logs/list-log-groups.png" caption="All log groups that correspond to our example Lambda service endpoints" >}}

If we open one of these groups, we see the __streams__ that are in this group. For Lambda functions, each of these streams represents a lifetime of a specific function instance. It is important to mention, looking at these streams, we get a glimpse of how AWS Lambda operates, how a single instance is sometimes kept around for more than one invocation.

{{< figure src="/images/post/cloudwatch-logs/list-log-streams.png" caption="All log streams corresponding to one of our demo service endpoints" >}}

Side note, based on the usage data that we have gathered at Testlio, there doesn't seem to be a straightforward cut-off point in terms of how long a specific instance is kept alive. In general, the likelihood of a single function instance being invoked multiple times seems to go up as we increase the memory size.

When we open up a specific stream, we can see the log events in chronological order. Among our own log lines, Lambda also includes three types of events, namely `START`, `END` and `REPORT`.These can be quite useful for long-term monitoring, as for example, filtering out only the `REPORT` events allows us to gather information about the execution time and memory usage of our Lambda function.

{{< figure src="/images/post/cloudwatch-logs/specific-log-stream.png" caption="Filtered REPORT events in a specific log stream" >}}

Using the AWS console is fine if we don't have much traffic or when we are not looking for anything particular. However, in production environments, it is often cumbersome and time-consuming to use the AWS console to narrow down logs to a specific event in the system, for that, there are tools we will look at next.

## Using CLI tools for logs

While there are various different CLI tools available for CloudWatch, in this post, we will focus on a really nice one by Jorge Bastida - [awslogs](https://github.com/jorgebastida/awslogs).

Installing this Python script via [pip](https://pypi.python.org/pypi/pip) is a breeze, just run `pip install awslogs --ignore-installed six`. Once it has finished installing, we have access to the main `awslogs` command. For comparison, the steps we took in the previous section can be reduced to three commands with `awslogs`.

First, listing/filtering log groups can be done by calling `awslogs groups`. This will output all available groups by their name, meaning we can filter over it with other commands, such as `grep`.

```shell
$ awslogs groups | grep /aws/lambda/demo
/aws/lambda/demo-service-dev-AnswersWebhook-H0LNVD2BPINR
/aws/lambda/demo-service-dev-QuestionsAnswersPost-1GFL16UO67BH7
/aws/lambda/demo-service-dev-QuestionsRandomGet-1QGCGRU4MLUDW
```

Listing all streams in a group can be done via `awslogs streams LOG_GROUP_NAME`. Amongst other options, we can also specify the start and end dates for filtering via the `--start` and `--end` options. For example, filtering our log streams to only those that saw activity in the last week.

```shell
$ awslogs streams /aws/lambda/demo-service-dev-QuestionsRandomGet-1QGCGRU4MLUDW --start="1w ago"
2016/06/29/[$LATEST]9a97da1c0585445f8fc95f4ce9727656
```

Finally, we can get the contents of a specific stream by using the `awslogs get` command. Similarly to the previous command (and what can be done in the console), we can also filter this one.

```shell
$ awslogs get --start="1w ago" --filter-pattern="REPORT" /aws/lambda/demo-service-dev-QuestionsRandomGet-1QGCGRU4MLUDW 2016/06/29/[\$LATEST]9a97da1c0585445f8fc95f4ce9727656
/aws/lambda/demo-service-dev-QuestionsRandomGet-1QGCGRU4MLUDW 2016/06/29/[$LATEST]9a97da1c0585445f8fc95f4ce9727656 REPORT RequestId: 2ca6ae88-3dd8-11e6-8c59-cb86947357c0	Duration: 355.05 ms	Billed Duration: 400 ms 	Memory Size: 256 MB	Max Memory Used: 36 MB
```

In addition to achieving feature parity with the console, we can do other cool things, like for example browse logs across many different streams, filtering them with regular expressions. As an example, here are some logs from one of our production services (slightly cropped to show that log lines are coming from different streams)

```shell
$ awslogs get /aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS ALL -s='1d ago' --filter-pattern='Searching'

/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:50:44.155Z	f277f2e4-3dde-11e6-a7ba-1922d5da6516	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'Medias') (term field=model 'Medias') (prefix field=model 'Medias') (term field=manufacturer 'Medias') (prefix field=manufacturer 'Medias')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:50:45.145Z	f30fc684-3dde-11e6-b504-53e504ae2ab1	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'Medias 0') (term field=model 'Medias') (prefix field=model 'Medias') (term field=manufacturer 'Medias') (prefix field=manufacturer 'Medias') (term field=model '0') (prefix field=model '0') (term field=manufacturer '0') (prefix field=manufacturer '0')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:50:45.681Z	f362534a-3dde-11e6-9be1-5b1bab4879f8	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'Medias 07') (term field=model 'Medias') (prefix field=model 'Medias') (term field=manufacturer 'Medias') (prefix field=manufacturer 'Medias') (term field=model '07') (prefix field=model '07') (term field=manufacturer '07') (prefix field=manufacturer '07')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:51:04.971Z	fee0fb0a-3dde-11e6-b49b-478da82b433e	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'Medias') (term field=model 'Medias') (prefix field=model 'Medias') (term field=manufacturer 'Medias') (prefix field=manufacturer 'Medias')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:51:07.263Z	003eb659-3ddf-11e6-b757-1fe994a34b1a	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'Medias N-07D') (term field=model 'Medias') (prefix field=model 'Medias') (term field=manufacturer 'Medias') (prefix field=manufacturer 'Medias') (term field=model 'N-07D') (prefix field=model 'N-07D') (term field=manufacturer 'N-07D') (prefix field=manufacturer 'N-07D')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]8ae2b4c33211495f9c0ab2a052b653da 2016-06-29T09:51:32.296Z	0f2a994d-3ddf-11e6-9adb-c5a94ee64654	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'medias') (term field=model 'medias') (prefix field=model 'medias') (term field=manufacturer 'medias') (prefix field=manufacturer 'medias')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:16:52.636Z	fb2c13a1-3dea-11e6-b76a-5358827c090c	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nex') (term field=model 'nex') (prefix field=model 'nex') (term field=manufacturer 'nex') (prefix field=manufacturer 'nex')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:16:53.071Z	fb6ee82c-3dea-11e6-839b-abc46b8ef1b8	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nexu') (term field=model 'nexu') (prefix field=model 'nexu') (term field=manufacturer 'nexu') (prefix field=manufacturer 'nexu')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:16:53.750Z	fbd63623-3dea-11e6-813f-eb0dbee13977	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nexus') (term field=model 'nexus') (prefix field=model 'nexus') (term field=manufacturer 'nexus') (prefix field=manufacturer 'nexus')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:17:13.785Z	07c7c43d-3deb-11e6-9f17-8d478e5df2c6	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nex') (term field=model 'nex') (prefix field=model 'nex') (term field=manufacturer 'nex') (prefix field=manufacturer 'nex')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:17:14.357Z	081e6ffb-3deb-11e6-8cfe-0d8d31e304a2	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nexu') (term field=model 'nexu') (prefix field=model 'nexu') (term field=manufacturer 'nexu') (prefix field=manufacturer 'nexu')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:17:15.978Z	0913a4b3-3deb-11e6-b10b-9b2b2f205bb8	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'nexus 7') (term field=model 'nexus') (prefix field=model 'nexus') (term field=manufacturer 'nexus') (prefix field=manufacturer 'nexus') (term field=model '7') (prefix field=model '7') (term field=manufacturer '7') (prefix field=manufacturer '7')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/29/[3]1dfecaa027ff44debcdcb9ac2c6ef23a 2016-06-29T11:17:41.376Z	1838bfee-3deb-11e6-a8be-1dd128e227e5	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'wi') (term field=model 'wi') (prefix field=model 'wi') (term field=manufacturer 'wi') (prefix field=manufacturer 'wi')))"
```

This kind of search capabilities allow for easier debugging, for example, if we know the request ID of a specific request, we can filter to only search for that.

```shell
$ awslogs get /aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS ALL -s='1d ago' --filter-pattern='"141d2c66-3ea9-11e6-b716-2f2fe2863b8d"'

/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 START RequestId: 141d2c66-3ea9-11e6-b716-2f2fe2863b8d Version: 3
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:38.924Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Unpacked query { q: 'apple' }
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:38.925Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Searching in domain "devices-prod-devices" with query "(and (or (term field=identifier 'apple') (term field=model 'apple') (prefix field=model 'apple') (term field=manufacturer 'apple') (prefix field=manufacturer 'apple')))"
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:38.960Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Found 10 devices with guids:  [ '90082236-dec9-40f9-ad18-14097202ecbe',
  '24b34e19-f05d-4017-b80c-1add5f02c58a',
  ...
  '472a62fb-6afe-44e5-a6fc-a1acc17d320b' ]
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:39.088Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Resolved to 10 devices from DB
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:39.106Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Fetching all 91 OSes [ 'fe66a2c5-0f80-4522-9bd7-7f424e2b4ae3',
  'ec79e9af-f319-4cff-8205-d56538fedc97',
  ...
  'abdc5cd3-afb7-4134-bef0-7cbd9f0f79bd' ]
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 2016-06-30T09:57:39.285Z	141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Found 91 OSes from DB
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 END RequestId: 141d2c66-3ea9-11e6-b716-2f2fe2863b8d
/aws/lambda/device-service-prod-DeviceFilteredGet-1VXYLZSBFBLGS 2016/06/30/[3]460025f44fc84f66bee35dce7e3e8098 REPORT RequestId: 141d2c66-3ea9-11e6-b716-2f2fe2863b8d	Duration: 541.88 ms	Billed Duration: 600 ms 	Memory Size: 256 MB	Max Memory Used: 44 MB
```

This means, that when combined with services such as [Raygun](https://raygun.com), getting logs for specific requests can be quite simple. Armed with an endpoint, which we can map to a single Lambda function, and a request ID, we can find all of related logs quite quickly.

On the other hand, we can also leverage the `--watch` option to keep a specific log group open indefinitely. This can be useful for constantly monitoring a specific resource in our system. This becomes even more useful when combined with `grep`, as that way we can also filter out specific events. In fact, this is really similar to what CloudWatch already offers as part of their Logs Metric Filter offering. This allows us to count the number of occurrences of some filter expression in the log group.

Doing something similar locally with `awslogs` is trivial. For example, if we wanted to keep an eye on all requests that produce a 500 error on API Gateway, then we could watch a log group and apply our filter via `--filter-pattern` to it. Note, this example relies on API Gateway's ability to [push access logs to CloudWatch](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-stage-settings.html#how-to-stage-settings-console)

```shell
$ awslogs get API-Gateway-Execution-Logs_cqy6c35ye6/prod ALL -s="1d ago" --watch --filter-pattern='"Method completed with status: 500"'
```

If a new incoming request ends up producing a 500 on our API in API Gateway, then we would immediately get a line in our log that looks something like this.

```shell
API-Gateway-Execution-Logs_cqy6c35ye6/prod 0266e33d3f546cb5436a10798e657d97 Method completed with status: 500
```

Unfortunately from this we can't quite narrow down our hunt to a single request, but we will be able to do so, by using `grep` with the particular API Gateway instance ID. Somewhere in the resulting output will be a sequence of log lines that will also include the request ID.

```shell
$ awslogs get API-Gateway-Execution-Logs_cqy6c35ye6/prod ALL -s="1d ago" | grep "0266e33d3f546cb5436a10798e657d97"
API-Gateway-Execution-Logs_cqy6c35ye6/prod 0336dcbab05b9d5ad24f4333c7658a0e Starting execution for request: 691c044c-3e9d-11e6-b7f6-a1b4808b785c
API-Gateway-Execution-Logs_cqy6c35ye6/prod 0336dcbab05b9d5ad24f4333c7658a0e HTTP Method: GET, Resource Path: /devices
...
API-Gateway-Execution-Logs_cqy6c35ye6/prod 0336dcbab05b9d5ad24f4333c7658a0e Method completed with status: 500
```

Now, armed with the request ID, endpoint and HTTP method, we can look into the logs on the specific Lambda function as we did before.

## Potential workflows

Are there workflows we can build on top of CloudWatch? For starters, we can make sure that our error tracking system (such as Raygun), always includes the request ID in the error that is stores.

This request ID can then be used to trace logs across multiple levels, including API Gateway, as well as the Lambda function. This is quite trivial to achieve, as Lambda exposes this ID on the `context` object, as `context.awsRequestId`.

Building on top of that, we could also start automating these processes, by using webhooks from Raygun to capture an error, which can then be supplemented with logs from various streams. The results could then be stored somewhere or simply posted to Slack or sent to an email. As always, this could be built as a simple Lambda backed service.

As an alternative, we could also build something on top of CloudWatch, that would help us analyse our log files across our entire stack. Services such as [Logstash](https://www.elastic.co/products/logstash) can be utilised to collect and analyse log files more in-depth than CloudWatch allows.

In addition, enforcing a common structure in our logs, for example, making all log output be in JSON, allows us to use advanced features in CloudWatch to not only filter by strings, but also by values and key paths in the JSON. These extracted values could then be used to create graphs and metrics, which could allow us to quickly identify problems by having patterns emerge faster.

## Conclusion

The serverless programming model puts a lot of focus on the ability to monitor the execution of our functions. Logging is a crucial part of this monitoring, not only for debugging purposes, but also for understanding usage patterns, prioritising upcoming features etc. With AWS Lambda, we get high-level logging _for free_ via CloudWatch, allowing us to capture logs in a centralised place that can then be filtered and explored.

Furthermore, as CloudWatch also includes usage metrics from AWS resources, we can start combining these with logs to form a detailed look at how our services are operating. Allowing us to not only offer a better product to our customers, but also generate as many fancy graphs and metrics as we want.
