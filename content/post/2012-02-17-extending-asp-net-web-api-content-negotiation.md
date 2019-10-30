---
title: 'Extending ASP.NET Web API  Content Negotiation'
author: Pedro
type: post
date: 2012-02-17T14:00:00+00:00
url: "2012/02/17/extending-asp-net-web-api-content-negotiation"
dsq_thread_id:
  - 579759610
categories:
  - .Net
tags:
  - ASP.NET Web API
  - conneg
  - rest

---
The ASP.NET team released the beta version of the [ASP.NET Web API][1], previously known as WCF Web API, as part of the [beta release of ASP.NET MVC 4][2]. Having experience implementing web APIs with [Restfulie][3], I was curious and decided to check how the ASP.NET Web API works to compare it with Restfulie.

The first thing I noticed was a difference in the Content Negotiation implementation. I don’t intend to do a full comparison here, but to describe how to use one of the extension points in the Web API to add the behavior that I wanted.

If you are not interested in in the reasons why the Content Negotiation implementation differs, and why I prefer the Restfulie one and just want to see DA CODEZ, feel free to jump to the “Message Handlers to the Rescue” part.

## Content Negotiation

Content Negotiation, simply put, is the process of figuring out what is the media-type that needs to be used in the Resource Representation that is going to be returned in a HTTP Response. This negotiation between client and server happens through the interpretation of the HTTP Accept Header values.

When a client makes a request, if the client wants to specify a set of media-types that it accepts the resource to be formatted into, then these media types should be included in the Accept Header. All the semantics and options regarding the usage of the Accept header can be found on [section 14.1][4] of [RFC 2616][5] (HTTP specification). In the Accept Header definition there is a part that reads:

> If no Accept header field is present, then it is assumed that the client accepts all media types. If an Accept header field is present, and if the server cannot send a response which is acceptable according to the combined Accept field value, then the server SHOULD send a 406 (not acceptable) response.

The SHOULD key word in the RFC 2616, according to [section 1.2][6] of the RFC, is to be interpreted as described in the [RFC 2119][7]. The description of SHOULD in this RFC is:

> SHOULD   This word, or the adjective &#8220;RECOMMENDED&#8221;, mean that there may exist valid reasons in particular circumstances to ignore a particular item, but the full implications must be understood and carefully weighed before choosing a different course.

With this definition, I interpret the Accept Header behavior as: unless you have a specific reason not to, if the HTTP Request contains an Accept Header and the server does not support the requested media-type, a 406 – Not Acceptable response should be returned.  Returning other Status Code might be OK, as it’s not mandatory to return 406, but this should be the exception, not the default behavior.

## ASP.NET Web API Content Negotiation Implementation

The namespace System.Net.Http.Formatting has a MediaTypeFormatter class whose instances are used by the Web API (I’m going to omit the ASP.NET part from now on) to format the resource representation to a specific media-type.

The Web API ships with JsonMediaTypeFormatter and XmlMediaTypeFormatter implementations. And it defaults to the JSON formatter when no specific media-type is requested. So, if the Accept Header is not included in the HTTP Request, Web API will use the JSON formatter to format the Resource Representation that is going to be returned.

You can add your own MediaType formatter by adding your formatter implementation to the MediaTypeFormatter collection of the HTTPConfiguration object in the GlobalConfiguration class.

## It seems like something is not right here

While I was playing with the Web API, I decided to test the behavior of the Content Negotiation implementation in the scenario where the media-type in the Accept Header is not accepted by the server. That is, when the HTTP Request Accept Header specifies a media-type that the Web API doesn’t have a MediaTypeFormatter implementation that can handle it. By the definition described in the Content Negotiation section above, I expected to receive a 406 response back. To my surprise, I got a 200 – OK response back with a Resource Representation with text/json contant-type. Web API fell back to the default MediaTypeFormatter instead of returning 406.

<noscript>
  <pre><code class="language-text text">C:\dev&gt; curl -v -H "Accept: application/some_unknown_media_type" localhost:1705/api/values
