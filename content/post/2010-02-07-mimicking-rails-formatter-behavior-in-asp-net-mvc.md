---
title: Mimicking Rails formatter behavior in ASP.NET MVC
author: Pedro
type: post
date: 2010-02-07T18:53:08+00:00
aliases:
  - /2010/02/07/mimicking-rails-formatter-behavior-in-asp-net-mvc
dsq_thread_id:
  - 395135132
categories:
  - .Net
tags:
  - ASP.NET MVC
  - Rails
  - Routes
  - TDD

---
I was reading [this (pt-BR) thread][1] at the brazilian .Net mailing list &#8211; [dotNetArchitects][2] – which, at first, did not have nothing to do with Rails nor output format. But, as usual, the thread deviated from the initial subject – what is not a bad thing &#8211; and somehow got into the fact that it would be nice to have in ASP.NET, more specifically in [ASP.NET MVC][3], a behavior similar to what Rails have by default to handle output formatters.

In Rails is possible to handle the format that will be returned by the controller as in the following example:

```ruby
class PeopleController < ApplicationController
    def index 
        @people = People.find(:all)
        respond_to do |format|
            format.html
            format.json(render :json > @people.to_json)
            format.xml(render :xml > @people.to_xml)
    end
end
```

With the controller above, if one requests the url `/people/index` the result will be in `html`. but, if `people/index.json` is requested instead, the result will be a `json` file.

That’s a great feature when one is developing an API and let the client code to decide the format it wants to receive the data. Unfortunately, we can’t do that out of the box with ASP.NET MVC. But, ASP.NET MVC gives us lots of extensions points and that allows us to quite easily implement the functionality to mimic this neat behavior. 

_First, a disclaimer. The solution that I will present is heavily based on the solution present in the routing chapter of the book [ASP.NET MVC in Action][4]. The new edition of the book, now covering MVC 2, is, by the time I write this post, in public review. You can get more information about it on this_ [_post_][5] _from Jeffrey Palermo. If you are an ASP.NET developer and don’t have read the book yet, stop everything you are doing and [go buy it now][4]._

All that said, lets have some fun.

In our application we have this simple Person entity:

```csharp
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

The application itself is really simple, it just shows a list of People. And yes, I was lazy and used the MVC sample.

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_1" src="https://static.flickr.com/4012/4334558401_cdf8faffec.jpg" />

I will omit the view code for the sake of brevity as it has nothing to do with the goal of this post.

The `PeopleController` is pretty dumb too, it just returns a collection of `Person` to the View: 

```csharp
public class PeopleController : Controller
{
    public ActionResult Index()
    {        
        var people = new[]
                {
                    new Person{FirstName = "Joao", LastName = "Silva"},
                    new Person{FirstName = "John", LastName = "Doe"},
                    new Person{FirstName = "Jane", LastName = "Smith"}
                 };
        return View(people);
    }
}
```


So, it works, pretty straight-forward with default routing. Now, we want to mimic the Rails formatter behavior. So, what we want is that the url `/People/Index.json` returns a `json` file instead of the html page. Ok, what we first need to do is check to see if this new url will be routed to the correct Action with the actual route configuration. 

With the help of [MvcContrib][6] project, we can write the following test to check if our route works as we expect:

```csharp
[TestFixtureSetUp]
public void Setup()
{
    MvcApplication.RegisterRoutes(RouteTable.Routes);
}

[Test]
public void Should_map_People_url_to_people_with_default_action()
{
    "~/people".Route().ShouldMapTo<PeopleController>(x => x.Index());
}

