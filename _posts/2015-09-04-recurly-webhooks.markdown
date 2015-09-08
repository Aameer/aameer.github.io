---
layout: post
category: articles
analytics: true
tags: [Recurly, Webhooks, python, Django, Ip authentication]
comments: true
share: true
title: "recurly webhooks"
date: 2015-09-04T11:23:52+05:30
---

webhooks:
---------
A webhook in web development is a method of augmenting or altering the behavior of a web page, or web application, with custom callbacks.
These callbacks may be maintained, modified, and managed by third-party users and developers who may not necessarily be affiliated with 
the originating website or application [more on wiki](https://en.wikipedia.org/wiki/Webhook).

webhooks on reculry:
--------------------
Recurly allows us to use webhooks to get the updates about account, take the case of a user who has cancelled the subscription which will
expire at the end of current billing cycle (say end of the month) and you want to get notified when this will happen so that you can 
perform some buisness logic.

setup:
------
To setup webhooks on recurly we have to login into recurly *console > Developers > webhooks.* There we will add the url to which recurly
will try to *POST* say `http://mysite.com/test_recurly_webhook/` and then add our credentials. After this we need to setup a view which will 
listen to this *POST* for django.Since we are using `@csrf_exempt` we will try to only allow requests from the ip list (from [docs](https://docs.recurly.com/push-notifications))

![Recurly Webhook Cosole](/images/recurly_webhook_console.png)

mentioned under:

{% highlight python linenos %}
from django.http import HttpResponse,
from django.views.decorators.csrf import csrf_exempt
import recurly

allowedIps=['64.74.141.0', '8.36.93.0.0', '64.74.141.1', '8.36.93.0.1', '64.74.141.2', 
    '8.36.93.0.2', '64.74.141.3', '8.36.93.0.3', '64.74.141.4', '8.36.93.0.4', '64.74.141.5',
    '8.36.93.0.5', '64.74.141.6', '8.36.93.0.6', '64.74.141.7', '8.36.93.0.7', '64.74.141.8', 
    '8.36.93.0.8', '64.74.141.9', '8.36.93.0.9', '64.74.141.10', '8.36.93.0.10', '64.74.141.11', 
    '8.36.93.0.11', '64.74.141.12', '8.36.93.0.12', '64.74.141.13', '8.36.93.0.13', '64.74.141.14', 
    '8.36.93.0.14', '64.74.141.15', '8.36.93.0.15', '64.74.141.16', '8.36.93.0.16', '64.74.141.17', 
    '8.36.93.0.17', '64.74.141.18', '8.36.93.0.18', '64.74.141.19', '8.36.93.0.19', '64.74.141.20', 
    '8.36.93.0.20', '64.74.141.21', '8.36.93.0.21', '64.74.141.22', '8.36.93.0.22', '64.74.141.23', 
    '8.36.93.0.23']

def allow_by_ip(view_func):
    def authorize(request, *args, **kwargs):
        user_ip = request.META['REMOTE_ADDR']
        for ip in allowedIps:
            if ip==user_ip:
                return view_func(request, *args, **kwargs)
        return HttpResponse('Invalid Ip Access!')
    return authorize

@csrf_exempt
@allow_by_ip
def recurly_webhook(request):

    #get not needed
    if request.method == 'GET':
        logger.info('THIS IS THE MESSAGE')
        return HttpResponse('get is up')

    if request.method == 'POST':
        print(request.body)
        #to parse the xml payload
        notification=recurly.objects_for_push_notification(request.body)
        if notification['type']=='expired_subscription_notification':
            #this is an expired susbscription notification, get user account
            #your buisness logic
        logger.info('log for %s' % notification['type'])
        return HttpResponse('success')
{% endhighlight %}

You can remove the `@allow_by_ip` decorator if you dont want authentication by ip.Hope you liked the post

some refrences:
[recurly docs 1](https://recurly.readme.io/v2.0/page/webhooks),
[recurly docs 2](https://docs.recurly.com/push-notifications),
[recurly docs 3](https://dev.recurly.com/page/python).