* About to connect() to localhost port 1705 (#0)
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 1705 (#0)
&gt; GET /api/values HTTP/1.1
&gt; User-Agent: curl/7.21.1 (i686-pc-mingw32) libcurl/7.21.1 OpenSSL/0.9.8r zlib/1.2.3
&gt; Host: localhost:1705
&gt; Accept: application/some_unknown_media_type
&gt;
&lt; HTTP/1.1 200 OK
&lt; Server: ASP.NET Development Server/10.0.0.0
&lt; Date: Fri, 17 Feb 2012 01:27:42 GMT
&lt; X-AspNet-Version: 4.0.30319
&lt; Transfer-Encoding: chunked
&lt; Cache-Control: no-cache
&lt; Pragma: no-cache
&lt; Expires: -1
&lt; Content-Type: application/json; charset=utf-8
&lt; Connection: Close
&lt;
["value1","value2"]* Closing connection #0</code></pre>
</noscript>

This was a surprise to me, so I filled a bug report for this in the Web API Forum. [Henrik Frystyk Nielsen][8] replied saying:

> It&#8217;s a reasonable to respond either with a 200 or with a 406 status code in this case. At the moment we err on the side of responding with a 2xx but I completely agree that there are sceanarios where 406 makes a lot of sense. Ultimately we need some kind of switch for making it possible to respond with 406 but it&#8217;s not there yet.

Fine. The 406 response is not mandatory, the Web API is still in beta. I understand the 406 handling not being there yet. And, the last thing I want to do is to get into a discussion of RFC semantics with one of the authors of the specification  :P

## Message Handlers to the Rescue

When a HTTP Request is made to a Web API application, that request is passed  to an instance of the HttpServer class. This class derives from HttpMessageHandler and it will delegate the processing of the HTTP request to the next handler present in the HttpConfiguration.MessageHandlers collection. All handlers in the collection will be called to process the request, the last one being the HttpControllerDispatcher that, as you can tell, will dispatch the call to an Action in a Controller. [FubuMVC][9] [Behavior Chains][10] implements an approach similar to this. (Fubu guys, I know it’s not the same thing, but y’all get my point :))

With that said, in order to return a response with 406 status code, what I need to do is implement a HttpMessageHandler that does that.

<noscript>
  <pre><code class="language-c# c#">public class NotAcceptableConnegHandler : DelegatingHandler
{
	private const string allMediaTypesRange = "*/*";

	protected override Task&lt;HttpResponseMessage&gt; SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
	{
		var acceptHeader = request.Headers.Accept;
		if (!acceptHeader.Any(x =&gt; x.MediaType == allMediaTypesRange))
		{
			var hasFormetterForRequestedMediaType = GlobalConfiguration
								.Configuration
						                .Formatters
								.Any(formatter =&gt; acceptHeader.Any(mediaType =&gt; formatter.SupportedMediaTypes.Contains(mediaType)));

			if (!hasFormetterForRequestedMediaType)
				return Task&lt;HttpResponseMessage&gt;.Factory.StartNew(() =&gt; new HttpResponseMessage(HttpStatusCode.NotAcceptable));
		}
		return base.SendAsync(request, cancellationToken);
	}   
}</code></pre>
</noscript>

With the handler created, the next step is to configure Web API to include this handler in the HTTP Request processing pipeline. To do so, in the Application_Start method in the Global.asax , add an instance of the new handler to the MessageHandlers collection.

<noscript>
  <pre><code class="language-c# c#">protected void Application_Start()
{
        //Default Registration and configuration stuff
	AreaRegistration.RegisterAllAreas();

	RegisterGlobalFilters(GlobalFilters.Filters);
	RegisterRoutes(RouteTable.Routes);

	BundleTable.Bundles.RegisterTemplateBundles();
        
        //Call to a Configure method created to add message handlers to the Request pipeline
	Configure(GlobalConfiguration.Configuration);
}

static void Configure(HttpConfiguration configuration)
{
	configuration.MessageHandlers.Add(new NotAcceptableConnegHandler());
}</code></pre>
</noscript>

With the application configured to use the custom handler, when that same request is made, a response with 406 Status Code is now returned.

<noscript>
  <pre><code class="language-text text">C:\dev&gt; curl -v  -H "Accept: text/unknown_media_type" localhost:1705/api/values
* About to connect() to localhost port 1705 (#0)
*   Trying 127.0.0.1... connected
* Connected to localhost (127.0.0.1) port 1705 (#0)
&gt; GET /api/values HTTP/1.1
&gt; User-Agent: curl/7.21.1 (i686-pc-mingw32) libcurl/7.21.1 OpenSSL/0.9.8r zlib/1.2.3
&gt; Host: localhost:1705
&gt; Accept: text/unknown_media_type
&gt;
&lt; HTTP/1.1 406 Not Acceptable
&lt; Server: ASP.NET Development Server/10.0.0.0
&lt; Date: Fri, 17 Feb 2012 02:59:58 GMT
&lt; X-AspNet-Version: 4.0.30319
&lt; Cache-Control: no-cache
&lt; Pragma: no-cache
&lt; Expires: -1
&lt; Content-Length: 0
&lt; Connection: Close
&lt;
* Closing connection #0</code></pre>
</noscript>

## The power of Russian Dolls

The Web API HTTP Request pipeline provides a powerful extension point to introduce specialized, SRP adherent, handlers keeping the controller action implementation simple and focused.

 [1]: http://www.asp.net/web-api
 [2]: http://www.asp.net/vnext/overview/downloads
 [3]: http://restfulie.caelum.com.br/
 [4]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
 [5]: http://www.w3.org/Protocols/rfc2616/rfc2616.html
 [6]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec1.html#sec1.2
 [7]: http://www.ietf.org/rfc/rfc2119.txt
 [8]: http://en.wikipedia.org/wiki/Henrik_Frystyk_Nielsen
 [9]: http://mvc.fubu-project.org/
 [10]: http://codebetter.com/jeremymiller/2011/01/09/fubumvcs-internal-runtime-the-russian-doll-model-and-how-it-compares-to-asp-net-mvc-and-openrasta/
