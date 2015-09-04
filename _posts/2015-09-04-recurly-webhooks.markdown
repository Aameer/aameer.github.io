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
mentioned under:

{% highlight python linenos %}
from django.http import HttpResponse,
from django.views.decorators.csrf import csrf_exempt
import recurly

allowedIps =['127.0.0.1', '74.201.212.175', '64.74.141.175', '75.98.92.102', '74.201.212.0', '74.201.212.1', 
        '74.201.212.2', '74.201.212.3', '74.201.212.4', '74.201.212.5', '74.201.212.6', '74.201.212.7', 
        '74.201.212.8', '74.201.212.9', '74.201.212.10', '74.201.212.11', '74.201.212.12', '74.201.212.13', 
        '74.201.212.14', '74.201.212.15', '74.201.212.16', '74.201.212.17', '74.201.212.18', '74.201.212.19', 
        '74.201.212.20', '74.201.212.21', '74.201.212.22', '74.201.212.23', '64.74.141.0', '64.74.141.1',
        '64.74.141.2', '64.74.141.3', '64.74.141.4', '64.74.141.5', '64.74.141.6', '64.74.141.7', '64.74.141.8', 
        '64.74.141.9', '64.74.141.10', '64.74.141.11', '64.74.141.12', '64.74.141.13', '64.74.141.14', '64.74.141.15', 
        '64.74.141.16', '64.74.141.17', '64.74.141.18', '64.74.141.19', '64.74.141.20', '64.74.141.21', '64.74.141.22', 
        '64.74.141.23', '75.98.92.28', '75.98.92.29', '75.98.92.30', '75.98.92.31', '75.98.92.32', '75.98.92.33', 
        '75.98.92.34', '75.98.92.35', '75.98.92.36', '75.98.92.37', '75.98.92.38', '75.98.92.39', '75.98.92.40', 
        '75.98.92.41', '75.98.92.42', '75.98.92.43', '75.98.92.44', '75.98.92.45', '75.98.92.46', '75.98.92.47', 
        '75.98.92.48', '75.98.92.49', '75.98.92.50', '75.98.92.51', '75.98.92.52', '75.98.92.53', '75.98.92.54', 
        '75.98.92.55', '75.98.92.56', '75.98.92.57', '75.98.92.58', '75.98.92.59', '75.98.92.60', '75.98.92.61', 
        '75.98.92.62', '75.98.92.63', '75.98.92.64', '75.98.92.65', '75.98.92.66', '75.98.92.67', '75.98.92.68', 
        '75.98.92.69', '75.98.92.70', '75.98.92.71', '75.98.92.72', '75.98.92.73', '75.98.92.74', '75.98.92.75', 
        '75.98.92.76', '75.98.92.77', '75.98.92.78', '75.98.92.79', '75.98.92.80', '75.98.92.81', '75.98.92.82', 
        '75.98.92.83', '75.98.92.84', '75.98.92.85', '75.98.92.86', '75.98.92.87', '75.98.92.88', '75.98.92.89', 
        '75.98.92.90', '75.98.92.91', '75.98.92.92', '75.98.92.93', '75.98.92.94', '75.98.92.95']

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
