---
layout: post
title: "Cross Origin Resource Sharing (CORS)"
date: 2014-10-31T02:07:18+05:30
---
[Same Origin Policy](http://en.wikipedia.org/wiki/Same-origin_policy) an imporatnt concept of web application security model but while biulding RESTful Api's you may need work arounds to tackle it, some are mentioned under:

* Integration Middleware

![Diagram1](/images/cors1e.png)

Here one server acts as proxy for other servers, issue with this approach is that there are more servers to maintain, and there is more latency.

* jsonp
[jsonp](http://en.wikipedia.org/wiki/JSONP) JavaScript Object Notation with padding. Take a pure json, make it function call then eval in the browser.Note same origin policy doesnt apply to resource loading(script tags), lets see an example 

json:{'price':42}
jsonp:callback({'price':42});

{% highlight javascript linenos %}
$.ajax({
	...
	dataType:"json",
	...
});

$.ajax({
	...
	dataType:"jsonp",
	jsonp:"jsonp",
	...
});
{% endhighlight %}

what it does is create a script tag and making your browser "eval" the lot.Each time each request comes in.Some known security holes are there, any script you bring in has access to your data/dom/ private parts.Moreover Jsonp is GET only and for our usual RESTful Api's we need other http verbs too.

* CORS(Cross Origin Resource Sharing)
CORS allows servers to specify who, what can access endpoint directly.It uses plain json , and all http verbs: PUT,DELETE, etc are available

![Diagram2](/images/cors2e.png)

It is trivial to consume, has plain web calls , and is direct although there is some complexity on the server/config side.There is good browser support for details check [here](http://caniuse.com/#feat=cors)
All verbs, all data types


* Client Side Example

{% highlight javascript linenos %}
$.ajax ({
	dataType:"json",
	xhrFields:{withCredentials:true} //pass cookie, credentials
	...
});
{% endhighlight %}

Most of the work happens between browser and server, using http headers called "pre flight checks".The browser passes origin header to server e.g
`Origin:http://www.example-social-network.com`, server responds(header) saying what is allowed `Access-Control-Allow-Origin :http://www.example-social-network.com`

![Diagram4](/images/cors4e.png)

"Pre flight checks" are performed by browser, opaque to client app.Browser enforces.you dont see them, browser Uses "OPTION" http verb.Can be anoyance if we use `Access-Control-Allow-Origin:*`. Downside of approach is that it allows any script with right credentials to pull data from you.

* Common pattern:

`Access-Control-Allow-Origin:$origin-from-request`
The returned value is really echoing back what Origin was checked  off against a whitelist


* Middleware

All app server environments have a way to do the right thing with CORS headers: rack-cors:ruby, Servlet-filter: java, Node :express middleware
etc.

![Diagram3](/images/cors3e.png)

* Other CORS headers

`Access-Control-Allow-Headers` (headers to be included in requests)
`Access-Control-Allow-Methods:GET, PUT , POST , DELETE` etc
`Access-Control-Allow-Credentials:boolean`

* Minimal setup

`Access-Control-Allow-Methods:GET,POST,PUT,DELETE`
`Access-Control-Allow-Credentials:true`
`Access-Control-Allow-Origin:$ORIGIN`
`$ORIGIN = if (inWhitelist(requestOriginHeader))return requestOriginHeader`


* My implemented Code (for nginx):
----------------------------------

{% highlight nginx linenos %}
# pass requests for dynamic content to rails/turbogears/zope, et al
location / {
	 ...
	 if ($http_origin ~* (https://abc.com|https://abc.com|http://xyz.com|http://somerandomsite.org|http://secondrandomsite.com|http://somedomain.com)) {
	     set $cors "true";
	 }
	# Handle CORS
	#set $cors "true";
	# if it's a GET or POST, set the standard CORS responses header
	if ($cors = "true") {
	 add_header 'Access-Control-Allow-Origin' "$http_origin";
	 add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, DELETE, PUT';
	 add_header 'Access-Control-Allow-Credentials' 'true';
	 add_header 'Access-Control-Allow-Headers' 'User-Agent,Keep-Alive,Content-Type';
	}
	...
}
{% endhighlight %}

{% highlight javascript linenos %}
server_name = "mytestsever.com";

//GET
ajaxOptsCrossOrigin = {
	xhrFields: {withCredentials: true}, //adds cookies, credentials
	url: "https://"+server_name+"/get_link/",
	dataType: 'json',
	crossDomain: true,
	data: data_vals,
	success: function(resp){
		//do something on success
	},
};
jQuery.ajax(ajaxOptsCrossOrigin);

//POST
ajaxOpts = {
	url:"https://"+server_name+"/post_link/",
	crossDomain:true,
	xhrFields: {withCredentials:true},
	type:'POST',
	data: jQuery('#some_form').serialize(),
	dataType:'json',
	success: {
		//do something
	},
	error: {
		//do something
	},
};
jQuery.ajax(ajaxOpts);
{% endhighlight %}

For django one can use [django-cors-headers](https://github.com/ottoyiu/django-cors-headers) plugin and add the headers in the view which responds to the urls in the ajax calls.

Learn more about CORS here: [wikipedia](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing), [html5rocks](http://www.html5rocks.com/en/tutorials/cors/), [enable-cors.org](enable-cors.org), [Michael Neale video](http://www.youtube.com/watch?v=rlnhiwN8AnU). Thanks for Reading
