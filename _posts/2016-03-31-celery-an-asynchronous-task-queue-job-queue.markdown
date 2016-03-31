---
layout: post
nalytics: true
tags: [Celery, Task queue, distributed message passing, Ubuntu, Django, Python, flower]
comments: true
share: true
title: "Celery an asynchronous task queue/job queue"
date: 2016-03-31T23:48:28+05:30
---

What is Celery:
---------------
Celery is an asynchronous task queue/job queue based on distributed message passing. It is focused on real-time operation, but supports scheduling as well.The execution units, called tasks, are executed concurrently on a single or more worker servers using multiprocessing, Eventlet, or gevent. Tasks can execute asynchronously (in the background) or synchronously (wait until ready). Celery is used in production systems to process millions of tasks a day. More [here](http://www.celeryproject.org/)

What will we do:
----------------
We will build a demo celery project to get a basic understanding of it. So let us start.

make a virtualenvironment
`mkviretualenv celerydemo`

start from here:
do go through the [docs](http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html#first-steps) atleast once.As per docs in this tutorial you will learn the absolute basics of using Celery. You will learn about:

Choosing and installing a message transport (broker).
* Installing Celery and creating your first task.
* Starting the worker and calling tasks.
* Keeping track of tasks as they transition through different states, and inspecting return values.

1.chosing a broker: rabbitmq for this post
`sudo apt-get install rabbitmq-server` more about rabbitmq [here](https://www.rabbitmq.com/)
if you are installing it on webfaction or something then check this post (comming soon!)


Installing celery:
`pip install celery`

Install django and getting project ready:
`pip install Django==1.7`
`mkdir celerydemo`
`cd celerydemo`
`django-admin startproject mysite`
`pwd > ~/.virtualenvs/celerydemo/.project`
`python manage.py migrate`
`python manage.py runserver`

Although with the latest version of celery you may not need this but we will still use it to play around , you can find details [here](http://docs.celeryproject.org/en/latest/django/first-steps-with-django.html)
`pip install django-celery`
Add `djcelery` to `INSTALLED_APPS`.
`python manage.py migrate djcelery`

inside the mysite/celery.py add some code for initializing celery check file [here](https://github.com/Aameer/celerydemo/blob/master/mysite/celery.py)
Also add in mysite/__init__ some code details [here](https://github.com/Aameer/celerydemo/blob/master/mysite/__init__.py)

plus then we can have a demoapp whcih will have tasks.py file from where we can add tasks and import and call delay on them.   

run the celery:
`python manage.py celeryd -B -l info`

run the camera
`python manage.py celery events --camera=djcelery.snapshot.Camera`

run the server:
`python manage runserver`

to test out everything:
`python manage.py shell`
`from demoapp.tasks import add`
`add.delay(4,4)`

check the celery console to see the tasks

for monitoring if the tasks are not showing up on the admin panel beacause of some issue with django-celery you can use [flower](http://flower.readthedocs.org/en/latest/install.html)
`pip install flower`
to start flower:
`flower -A mysite --port=5555`
then visit:
localhost:5555
then we can see the tasks in real time on the dash flower board
note naming the task(`@task(name='demoapp.tasks.add')`) and using snapshot(`python manage.py celery events --camera=djcelery.snapshot.Camera`) together could work just fine wihthout flower.

Moreover you can demonize celery with supervisor or circus but thats topic for another day!
