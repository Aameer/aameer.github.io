---
layout: post
analytics: true
tags: [Celery, Task queue, distributed message passing, Ubuntu, Django, Python, flower]
comments: true
share: true
title: "Installing rabbitmq on webfaction"
date: 2016-04-02T00:37:05+05:30
---

Erlang is needed to install rabbitmq on webfaction. First create a custom app erlang on webfaction listneing to some port (say 25090)
`wget http://www.erlang.org/download/otp_src_18.3.tar.gz`
`gunzip -c otp_src_18.3.tar.gz | tar xf -`
cd into the directory (cd `otp_src_18.3` im my case)
`./configure --prefix=/home/username/`
`make`
`make install`
`epmd -port 25090 -daemon`

install rabbitmq:
First create a custom app rabbitmq on webfaction listneing to some port (in my case say  22136)
`wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.2.4/rabbitmq-server-generic-unix-3.2.4.tar.gz`
`gunzip -c rabbitmq-server-generic-unix-3.2.4.tar.gz | tar xf -`
`pwd`
`/home/username/webapps/rabbitmq`
`ln -s  $PWD/rabbitmq_server-3.2.4 ~/lib/erlang/lib/`
`vi rabbitmq_server-3.2.4/sbin/rabbitmq-defaults`

replace this:
`#CONFIG_FILE=${SYS_PREFIX}/etc/rabbitmq/rabbitmq`
`#LOG_BASE=${SYS_PREFIX}/var/log/rabbitmq`
`#MNESIA_BASE=${SYS_PREFIX}/var/lib/rabbitmq/mnesia`

with this:
`CONFIG_FILE=/home/username/webapps/rabbitmq/rabbitmq_server-3.3.5/sbin/`
`LOG_BASE=/home/username/logs/user/rabbitmq`
`MNESIA_BASE=/home/username/webapps/rabbitmq/rabbitmq_server-3.3.5/sbin/`

open this
`vi rabbitmq_server-3.2.4/sbin/rabbitmq-env`

and add this at end:
`export ERL_EPMD_PORT=25090`
`export RABBITMQ_NODE_PORT=22136`
`export ERL_INETRC=$HOME/.erl_inetrc`

also add these file with related content
`vi $HOME/hosts`
`127.0.0.1 localhost.localdomain localhost`
`::1      localhost6.localdomain6 localhost6`
`127.0.0.1 webfaction-server-address wefaction-server-address.webfaction.com`

`vi $HOME/.erl_inetrc`
`{hosts_file, "/home/username/hosts"}.`
`{lookup, [file,native]}.`

`./rabbitmq_server-3.2.4/sbin/rabbitmq-server -detached`
`./rabbitmq_server-3.2.4/sbin/rabbitmqctl status`

this post was heavily influenced from:
[ref1](http://www.markliu.me/2011/sep/29/django-celery-on-webfaction-using-rabbitmq/)
[ref2](https://community.webfaction.com/questions/17426/tutorial-django-celery-rabbitmq-virtualenv)
