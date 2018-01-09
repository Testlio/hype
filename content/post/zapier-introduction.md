+++
author = "rasmus"
comments = false
date = "2018-01-09T13:00:00+03:00"
title = "Delegating engineering work to non-engineers with Zapier"
image = "images/post/zapier-introduction/cover.jpg"

share = true
slug = "delegating-engineering-with-zapier"
tags = ["zapier", "api", "integrations"]
+++

The tech industry is moving faster than ever, providing developers and product managers with a challenge - build new features quicker and validate changes with users more frequently. These kinds of iterations could be small MVPs, however, these would often require more valuable development time than we would like to spend. It is even more true in cases where we don't have enough data to be confident about the success of the new feature.

<!--more-->

This is usually a situation where you might think of a hack(ish solution) to prove the point and either make it pretty later or discard it if the idea doesn't pan out. Nobody likes hacks, as ever so often we move on and forget about them and eventually our codebase looks like *s#&t*.

So how would you approach the situation where a new feature demands more work than you can or want to do? In our real-world situation we delegated most of that work away from the engineering team to other team members without technical know-how. Expectedly they did not start writing code, but rather (with modest help from the engineering department) used great tools to build and manage a great chunk of that feature themselves. In this blog post I’m making a brief introduction to **[Zapier](https://zapier.com)** which was one of the tools that helped us do exactly that.

##### What is Zapier? ⚡️

Zapier is a platform that is meant for connecting your applications and automating workflows. Zapier moves information between web apps using integrations which they call “Zaps”. The sound of the word “integration” scares off many engineers, not to mention non-engineers, but Zapier has managed to make them so simple that even not technical users will manage to set them up.

There are many alternatives to Zapier out there, probably the best known being [IFTTT](https://ifttt.com/), which is a more end-user friendly service. Others like [Workato](https://www.workato.com/), [Integromat](https://www.integromat.com) and [Automate.io](https://automate.io) offer a very similar platform to Zapier, but with a somewhat different app directory. We definitely encourage you to check out the alternatives, but Zapier has so far met all our expectations which is why we have kept using it for a couple of years already.

##### How could I use it?

Setting up a Zapier Zap starts with choosing the **trigger** app. This is an application which will call your Zap and make it execute. Trigger apps can be browsed in [Zapier app directory](https://zapier.com/apps). There are loads of well-known apps already in the directory like [Gmail](https://zapier.com/apps/gmail), [Slack](https://zapier.com/apps/slack), [Github](https://zapier.com/apps/github), [Mailchimp](https://zapier.com/apps/mailchimp) or [Trello](https://zapier.com/apps/trello). In our case the trigger was [Google Forms](https://zapier.com/apps/google-forms). Alternatively you might create your own app which would integrate directly with your system (which is discussed in more detail later in this article).

The second part of the Zap requires choosing the **action** - an app (that can similarly be chosen from the app directory or be built by yourself) which will be called by the Zap. In our case the action was our own private app. Luckily we had an API which supported adding a new endpoint and building Zapier private app on top of that. If you don’t have the API ready then building one might be too time-consuming for a simple task.

{{< figure src="/images/post/zapier-introduction/zap-0.png" caption="Setting up Zap's trigger and action" >}}

Connecting the Zap trigger with action basically means linking the required data fields. Zapier has a nice UI for doing that so there’s really no technical knowledge required. You can add more than one action to the Zap chain or use any Zapier apps in the middle of the chain to process and manipulate the data.

##### Building your own app

Once you've chosen to build your own Zapier app to be acting as a trigger or action then you might be wondering how to set it up. Creating an app can be done with the Web Builder (which is the simpler way of doing it, but considered legacy for now) or the recommended CLI (Command Line Interface). The Web Builder requires specifying the app name, description and audience (private, public, etc). Creating a CLI app is slightly more demanding in terms of technical expertise, but the [CLI Github page](https://github.com/zapier/zapier-platform-cli) has all the necessary documentation for that. If you're in doubt about which interface to choose, then check out [an article](https://zapier.com/developer/documentation/v2/cli-vs-web-builder/) which discusses exactly that. In this post I'm using examples and screenshots from an app that is built with the Web Builder.

Having created an app, you should first specify the authentication details. Check the Zapier [authentication spec for app developers](https://zapier.com/developer/documentation/v2/authentication-fields/) for a thorough overview on the authentication topic. We have set up our authentication using an API key field which is injected to the authentication mapping.

{{< figure src="/images/post/zapier-introduction/zap-3.png" caption="Our authentication relies on the API key" >}}

In the authentication settings you can specify the Auth Type (Basic Auth, OAuth, API Key, etc), Mapping and Connection Label. Auth Mapping lets you map Basic Auth, Digest Auth, URL params and headers. You can use {{variables}} from the users auth input keys. Connection Label helps to identify the user account being used for the particular integration. Depending on the authentication type you might need to specify other parameters, e.g. Client ID, Client Secret, Auth URL and Access Token URL for OAuth.

{{< figure src="/images/post/zapier-introduction/zap-4.png" caption="Setting up Basic Auth" >}}

Secondly, you should add some [triggers](https://zapier.com/developer/documentation/v2/triggers/) and/or [actions](https://zapier.com/developer/documentation/v2/actions/) to your Zapier application. Triggers bring your application data into Zapier and actions send data from Zapier to your application.

You can trigger a Zap using webhooks or polling. Zapier supports different types of webhooks like Static, REST and REST Notification hooks. Depending on the choice you'll need to configure URL-s for polling, subscribing to and unsubscribing from webhooks. You also need a Test Trigger to verify user's auth credentials.

Creating a new trigger allows you to specify the expected trigger fields and data source (hooks or polling). In addition you will provide a sample result which will only be used in the Zap Editor. You should provide a hard-coded fallback JSON object for a single result which will be used as sample data if your API returns no live results.

{{< figure src="/images/post/zapier-introduction/zap-6.png" caption="Setting up a trigger" >}}

Adding an action is arguably a bit easier, but in many ways similar to creating a trigger. You can think of Zap actions as POSTs, writes, or the creation of a resource. It involves Zapier sending data to your app. You have to specify the required action fields, provide the action endpoint URL and a sample result.

{{< figure src="/images/post/zapier-introduction/zap-7.png" caption="Setting up an action" >}}

In addition to the web interface you can use the [Scripting API](https://zapier.com/developer/documentation/v2/scripting) for gaining extra flexibility. Zapier's Web Builder scripting functionality allows you to manipulate the requests and responses that are exchanged between your app's API and Zapier. You can modify HTTP requests just before they are sent and parse responses before Zapier does anything with them.

{{< figure src="/images/post/zapier-introduction/zap-5.png" caption="Our app triggers and actions" >}}

##### Zaps in practice

Our Zap asked for new Google Form responses in a synchronised manner and sent the results directly to our platform. This was achieved without building our own scheduled fetching mechanism that would need to integrate with Google Forms API. We only built an API endpoint and a Zapier app which took much less time and effort.

Zapier offers great visibility of what is going on inside the Zaps. Each piece of data you run through your Zap is called a task and Zapier keeps a history of tasks with full data log that helps you debug the integrations.

{{< figure src="/images/post/zapier-introduction/zap-1.png" caption="Our Zap task history" >}}

There are also cool built-in Zapier apps like [Code by Zapier](https://zapier.com/apps/code/integrations) which allows you to run JavaScript or Python inside the Zap. It’s also worth checking out Zapier’s [Email](https://zapier.com/apps/email/integrations), [Webhooks](https://zapier.com/apps/webhook/integrations) and [SMS](https://zapier.com/apps/sms/integrations) built-in apps. In our case we tuned up the Zap with some JavaScript code that took care of parsing out pure results. The Code by Zapier app did some calculations and provided “pure” data which corresponded to our platform data model.

```js
output = {
    email: inputData.email,
    score: score,
    testID: inputData.testID,
    tag: userTag
}
```
_Code by Zapier expects you to provide an output object that defines params which could be used by other actions in the Zap chain._

{{< figure src="/images/post/zapier-introduction/zap-2.png" caption="Code output usage in the next action" >}}

Separating the integration related code helped us maintain a clean codebase and made it very easy to debug and modify the integration in one single place. In addition the API endpoint became generic enough to support new Zaps without adding any code to our main platform application. This effectively meant that the non-technical part of our team could take full ownership of these integrations and add new Zaps on their own without spending any engineering resources. They also take care of monitoring the integrations using the Zapier task history, leaving the engineering department to deal with more urgent matters.

##### Everything is awesome? 🎉

As you might imagine, Zapier worked really well for us. We needed a quick way to build up an integration with an external application and delegate some work away from the engineering team, but that doesn’t mean it’s a perfect fit for you.

Zapier tends to focus on simplicity rather than quantity. It’s possible to upgrade your plan to support unlimited zaps and 75k+ task runs, but all of this could become unreasonably expensive (see their [pricing page](https://zapier.com/pricing/)). In a world of scalability and ever-growing amounts of information this could quickly become a blocker.

Secondly, you might not have the required non-technical resource and initiative available, who could accept the ownership of Zaps and integrations. In that case I’d still say that building with Zapier could simplify and speed up your development, but it really depends on the task at hand. It is very likely that Zapier apps won't provide you the expected flexibility or features and building your own Zapier app or an API to support it could be a massive overhead.

Zapier Co-founder and CEO Wade Foster suggests in [an interview](https://www.inc.com/sujan-patel/building-zapier-through-action-not-perfection.html) to build (and fail) fast, default to action and worry less about perfection. Our takeaway would be similar - it’s often wise to launch quickly and not spend too much time on making things perfect. And Zapier is a great tool that helps you achieve exactly that. In addition it is simple enough to delegate some of the burden of building new features to your non-technical part of the team, saving valuable engineering resource and speeding things up even more.

_Also check out [Zapier help](https://zapier.com/help/) and [developer documentation](https://zapier.com/developer/documentation/v2/) for more details about how it all works._ 🤓

