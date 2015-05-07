---
layout: post
category: articles
analytics: true
tags: [screen, Debian, shell, python]
comments: true
share: true
title: "Screen command Ubuntu"
date: 2015-05-07T13:58:39+05:30
---

Many times we want to run a bash or a python script on a remote terminal  and want it to continue to run even if we close the terminal. There is a simple way to do it with the help of 'screen' command.

How to start a script as screen
-------------------------------
* ssh into your remote machine
* `screen -L` (-L to show ouput on the terminal) you will see some text hit enter again Now you can enter your command here e.g
* `bash myscript.sh` and then close the window by clicking on the cross mark on the topleft corner of the window

How to check the progress
-------------------------
* now ssh again into your remote machine
* `screen -ls` will list all the screens 
* `screen -r <screen-id>` will move into that screen and show us the output. This will help you keep track of what is happening with the script