[Test]
public void Should_map_People_index_json_url_to_people_matching_index_action()
{
    "~/people/index.json".Route().ShouldMapTo<PeopleController>(x => x.Index());
}
```


As expected, the former test passes, but the latter, the one that tests the new behavior we want to test don’t. And more important, it fails as we expected it to fail.

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_2" src="https://static.flickr.com/4003/4335338096_9f9e265960.jpg" /></p> 

In order to make it work, we need to add a new route to our Route Dictionary.

```csharp
routes.MapRoute(
"Format",
"{controller}/{action}.{format}/{id}",
new {id = ""});
```


With this new route defined, we now get a green test. Neat.

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_3" src="https://static.flickr.com/2769/4335379468_b6e8aae0dc.jpg" /></p> 

With the route working and the request being routed to the right method, the controller now needs to extract the format information from the route data and handle the output format accordingly.&#160; To achieve this, I’ll create a [Layer SuperType][7], a abstract class that will derive from the base `Controller` class. The `PeopleController`, then, will derive from this new class instead of the base `Controller` one.

This new Controller class, that I will name `RailsWannabeController`, will override the `OnActionExecuting` method of the Controller base class in order to extract the requested format from the `RouteData`. It also will have a property named `Format` who will store the format information. Finally, it will have a `FormatResult` method that returns the right `ActionResult` accordingly to the requested format.

Here is the test to ensure that `RailsWannabeController` correctly extracts the format information out of the `RouteData`.

```csharp
[Test]
public void Should_extract_format_information_from_RouteData()
{
    var expectedFormat = "json";
    var routeData = Stub<RouteData>();
    routeData.Values.Add("format",expectedFormat);
    
    var filterContext = 
        new ActionExecutingContext()
        {
            Controller = Stub<RailsWannabeController>(),
            RouteData = routeData
         };

    var controller = new StubController();
    controller.ActionExecuting(filterContext);
    var requestedFormat = controller.RequestedFormat;
    Assert.AreEqual(expectedFormat,requestedFormat);
}
```


As the `RailsWannabeController` will be a abstract class, a `<strike>MockController</strike> StubController` concrete class has to be created. This class will derive from `RailsWannabeController` and will have a public method to enable the `OnActionExecuting` method to be called. It also will have a `RequestedFormat` property to expose the value of the requested format extracted from the `RouteData`.

```csharp
public class StubController : RailsWannabeController
{
    public string RequestedFormat { get { return base.Format; } }
    
    public void ActionExecuting(ActionExecutingContext filterContext)
    {
        base.OnActionExecuting(filterContext);
    }
}
```
 
Now, all we need to do to make this test pass is code the RailsWannabeController class.

```csharp
public class RailsWannabeController : Controller
{
    protected static string[]  ValidFormats = new [] {"html","json","xml"};
    protected string Format { get; set; }

    private const string formatKey = "format";

    protected override void OnActionExecuting(ActionExecutingContext filterContext)
    {
        base.OnActionExecuting(filterContext);
        ExtractRequestedFormat(filterContext.RouteData.Values);
    }
    
    private void ExtractRequestedFormat(RouteValueDictionary routeValues)
    {
        if(routeValues.ContainsKey(formatKey))
        {
            var requestedFormat = routeValues[formatKey].ToString().ToLower();
            if(ValidFormats.Contains(requestedFormat))
            {
                Format = requestedFormat;
                return;
            }
        }
        Format = "html";
    }
}
```


Now that we got a green bar, we have to test if the `ActionResult` returned by the `FormatResult` method is from the type requested. That means that if the “`json`” format is passed in the `RouteData`, the `FomatResult` method the object returned must be of type `JsonResult. `

```csharp
public void Should_return_the_correct_Action_Result()
{
    var expectedFormat = "json";
    var routeData = Stub<RouteData>();
    routeData.Values.Add("format", expectedFormat);

    var filterContext = new ActionExecutingContext()
    {
        Controller = Stub<RailsWannabeController>(),
        RouteData = routeData
    };

    var controller = new StubController();
    controller.ActionExecuting(filterContext);

    var people = new[]{
                         new Person{FirstName = "Joao", LastName = "Silva"},
                         new Person{FirstName = "John", LastName = "Doe"},
                         new Person{FirstName = "Jane", LastName = "Smith"}
                     };

    var formattedResult = controller.GetFormattedResult(people);
    Assert.AreEqual(typeof(JsonResult),formattedResult.GetType());
}
```
There is a LOT of code in this test. Much more then I’d like it to have. But to keep the code as explicit as possible, for sake of clarity, I left it that way. I’ll let the refactoring of this code as an exercise for the reader.

As you may guess from the code above, the~~`MockController`~~`StubController` class had to be modified to include the `GetFormattedResult` method.

```csharp
public class StubController : RailsWannabeController
{
    public string RequestedFormat { get { return base.Format; } }
    
