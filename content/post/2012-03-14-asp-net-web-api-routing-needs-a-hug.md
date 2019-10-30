---
title: ASP.NET Web API Routing needs a hug
author: Pedro
type: post
date: 2012-03-14T04:12:50+00:00
url: "2012/03/14/asp-net-web-api-routing-needs-a-hug"
dsq_thread_id:
  - 610313192
categories:
  - .Net
  - WebAPIContrib
tags:
  - ASP.NET Web API
  - WebAPIContrib

---
One of my goals with the [WebAPIContrib][1] is to be able to write as few code as possible in my API application and let the Contrib help me with the boring boilerplate code.

Looking through the ASP.NET Web API samples, tutorials and blog posts out there, the first thing that jumps on my eye is the whole HttpResponseMessage noise.

<noscript>
  <pre><code class="language-c# c#">public HttpResponseMessage&lt;Person&gt; Post(Person person)
{
	_repository.Save(person);
  	var responseMessage = new HttpResponseMessage(person, HttpStatusCode.Created);

  	responseMessage.Headers.Location = new Uri(&ldquo;http://api.example.com/People/&rdquo;+person.Id);
	return responseMessage;
}</code></pre>
</noscript>

I don’t want to have to write all these status code and Location Header settings all over my code. This is boring and adds noise to what actually is important on my controller action. So, my idea was to create objects to represent each status code, something similar [to what Restfulie does][2], and encapsulate all this noise inside these specialized HttpResponseMessages.

I imagined doing something similar to:

<noscript>
  <pre><code class="language-c# c#">public class CreateResponse : HttpResponseMessage
{
	public CreateResponse() : base(HttpStatusCode.Created) { }
}</code></pre>
</noscript>

But, in the create response I also want to set the Location Header of the new resource. As a first pass, I went with: 

<noscript>
  <pre><code class="language-c# c#">public interface IApiResource
{
    void SetLocation(ResourceLocation location);
}


public class ResourceLocation
{
    public Uri Location { get; private set; }

    public void Set(Uri location)
    {
        Location = location;
    }
}

public class CreateResponse : HttpResponseMessage
{
     public CreateResponse()
     {
         StatusCode = HttpStatusCode.Created;
     }

    public CreateResponse(IApiResource resource) :this()
    {
        var location = new ResourceLocation();
        resource.SetLocation(location);
        Headers.Location = location.Location;
    }
}</code></pre>
</noscript>

Now, in the API code I can have:

<noscript>
  <pre><code class="language-c# c#">public class Contact : IApiResource
{
    public int Id { get; set; }
    // ... other Contact Members

    public SetLocation(ResourceLocation location)
    {
        location.Location = new Uri("http://api.contoso.com/contacts/"+Id);
    }
}


public class ContactsController : ApiController
{
    public HttpResponseMessage Post(Contact contact)
    {
       //create new contact resource
       return new CreateResponse(contact);
    }
}</code></pre>
</noscript>

Although this achieve my goal of setting the status code and Location header in the HttpResponseMessage I’m still not happy. Mainly because of manually building the URI of the Location. I don’t have to hardcode URLs when I can use the type system to help me with that.

## Embracing the type system, or not.

I’m really not happy with hardcoding the URI to set the Location Header. A better way of setting that would be:

<noscript>
  <pre><code class="language-c# c#">public SetLocation(ResourceLocation location)
{
    location.SetIn&lt;ContactsController&gt;(x =&gt; x.Get(Id));
}</code></pre>
</noscript>

With that, all I would have to do is generate the URL for that controller action and set it to the Location Header. Piece of cake, it’s just use the built-in UrlHelper. Or, that’s what I thought.

The [System.Web.**Http**.Routing.UrlHelper][3] has a [Route][4] method that returns an URL for a set of route values. OK, so at first glance I just need to translate my Expression<Action<TController>> in a dictionary of route values. This is not a problem, [MVCContrib, for example, uses][5] a [helper from the MVCFutures assembly][6] to do this. As I’m working in the WebAPIContrib, I didn’t want to take a dependency in the MVCFutures just for this. I ended up coding the reflection part myself.

OK, no I have a RouteValueDictionary, to get the URL back is just call urlHelper.Route(routeValues), right? Not so quickly.

## Y’all gotta know the names. All the names

The Route method on UrlHelper is not like the Action method on it’s System.Web.MVC sibling (more on this Mvc x Http later). As you can see in the [MSDN documentation][4], the Route method signature is:

<noscript>
  <pre><code class="language-c# c#">public string Route(
	string routeName,
	IDictionary&lt;string, Object&gt; routeValues
)</code></pre>
</noscript>

Together with the route values you also have to supply the route name. As I’m building a library for others to use while building their APIs, I don’t know which route to use. My first thought was: I’m just going to iterate over the HttpRouteCollection and the first URL I get back would win. That’s when I started learning more about the ASP.NET Web API routing implementation than I was wanting to.

## No foreach for you

When I tried to iterate over the HttpRouteCollection, the following exception was thrown:

<noscript>
  <pre><code class="language-text text">System.NotSupportedException was caught
  Message=This operation is only supported by directly calling it on 'RouteCollection'.
  Source=System.Web.Http.WebHost
  StackTrace:
       at System.Web.Http.WebHost.Routing.HostedHttpRouteCollection.GetEnumerator()
       at WebApiContrib.Internal.RoutingHelper.GetUriFor(RouteValueDictionary routeValues, HttpControllerContext controllerContext) in C:\dev\WebAPIContrib\src\WebApiContrib\Internal\RoutingHelper.cs:line 42
  InnerException: </code></pre>
</noscript>

Checking on Google, I found out that I had to get a read lock first, before iterating over the collection. This is done through the [GetReadLock][7] method that returns a disposable lock object. `:SadTrombone:` Fooled one more time by the Mvc vs. Http identity issues. Only [System.Web.Routing.RouteCollection][8] has the GetReadLock method. The [HttpRouteCollection][9] counterpart doesn’t have it.&#160; So, I had to fall back to for loops to get the routes out of the HttpRouteCollection.

Then, another surprise. To refresh your memory, we are doing all this looping things in order to get the route name so that we can use the UrlHelper to get the Location header URI. So, after I was able to get a route object instance, I found out that there is no way to get it’s name. Yeap, you create a route with a name. The UrlHelper asks you for a route name. You can even check in the collection if there are any routes with a given name. But you can’t get the name from a route instance. With dotpeek’s help I figured out that the name is used as the key in the private dictionary that the HttpRouteCollection wraps. But the HttpRouteCollection doesn’t make the Keys property of the dictionaty available to the outside world. That’s it. There is no way for you to programmatically find a routes name.

## Ah, the ControllerContext

At this point I was already pretty frustrated. I was working on this during the Dallas Day of .Net and I was afraid I wouldn&#8217;t be able to keep cursing only in my head anymore. Actually I believe I said one or two “WTF!” loudly. Anyways, I gave up the whole let’s-find-out-the-route-name-thing and decided to hard code the default one instead. Just so that I could at least see the thing working. 

To my despair, I realized that the System.Web.Http.Routing.UrlHelper depends on the HttpControllerContext, not in the RequestContext as System.Web.Mvc.UrlHelper. As I can’t simply create, or at least not as easily, an instance of the HttpControllerContext as [I can create one][10] of the RequestContext, I just changed the constructor of the CreateResponse to also include the HttpControllerContext. 

<noscript>
  <pre><code class="language-c# c#">public HttpResponseMessage&lt;Contact&gt; Post(Contact contact)
{
    this.repository.Post(contact);

    return new CreateResponse&lt;Contact&gt;(contact, this.ControllerContext);
}
</code></pre>
</noscript>

I’m definitely not happy with this and I won’t include any of it, as it is, in the WebAPIContrib.

## Least Surprise who?

Let’s be clear: this whole thing of System.Web.Mvc vs. System.Web.Http is confusing as hell. Things have the same name but are different types. The api for those types are different. They behave differently. I can see a lot of people struggling as I did when trying to use whatever they are using in their MVC projects in a Web API one.

## But the Web API is still in beta and this is what a beta is for, right?

Yes, the ASP.NET Web API is still in Beta and I expect a lot of the library api to change. [As Henrik Nielsen asked me to do][11], I’ll register the issues I had on user voice. And I can only hope they will fix it. Or maybe accept a pull request ;-)

 [1]: https://github.com/HeadspringLabs/WebAPIContrib
 [2]: https://github.com/mauricioaniche/restfulie.net/tree/master/Restfulie.Server/Results
 [3]: http://msdn.microsoft.com/en-us/library/system.web.http.routing.urlhelper(v=vs.108).aspx
 [4]: http://msdn.microsoft.com/en-us/library/hh834562(v=vs.108).aspx
 [5]: http://mvccontrib.codeplex.com/SourceControl/changeset/view/bb5feec203bb#src%2fMVCContrib%2fActionResults%2fRedirectToRouteResult.cs
 [6]: http://aspnet.codeplex.com/SourceControl/changeset/view/72551#266392
 [7]: http://msdn.microsoft.com/en-us/library/system.web.routing.routecollection.getreadlock.aspx
 [8]: http://msdn.microsoft.com/en-us/library/cc680101.aspx
 [9]: http://msdn.microsoft.com/en-us/library/system.web.http.httproutecollection(v=vs.108).aspx
 [10]: https://github.com/mauricioaniche/restfulie.net/blob/master/Restfulie.Server/Marshalling/UrlGenerators/AspNetMvcUrlGenerator.cs#L14
 [11]: https://twitter.com/#!/frystyk/status/178704676152283136
