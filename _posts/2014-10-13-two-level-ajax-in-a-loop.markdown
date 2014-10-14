---
layout: post
title: "Two Level Ajax in a Loop"
description: "Wait for ajax response inside another ajax iterated by a loop"
modified: 2014-10-14
category: articles
analytics: true
tags: [javascript, ajax, Namespaces in JS]
comments: true
share: true
date: 2014-10-13T20:33:36+05:30
---
Recently I faced an issue which can be closely described as using *AJAX* `ajaxOptsAjaxFirst` inside sucess function of another *AJAX* `ajaxOptsAjaxSecond` which is iterated by a loop. If you are familiar with ajax then skip the next paragraph.

For those of you who are not familiar with ajax, let me try to briefly explain *AJAX*. *AJAX* stands for Asynchonous Javascript. As it's name suggests, it is used to send to and retrieve data from a server asynchronously (in the background) without interfering with the display and behavior of the existing page. See how to use ajax with jquery [here](http://api.jquery.com/jquery.ajax/).

I was runnig a javascript which called a function `element_class_clicked` in loop for all the elements present on DOM (See code Below).

Also a refrence has to be passed to this function. As the code demands to call two ajax calls one after the other was done, so with the help of call backs we did that. The `ajaxOptsAjaxFirst` call was internally (`some_fn`) calling the `ajaxOptsAjaxSecond` and all of this was done inside namespaces. Code with issues is as under:

{% highlight javasript linenos %}
var NAMESPACE ={
  SubpartNamespace : {  
    element_class_clicked : function(clicked_element){
       var form_vals = NAMESPACE.SubpartNamespace.get_apparel_details(jQuery(clicked_element).children('.class1'));
       ajaxOptsAjaxSecond = {
       	xhrFields: {withCredentials: true},
       	url: "http://"+dressy_server+"/link1/",
       	dataType: 'json',
       	crossDomain: true,
       	data: form_vals,
       	success: function(resp){
       		alert(resp)
       	  //some code
       	},
       };
       var ajaxOptsAjaxFirst =  {
       	url: "http://"+dressy_server+"/link2/",
       	dataType: 'json',
       	crossDomain: true,
       	data: form_vals,
       	success: function(resp){
       	  if(resp.ready === true){
       	    //some more code
       	    NAMESPACE.SubpartNamespace.some_fn(some_data);
       	  } 
       	  else{
       	    console.log('Resp not ready for action');
       	  }
       	},
       };
       loginAjaxOpts = {
       	url: "http://"+server_name+"/login_link/",
       	crossDomain: true,
       	xhrFields: {withCredentials: true},
       	success: function(resp) {
       	  //console.log(resp);
       	  if (resp.status === "authenticated") {
       	      //some code
       	      jQuery.ajax(ajaxOptsAjaxFirst);
       	  } else {
       	      console.log("Please login first");
       	      
       	  }
         },
       };
       jQuery.ajax(loginAjaxOpts);
    },
    some_fn : function(some_data){
      //some more code
      jQuery.ajax(ajaxOptsAjaxSecond);	
      //some more code
    },
    toggle_fn : function(eventObj){
      eventObj.preventDefault();
      if (AUTH_STATUS) {
         var this_item = this.parentElement.parentElement; 
         if(jQuery(this).parent().siblings('canvas').is(':visible') == false){
           NAMESPACE.SubpartNamespace.element_class_clicked(this_item);
         }else{
           NAMESPACE.SubpartNamespace.element_other_class_clicked(this_item);
         };
      } else {
         console.log("Please log in first");
      } 
    },
  },
};

window.onload= function() {
  $('.element\_class').on('click', NAMESPACE.SubpartNamespace.toggle\_fn);   
  $('.'+someClass+'').click(function(){ 
    a=$('.element_class');
    $.each(a,function(c){
      if($(a[c]).parent().siblings('canvas').size()<=0){
         $(a[c]).trigger('click');
         //trigger click in this case will aslo call element_class_clicked(this_item) with same args
      }else{
        var this_item = this.parentElement.parentElement;
        NAMESPACE.SubpartNamespace.element_class_clicked(this_item);
      }    
    }); 
  });
};
{% endhighlight %}

Here the problem was that sometimes on slower servers or clients on slower networks, before we recieved the response for `ajaxOptsAjaxSecond`, the code would proceed ahead and thus give wrong results, as it would was loose the context for `clicked_element` for that paticular response.

I tried to use many solutions and hacks, like setimeout , many other solutions mentioned on [stackoverflow](www.stackoverflow.com) but none of them seemed to work for my problem. Then I solved it with by having simple abstraction for the `ajaxOptsAjaxFirst`. Code is as under:

{% highlight javascript linenos %}
var NAMESPACE ={
  SubpartNamespace : {  
    seperate_ajax_fn: function(canvas, form_vals){
      ajaxOptsAjaxSecond = {
        xhrFields: {withCredentials: true},
        url: "http://"+dressy_server+"/link1/",
        dataType: 'json',
        crossDomain: true,
        data: form_vals,
        success: function(resp){
          //some code
          alert(resp);
        },
      };
      jQuery.ajax(ajaxOptsAjaxSecond);
    },
    element_class_clicked : function(clicked_element){
      var form_vals = NAMESPACE.SubpartNamespace.get_apparel_details(jQuery(clicked_element).children('.class1'));
      var ajaxOptsAjaxFirst =  {
        url: "http://"+dressy_server+"/link2/",
        dataType: 'json',
        crossDomain: true,
        data: form_vals,
        success: function(resp){
          if(resp.ready === true){
            //some more code
            NAMESPACE.SubpartNamespace.some_fn(some_data);
          } 
          else{
            console.log('Resp not ready for action');
          }
        },
      };
      loginAjaxOpts = {
        url: "http://"+server_name+"/login_link/",
        crossDomain: true,
        xhrFields: {withCredentials: true},
        success: function(resp) {
          //console.log(resp);
          if (resp.status === "authenticated") {
            //some code
            jQuery.ajax(ajaxOptsAjaxFirst);
          } else {
            console.log("Please login first");
          }
        },
      };
      jQuery.ajax(loginAjaxOpts);
    },
    some_fn : function(some_data){
      //some more code
      NAMESPACE.SubpartNamespace.seperate_ajax_fn(canvas, form_vals);
      //some more code
    },
    toggle_fn : function(eventObj){
      eventObj.preventDefault();
      if (AUTH_STATUS) {
        var this_item = this.parentElement.parentElement; 
        if(jQuery(this).parent().siblings('canvas').is(':visible') == false){
          NAMESPACE.SubpartNamespace.element_class_clicked(this_item);
        }else{
          NAMESPACE.SubpartNamespace.element_other_class_clicked(this_item);
        };
      } else {
        console.log("Please log in first");
      } 
    },
  },
};
{% endhighlight %}

What it does is, it creates a stack for the `seperate_ajax_fn` and thus saves the context for arguments.Thus this solution will even work for slower networks as it will wait for the response for `ajaxOptsAjaxFirst` and pass corresponsing arguments which it had saved in stack. Here with just some restucturing of code in the Namespace we got the desired result.

Hope this helps someone. If you have any other approach which you think is preferable (easy to use/efficient) than above mentioned one then please do share. Thanks for reading the post. Happy coding... 