    public void ActionExecuting(ActionExecutingContext filterContext)
    {
        base.OnActionExecuting(filterContext);
    }

    public ActionResult GetFormattedResult(object model)
    {
        return base.FormatResult(model);
    }
}
```

In the `RailsWannabeController` we have to create a `FormatResult` method that, as it name states, format the result accordingly to the format requested.

```csharp
protected ActionResult FormatResult(object model)
{
    switch (Format)
    {
        case "html":
            return View(model);
        case "json":
            return Json(model);
        case "xml":
            return new XmlResult(model);
        default:
            throw new FormatException(
                string.Format("The format \"{0}\" is invalid", Format));
    }
}
```


ASP.NET MVC framework gives us the `Json` method out of the box. The `XmlResult` is provided by the MvcContrib project.

Now, all we have to do is change the `PeopleController` class to derive from the Layer SuperType instead of the Controller base class. And change the `View()` method call to be a call to the `FormatResult` method.

```csharp
public class PeopleController : RailsWannabeController
{
    public ActionResult Index()
    {        
        var people = new[]
            {
                new Person{FirstName = "Joao", LastName = "Silva"},
                new Person{FirstName = "John", LastName = "Doe"},
                new Person{FirstName = "Jane", LastName = "Smith"}
             };
        return FormatResult(people);
    }
}
```

OK, all the tests are green and we are done coding. Let’s check if all we’ve done works properly.

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_4" src="https://static.flickr.com/4068/4337240149_0c587a80fd.jpg" />

Using the default route we get the same result. Great. 

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_6" src="https://static.flickr.com/4070/4337983998_793d67231f.jpg" />

If “json” format is provided in the URL we get a `json` file instead. Woot.

<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="Rails_MVC_7" src="https://static.flickr.com/4011/4337240393_cf05be4942.jpg" />

Finally, requesting a xml we get the result in a XML file. That’s it we are done.

I know the post is a little bit long, but that’s because my intention was to explicit all the process of mimicking the behavior of the Rails formatter. And, as you should do as well, I managed to have the code I was working on to be covered by tests that gave me the confidence that I wasn’t breaking anything while I was making the changes. It is specially important to cover your routes with tests as route changes can introduce some hard to find bugs. I hope this post helps to show that, although ASP.NET MVC may not have all the features we want it to have, it’s high extensibility allows us to extend it and easily introduce new behaviors.

As I’m not a native English speaker, I ask and encourage you to point out not only technical mistakes but also any language related mistakes that I’ve made at this post. 

**UPDATE:** As Giovanni correctly pointed out in his comment, the `MockController` class is not a Mock but a Stub. I renamed the class to `StubController` to avoid misunderstandings. Thanks, Giovanni.

 [1]: https://groups.google.com/group/dotnetarchitects/browse_thread/thread/036023bee2897584?hl=pt#
 [2]: https://www.dotnetarchitects.net/
 [3]: https://www.asp.net/mvc/
 [4]: https://www.amazon.com/gp/product/1933988622?ie=UTF8&tag=pedroreys-20&linkCode=as2&camp=1789&creative=9325&creativeASIN=1933988622
 [5]: https://jeffreypalermo.com/blog/mvc-2-in-action-book-conducting-public-reviews/
 [6]: https://www.codeplex.com/MVCContrib
 [7]: https://martinfowler.com/eaaCatalog/layerSupertype.html
