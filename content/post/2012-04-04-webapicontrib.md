---
title: WebApiContrib
author: Pedro
type: post
date: 2012-04-04T18:35:29+00:00
aliases:
  - /2012/04/04/webapicontrib/

dsq_thread_id:
  - 636665780
categories:
  - WebAPIContrib

---
The  announcement of the ASP.NET Web API got a lot of people interested in playing with, bending, learning the framework to see what was possible to do with it. Some folks were doing that already back when it was WCFWebApi. But the move to the ASP.NET team and having it being part of the ASP.NET MVC 4 beta definitely brought a lot of attention to the Web Api framework.

### The WebApiContrib organization

With all that attention, and people playing with it, different contrib efforts were sparkling all around. There was the original [WcfWebApiContrib][1] maintained by [Darrel Miller][2]. We at [Headspring][3] created our own contrib. Folks from [Thinktecture][4] created there own. Some other authors were pushing their own extensions to their own repositories. Basically, it was a mess.

[Ryan Riley][5] then started the effort to bring everybody together to contribute to a single contrib project. He created [an organization][6] on github and [a mailing list][7]. After I got contacted by Ryan, I talked with the other Headspring folks and we decided to transfer our [WebApiContrib][8] repository from our [HeadspringLabs][9] organization to the WebApiContrib one.

This was the first move that we made, after that, Thinktecture extensions were moved and the handlers from the WcfWebApiContrib were also moved.

### How can I get it?

Right now, you can [head to the github repository][8] and pull the code. I know it’s kinda ghetto in these [Nuget][10] and <del>Open Rasta</del> [OpenWrap][11] days.

In order to fix this, and [Chris Missal][12] is working on it, we need to solve some few issues:

  * Add the packaging task to our build – not a big deal
  * Have a CI server to generate the packages for us – Chris talked with the Codebetter folks and we will have it done by the end of the week approximately
  * Agree on how we are going organize the packages and distribute it.

The latter issue has two parts: package organization and distribution

<span style="text-decoration: underline;">Package Organization:</span> We are structuring the code so that one can install the core package that has no dependencies. This will give you all extensions that have no extra dependencies. For extensions that depends on other packages we are going to create a specific packages.

For example, if you want to use the [NotAcceptableMessageHandler][13], that has no dependencies, all you have to do is install the core package. On the other hand, if you want to use the [ServiceStackTextFormatter][14], a Media Type Formatter that uses ServiceStack.Text to handle JSON serialization, you will have to install the WebApiContrib.Formatting.ServiceStackTextFormatter package. Our goal is to keep contrib’s users packages dependencies clean, and only have what is really necessary for what the user is trying to do.

<span style="text-decoration: underline;">Distribution:</span> Nuget currently has two [WebApiContrib packages][15]. Those were created for the WCFWeb APIContrib project. This means that replacing those packages will break everything for people that was relying on the WCF version of the package. This might be ok to do, as the WCFWebApi no longer exist and we are going to deprecate the WCFContrib as well. But we haven’t decided that yet. And I’m willing to get some ideas on what should we do to make this as frictionless as possible.

I expect all this to be fixed by next week.

### What’s next

We don’t have an official roadmap besides [the issues list on github][16]. Besides what’s there, I have some ideas that are now floating on my head but I haven’t started working on them yet:

<span style="text-decoration: underline;">Provide a better way of creating/embedding links on resources</span>. The ASP.NET Web API has no built in helper to embed links into resources. My goal is to have a helper to make it easier, and hopefully in a strongly typed way, to configure your resource so that the Media Type Formatter can then pull that information and include the links in the resource representation.

<span style="text-decoration: underline;">Api Documentation Generation</span>. It’s key for the success of a api that it’s well documented. It’s in the [ASP.NET Web API team future roadmap][17] to include something to help in this documentation generation. But, until they get there, we still need to document the web apis that we are creating. My initial idea is to implement the [Swagger][18] specification and use [Swagger-UI][19] to expose the documentation.

### Remember it’s a Contrib effort

The goal of a Contrib effort is to group and curate extensions to the ASP.NET Web API that are created by the people using it. So, if you have an idea or created something that extends the ASP.NET Web API please, come talk to us. Send us an email on [the mailing list][7]; [open an issue][16] to discuss a new idea; send us a pull request. Even if you don’t have an specific idea or feature that you want to discuss but are willing to help our effort, come talk to us.  The WebApiContrib is a community effort and we need your help.

 [1]: http://webapicontrib.codeplex.com/
 [2]: http://www.bizcoder.com/
 [3]: http://www.headspring.com/
 [4]: http://www.thinktecture.com/
 [5]: http://wizardsofsmart.net/
 [6]: https://github.com/WebApiContrib
 [7]: https://groups.google.com/forum/#!forum/webapicontrib
 [8]: https://github.com/WebApiContrib/WebAPIContrib
 [9]: https://github.com/HeadspringLabs
 [10]: http://nuget.org/
 [11]: http://www.openwrap.org/
 [12]: http://chrismissal.lostechies.com
 [13]: https://github.com/WebApiContrib/WebAPIContrib/blob/master/src/WebApiContrib/MessageHandlers/NotAcceptableMessageHandler.cs
 [14]: https://github.com/WebApiContrib/WebAPIContrib/blob/master/src/WebApiContrib.Formatting.ServiceStack/ServiceStackTextFormatter.cs
 [15]: http://nuget.org/packages?q=WebApiContrib
 [16]: https://github.com/WebApiContrib/WebAPIContrib/issues?sort=created&direction=desc&state=open
 [17]: http://aspnetwebstack.codeplex.com/wikipage?title=Roadmap
 [18]: http://swagger.wordnik.com/
 [19]: https://github.com/wordnik/swagger-ui
