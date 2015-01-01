---
layout: post
title: "Django Allauth Integration Instagram as a Provider"
description: "Django allauth integration , instagram as provider"
modified: 2015-1-1
category: articles
analytics: true
tags: [Instagram API, Django, django-allauth, authentication]
comments: true
share: true
date: 2015-01-01T20:50:19+05:30
---

[Django alluth](https://github.com/pennersr/django-allauth) is an Integrated set of Django applications addressing authentication, registration, account management as well as 3rd party (social) account authentication. There is great deal of documentation and examples available out there about integrating it with different providers but for instagram (although similar to others) there wasn't a detailed stepwise guide, so I thought a post could save someone some time.

Most of what I have mentioned here is already present in the official docs [here](http://django-allauth.readthedocs.org/en/latest/installation.html), but it still may help someone to get the things up and running quickly.

Installation
------------
`pip install django-allauth`

settings.py
-----------
{% highlight python linenos %}

TEMPLATE_CONTEXT_PROCESSORS = (
    ...
    # Required by allauth template tags
    "django.core.context_processors.request",
    ...
    # allauth specific context processors
    "allauth.account.context_processors.account",
    "allauth.socialaccount.context_processors.socialaccount",
    ...
)

AUTHENTICATION_BACKENDS = (
    ...
    # Needed to login by username in Django admin, regardless of `allauth`
    "django.contrib.auth.backends.ModelBackend",

    # `allauth` specific authentication methods, such as login by e-mail
    "allauth.account.auth_backends.AuthenticationBackend",
    ...
)

INSTALLED_APPS = (
    ...
    # The Django sites framework is required
    'django.contrib.sites',

    'allauth',
    'allauth.account',
    'allauth.socialaccount',
    # ... include the providers you want to enable:

    'allauth.socialaccount.providers.instagram',
)

SITE_ID = 1
{% endhighlight %}

urls.py
-------

{% highlight python linenos %}
urlpatterns = patterns('',
    ...
    (r'^accounts/', include('allauth.urls')),
    ...
)

{% endhighlight %}

Post-Installation
-----------------
In your Django root execute the command below to create your database tables:

`python manage.py syncdb`

Now start your server, visit your admin pages (e.g. `http://localhost:8000/admin/`) and follow these steps:

* Add a Site for your domain, matching `settings.SITE_ID` (django.contrib.sites app).
* For each OAuth based provider, add a Social App (socialaccount app).
* Fill in the site and the OAuth app credentials obtained from the provider.For details, Check the pic, where to get these details?, check this [post](http://aameer.github.io/articles/geotagged-instagram-pictures/).

![Diagram1](/images/admin.png)

* Moreover the call back url mentioned in the app on instagram should be the urtl where the user is taken after he/she is authenticated and the redirect uri in our case would be `http://localhost:8000/accounts/instagram/login/callback/`.
* After setting up everything correctly you are ready to go.for pictures please refer to this post [here](http://aameer.github.io/articles/geotagged-instagram-pictures/).

