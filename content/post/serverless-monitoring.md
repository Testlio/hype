+++
author = "mario"
comments = false
date = "2018-01-22T10:00:00+03:00"
title = "Monitoring and debugging AWS Lambda based microservices using Dashbird"
image = "images/post/serverless-monitoring/cover.jpg"

share = true
slug = "serverless-monitoring"
tags = ["dashbird", "lambdahype", "serverless"]
+++

AWS Lambda backed microservices have caught on like wildfire ever since Amazon introduced the first version back in 2014. Dealing with an architecture where your back end logic is distributed over a large amount of functions poses a real problem when a function doesn't behave as expected. To provide insights into Function as a Service architectures, **[Dashbird](https://www.dashbird.io/)** collects all relevant CloudWatch logs and extracts meaningful and actionable metrics which help greatly to bring your service quality up a notch.

<!--more-->

##### The problem

Functions in a FaaS architecture behave inherently differently than servers. Since the release of the Serverless framework the question has been "how to measure application performance and track errors?". Looking back, this has been one of the biggest problems to solve. In Serverless, there are three critical metrics: invocation level performance, execution errors, and account level metrics. If you've ever built (or tried to build) a Serverless system, you've probably come across at least one of these problems:

1. **Snow blindness.** Sometimes code behaves differently from the way you expect. For instance, endpoints can perform slower than you think and if you don’t measure it, you will never know. Worse yet, your users probably will. You also might be close to timeouts or memory limits, without even knowing about it.

2. **Silent failures.** It’s good to catch errors and report them to a service that yells at you on Slack or e-mail. However, with Lambda functions, this does not work in all cases. Timeouts, configuration errors and early exits are not reported to error tracking services.

3. **No helicopter view.** You know that your AWS account has a limit for concurrent executions but it’s harder to know how many are going off at the moment and if you’re in the risk of running into this limit. Also it’s difficult to know which functions trigger (and cost) you the most.

All those are very hard to address by just attaching monitoring and reporting tools to your application code. On top of that, it’s an operational, performance and development overhead to attach it to every function and if you don’t, you’ll leave blind spots. Scaling your process without addressing those issues is impossible.

##### What is Dashbird?

Dashbird is a tailor-made service that works by parsing the pre-formatted CloudWatch logs that all Lambda functions emit with each invocation. Each piece of data in those logs is formatted into separate Lambda invocation events and is also compiled into a generic bird's eye view dashboard. Unlike alternatives such as [IOpipe](https://www.iopipe.com/) or [Thundra](https://www.thundra.io/), you don't need to attach a module to your monitored function, which would cause a delay in its execution, nor explicitly annotate monitored resources.

{{< figure src="/images/post/serverless-monitoring/dashbird-1.png" caption="Dashbird dashboard" >}}

Everything relevant is captured in the dashboard view. Even without further alerting, this prevents problems from going unnoticed and gives the developer a good overview of the overall health of his functions and their usage. The main dashboard consists of time-series metrics for invocation counts, invocation durations, memory usage vs the allocated maximum, health statistic and error reports.

Now, instead of diving into the countless log streams in CloudWatch, you have everything you want to know about your service performance just a few clicks away. All that without any code changes whatsoever.

##### Dashbird in practice

Dashbird adds no development overhead, meaning it does not affect the performance of your Lambda functions. All the data used by the service is fetched from CloudWatch after the Lambda invocation has ended. It takes about 1-2 minutes for new events to appear in the dashboard or, if you're in a hurry, about 20 seconds to appear on your screen using the live tailing feature.

{{< figure src="/images/post/serverless-monitoring/dashbird-2.png" caption="Live tailing Lambda invocations" >}}

Dashbird allows you to group Lambda functions any way you like using their names. This enables you to construct custom dashboards for service level monitoring, outlining its load, errors and other important metrics. This is especially useful if your AWS account includes Lambdas for several independent services.

{{< figure src="/images/post/serverless-monitoring/dashbird-3.png" caption="Grouping Lambda functions into a service dashboard" >}}

