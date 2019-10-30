In the <a href="http://blog.pedroreys.com/2010/02/07/mimicking-rails-formatter-behavior-in-asp-net-mvc/" target="_blank">previous post</a> I spiked a solution to return different result types depending on a requested format. While the solution I came up with was sufficient to address the points issued on a thread at .NetArchitects, <a href="http://www.lostechies.com/blogs/hex/" target="_blank">Eric Hexter</a> hit the nail on the head with the following <a href="http://twitter.com/ehexter/status/9712355635" target="_blank">tweet</a>:

> @[pedroreys](http://twitter.com/pedroreys) nice post on the formatter. I would consider http headers instead of routing, although that is hard to manually debug

The right way to specify the types which are accepted for the response is by using the <a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html" target="_blank">Accept</a> request-header. It’s been a while since Eric’s tweet, but here is the solution I came up with.

Lemme first define the expectations I defined:

If the Accept header of the request

&#8211; Contains <font face="Courier New">“text/html”</font> a <font face="Courier New">ViewResult</font> should be returned.

&#8211; Contains <font face="Courier New">“application/json”</font> and not contains “<font face="Courier New">text/html</font>” a <font face="Courier New">JsonResult</font> should be returned

&#8211; Contains “<font face="Courier New">text/xml</font>” and not contains “<font face="Courier New">text/html</font>” a <font face="Courier New">XmlResult</font> should be returned

If the requested content type can not be handled by none of the previous statements, a response with a <a href="http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html" target="_blank">HTTP Status Code 406 – Not Acceptable</a> should be returned.

For this post. I will start with the same controller class that I used in the previous one:

&#160;

<pre class="brush: csharp;">public class ClientController : Controller
    {
        public ActionResult Index()
        {
            var clients = new[] {
            new Client {FirstName = "John", LastName = "Smith"},
            new Client {FirstName = "Dave", LastName = "Boo"},
            new Client {FirstName = "Garry", LastName = "Foo"}
                               };

            return View(clients);
        }

    }

    public class Client
    {
        public string FirstName{get;set;}

        public string LastName{get; set; }
        
    }</pre>

&#160;

In order to be able to format the result based on the requested content-type on the HTTP header, I will create a base controller class that will provide this functionality to the controller. I will call it <font face="Courier New">ContentTypeAwareController</font>.

&#160;

<pre class="brush: csharp;">public class ContentTypeAwareController : Controller
{
   public virtual IContentTypeHandlerRepository HandlerRepository { get; set; } 

    protected ActionResult ResultWith(object model)
        {
            var handler = HandlerRepository.GetHandlerFor(HttpContext);
            return handler.ResultWith(model);
        }
    }</pre>

&#160;

That’s it. This base class is responsible just to ask the Handler Repository for the right handler to the given HttpContext and call the ResultWith() method on the returned handler.

The IContentTypeHandler interface is pretty simple as well:

&#160;

<pre class="brush: csharp;">public interface IContentTypeHandler
{
    bool CanHandle(HttpContextBase context);
    ActionResult ResultWith(object model);
}</pre>

&#160;

Simple and intuitive, right?

Let’s implement this interface then, but first let’s set the expectations as unit tests:

&#160;

<pre class="brush: csharp;">[TestFixture] 
    public class JsonHandlerTest : HandlerTestBase 
    { 
        [Test] 
        public void should_return_a_JsonResult() 
        { 
            var handler = new JsonHandler(); 
            var result = handler.ResultWith("whatever"); 
            result.ShouldBeType&lt;JsonResult&gt;(); 
        } 

        [Test] 
        public void should_Handle_request_when_content_type_requested_contains_Json() 
        { 
            Headers.Add("Accept", "application/json"); 

            var handler = new JsonHandler(); 
            handler.CanHandle(HttpContext).ShouldBeTrue(); 
        } 

        [Test] 
        public void should_not_handle_request_when_content_type_requested_does_not_contain_json() 
        { 
            Headers.Add("Accept", "text/xml"); 

            var handler = new JsonHandler(); 
            handler.CanHandle(HttpContext).ShouldBeFalse(); 
        } 
    }</pre>

&#160;

To keep the post short, I will put here just the test above, all the other tests are in the solution on <a href="http://github.com/pedroreys/BlogPosts" target="_blank">github</a>. 

The implementation of the <font face="Courier New">JsonHandler</font> is below:

&#160;

<pre class="brush: csharp;">public class JsonHandler : IContentTypeHandler
    {
        public const string _acceptedType = "application/json";
        public bool CanHandle(HttpContextBase context)
        {
            var acceptHeader = context.Request.Headers.Get("Accept"); 

            return acceptHeader.Contains(_acceptedType);
        } 

        public ActionResult ResultWith(object model)
        {
            return new JsonResult
                       {
                           Data = model,
                           ContentEncoding = null,
                        JsonRequestBehavior = JsonRequestBehavior.AllowGet
                       };
        }</pre>

&#160;

It’s important to set the value for the <font face="Courier New">JsonRequestBehavior</font> property on the <font face="Courier New">JsonResult</font> object. Otherwise you will end-up with a <font face="Courier New">InvalidOperationException</font>.

The other handlers are pretty straight-forward as well.

The one to handle requests for Xml:

&#160;

<pre class="brush: csharp;">public class XmlHandler : IContentTypeHandler
{
    public const string _acceptedType = "text/xml";

    public bool CanHandle(HttpContextBase context)
    {
        var acceptHeader = context.Request.Headers.Get("Accept");

        return acceptHeader.Contains(_acceptedType);
    }

    public ActionResult ResultWith(object model)
    {
        return new XmlResult(model);
    }
}</pre>

&#160;

And the other to be the default, Html handler:

&#160;

<pre class="brush: csharp;">public class HtmlHandler : IContentTypeHandler
{
    public const string _acceptedType = "text/html";
    
    public bool CanHandle(HttpContextBase context)
    {
        var acceptHeader = context.Request.Headers.Get("Accept");

        return acceptHeader.Contains(_acceptedType);
        
    }

    public ActionResult ResultWith(object model)
    {
        return new ViewResult
                   {
                       ViewData = new ViewDataDictionary(model),
                   };
    }
}</pre>

&#160;

&#160;

And finally, we will have a handler to return the 406 code in case of the others handlers not being able to handle the requested content type.

&#160;

<pre class="brush: csharp;">public class NotAcceptedContentTypeHandler : IContentTypeHandler
{
    public bool CanHandle(HttpContextBase context)
    {
        return true;
    }

    public ActionResult ResultWith(object model)
    {
        return new NotAcceptedContentTypeResult();
    }
}</pre>

&#160;

With all that code in place, all we have left to do is to make the <font face="Courier New">ClientController</font> to inherit from the <font face="Courier New">ContentTypeAwareController</font> and call the <font face="Courier New">ResultWith()</font> method:

&#160;

<pre class="brush: csharp;">public class ClientController : ContentTypeAwareController 
{
    public ActionResult Index()
    {
        var clients = new[]{
            new Client {FirstName = "John", LastName = "Smith"},
            new Client {FirstName = "Dave", LastName = "Boo"},
            new Client {FirstName = "Garry", LastName = "Foo"}
                           };

            return ResultWith(clients);
        }

    }</pre>



That’s it. The Client Controller now is able to respond to Json, Xml or Html requests. Let’s check it.

The web page is still being rendered as expected. Fine.

&#160;

[<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="html" src="http://farm5.static.flickr.com/4043/4594093858_af5b693db0.jpg" />](http://www.flickr.com/photos/22313104@N04/4594093858/ "html")

&#160;

But now, let’s see what responses do I get when I change the Http Header to “<font face="Courier New">application/json</font>”:

&#160;

[<img style="border-bottom: 0px; border-left: 0px; display: block; float: none; margin-left: auto; border-top: 0px; margin-right: auto; border-right: 0px" border="0" alt="json" src="http://farm5.static.flickr.com/4061/4594220022_321ba545c1.jpg" />](http://www.flickr.com/photos/22313104@N04/4594220022/ "json")

&#160;

Yeah,&#160; a Json result. Cool. What about “<font face="Courier New">text/xml</font>”:

&#160;

[<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="xml" src="http://farm4.static.flickr.com/3391/4593603183_2b80e4c978.jpg" />](http://www.flickr.com/photos/22313104@N04/4593603183/ "xml")

&#160;

Yeap, that works too. Finally, let’s see if the 406 status code is returned when an invalid content type is requested:

&#160;

[<img style="display: block; float: none; margin-left: auto; margin-right: auto" border="0" alt="406" src="http://farm2.static.flickr.com/1136/4594219796_7606432360.jpg" />](http://www.flickr.com/photos/22313104@N04/4594219796/ "406")</p> 

&#160;

So, with not so much code the Client Controller now is able to format its result based on the Content Type requested on the HTTP Header, as it should have been before. Off course this is a spike and the code can be improved. Feel free to grab it from <a href="http://github.com/pedroreys/BlogPosts" target="_blank">github</a> and improve it.