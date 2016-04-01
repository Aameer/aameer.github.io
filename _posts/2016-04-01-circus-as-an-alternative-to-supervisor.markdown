---
layout: post
analytics: true
tags: [Celery, Task queue, distributed message passing, Ubuntu, Django, Python, flower]
comments: true
share: true
title: "circus as an alternative to supervisor"
date: 2016-04-01T00:24:31+05:30
---

Using [circus](rcus.readthedocs.org) as an alternative to supervisor for python3 as supervisor (less than version 4) doesnt support python 3 and  If are not using virtualenvs and are using python3 in your apps then you may want to have a look below. Our objective in this post is to daemonize celery with the help of circus which is an alternative for its widely used counterpart supervisor

install circus:
`pip3.4 install circus`
`pip3.4 install circus-web`
`pip3.4 install chaussette`

then create a `circus.ini` file with following contents:

{% highlight python linenos %}
[watcher:celery]
cmd = full_path/python3.4 full_path/manage.py celeryd -B -l info

[watcher:celerycamera]
cmd = full_path/python3.4 full_path/manage.py celery events --camera=djcelery.snapshot.Camera

[watcher:dceleryflower]
cmd = full_path/python3.4 full_path/manage.py celery flower -A indeev5 --basic_auth=username:password --port=5555

{% endhighlight %}

to run this enter:
`circusd --daemon --log-level=ERROR circus.ini`

to check the status enter:
`circusctl`

to stop it enter:
`circusctl quit`