The function view shows all the meaningful data for a specific Lambda. You can see a history of last invocations with a clear distinction between successful and failed executions. Since CloudWatch logs are generic regardless of invocation outcome, the error distinction within Dashbird is made by a native regular expression.

Also you get the same metrics as in the dashboard view, but this time only for this specific function. Normally you would only care about the invocation failures, but if your Lambda behaves differently from the way business intended and if it has an especially high usage, you can also use the search within the function view to find a specific invocation.

{{< figure src="/images/post/serverless-monitoring/dashbird-4.png" caption="Lambda function view" >}}

Everything so far has led us to the actual logs generated by the function execution. In the invocation view, you see everything you have simply logged out in the Lambda source code. Furthermore the metadata helps you understand what exactly happened during the time the function was executing. Having all individual invocations explicitly compiled in an easy to find view is something I can no longer even think about not having. Working with Lambda functions every day, I find it invaluable not to have to search through log streams for what is often only one line that is causing all the problems in my services.

{{< figure src="/images/post/serverless-monitoring/dashbird-5.png" caption="Invocation view" >}}

##### Alerts and reporting

The smart parser within the service allows for error tracking, meaning you can get further KPIs about your functions. Knowing exactly how often an error occurs is helpful in large scale difficult systems where the root cause is often unknown and refactoring or potential fixes are deployed daily.

As with other error tracking services (e.g [Raygun](https://raygun.com/)), you get a crash dashboard with a time-series graph with failure counts along with timestamps from the error lifespan.

{{< figure src="/images/post/serverless-monitoring/dashbird-6.png" caption="Error tracking" >}}

In todays fast-paced business environment with often very short service level agreement deadlines, it's crucial to get crash alerts as fast as possible. Dashbird offers notifications through email and an integration with Slack with a short description of the error and a direct link to the failed invocation view. Can you imagine having a Lambda based service without such alerts?

The service also provides daily report emails with key points of interest compiled from the invocation logs from the previous 24 hours. All this reporting functionality means that even if your service runs flawlessly, you can still count on Dashbird to alert you when something goes wrong. Knowing that all your functions' emission gases are being monitored 24/7 can give you real peace of mind when working with the volatile, short-lived, parallel and highly scalable nature Lambdas. The aforementioned approach has helped us bring clarity and visibility into our Serverless systems.

{{< figure src="/images/post/serverless-monitoring/dashbird-7.png" caption="Integration with Slack" >}}

##### Quick and painless set up

Setting up Dashbird takes less than 5 minutes. You are required to create a new AWS IAM role for Dashbird to operate. The policy document contains permissions to list your Lambda functions and read your CloudWatch logs.

```js
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "logs:FilterLogEvents",
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "logs:describeLogStreams",
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "lambda:listFunctions",
                "Resource": "*"
            }
        ]
    }
```
_Policy required by Dashbird to operate_

If you managed to set up the delegation correctly, it should take a few minutes for invocation events to appear in the dashboard view.

Dashbird adds almost nothing to your monthly AWS bill. The data transfer out of CloudWatch is priced equivalently to the data transfer from EC2. The first GB of the month is free, with every GB after that a mere $0.09. The service itself is [priced](https://dashbird.io/pricing/) by usage with a 14 day free trial and a basic free tier with 10 Lambda functions and 500k invocations per month.

The main benefit of Dashbird is simplicity. It's easy to set up, it's easy to use and most importantly it's easy to understand. I hate the visual noise within CloudWatch logs. I don't want to spend time searching for specific functions and then specific invocations within the thousands of lines of execution logs.

Tailored services like Dashbird cut a good chuck of time from Serverless development and maintenance by doing exactly what is needed with as little hassle as possible. Dashbird is currently my go-to service for Serverless monitoring. It remains to be seen how the product develops in 2018, but if the progress is as good as last year, it could set the new benchmark for Lambda monitoring.


_We at Testlio have used Dashbird to monitor our Lambda functions since its inception. If you're facing the same problems we did, check out the full list of [product features](https://dashbird.io/features/) to see if it's a good fit for you._
