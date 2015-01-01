---
layout: post
title: "GeoTagged Instagram Pictures"
description: "Using google maps api and instagram api"
modified: 2014-12-31
category: articles
analytics: true
tags: [Instagram API, Google Maps API, Django, django-allauth, Geotagged-Pictures]
comments: true
share: true
date: 2014-12-31T22:06:17+05:30
---

Recently I worked on a problem to fetch geotagged pictures of a place from instagram api.As the instagram api allows searches based on longitude and latiude or based on location id for details click [here](http://instagram.com/developer/endpoints/).So I broke the problem down in to two parts first was to get the latitude and longitude of a particular place which the user is searching for and second was to get the pictures from instagram.

I have used django, python, javascript, django-alluth, bootstrap for this project.

* Googles api
-------------

when the user inputs a location name or address we have to get the latitude and longitude of a place, for this I used Google maps javascript api more details click [here](https://developers.google.com/maps/documentation/javascript/).With the help of google maps api we get location information of verbose name or an address. We can call Google api from  our server side or from the client side I went with the latter as it doesn't have any limits on the number of queries and this also means less load on your server. To use Google maps api you you need to activate following service(Geocoding API) in you [Google developers account](https://code.google.com/apis/console/) and then get the key(API Access as shown in the picture below) which you have to use to query Google api, note that I am querying for well know places which are mostly well known. After you get the location information from the Googles api we can proceed to the next step.

![Diagram1](/images/google_api.png)

* instagrams api
----------------

Instagrams api works mostly like  other social networking api's. Here first we will register as instagrams developers and create and app to get app secret and client id (check picture below) and after that we can proceed forward to query api , but wait for the queries we need the user needs to be authenticated, luckily django-alluth(https://github.com/pennersr/django-allauth) comes to our rescue and saves us lot of time and effort.
To get the users authenticated we just need to configure(check [this](http://aameer.github.io/articles/django-allauth-integration-instagram-as-a-provider/) post about django allauth configuration) the allauth and then we can get the access-token which we will use in our queries to get our result dump(json format) after getting the response from instagrams servers, we parse the result to get the url of the images and then display them on the front end.For codes check my github project [here](https://github.com/Aameer/instagramGTpics).

![Diagram2](/images/instagram_api.png)

Incase you feel something can be improved do let me know. By the way I have implemented both the locations and media end point of instagram just to compare the results. The media endpoint gives lot of latest pictures(with just one call) but there may be lot of noise on the other hand locations api gives very few pictures but with considerably less noise on top of that we need two calls for the locations endpoint.


