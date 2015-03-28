---
layout: post
description: "Using virtualenv"
modified: 2015-3-28
category: articles
analytics: true
tags: [virtualenv, Debian, virtualenvwrapper, python3.4]
comments: true
share: true
title: "using virtualenv"
date: 2015-03-28T10:39:28+05:30
---

use virtualenv for your projects( instructions fot ubuntu 14.04)
---------------------------------------------------------------

Ubunut 14.04 comes preloaded with python3 check with 
`python3`

install setuptools
`sudo apt-get install python3-setuptools`

`sudo easy_install3 pip`
`sudo pip3 install virtualenv`
`sudo pip3 install virtualenvwrapper`

I am a vim fan, if you don't have vim
`sudo apt-get install vim`

vim .bashrc and add this line in the end, SETUP virtualenvwrapper for python incase default doesn't work
`export VIRTUALENVWRAPPPER_PYTHON=/usr/bin/python3.4`
`export VIRTUALENVWRAPPPER_PYTHON`

needed for virtualenvwrapper
`export WORKON_HOME=$HOME/.virtualenvs`
`export PROJECT_HOME=$HOME/projects`
`source /usr/local/bin/virtualenvwrapper.sh`

save and quit the terminal

clone any repo you want to workon from bitbucket or github
`git clone the repo-url`

make a virtualenv 
`mkvirtualenv yourvirtualenvname`


get into the diretory (note where manage.py is) so that we jump directly here after workon
`pwd > ~/.virtualenvs/yourvirtualenvname/.project`

Now you can install things in your virtualenv and next time you want to work on it just type 
`workon <yourvirtualenvname>`
