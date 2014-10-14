---
layout: post
title: "Two Level Ajax in a Loop"
description: "Wait for ajax response inside another ajax iterated by a loop"
modified: 2014-10-14
category: articles
analytics: true
tags: [javascript, ajax]
comments: true
share: true
date: 2014-10-13T20:33:36+05:30
---



Recently I faced an issus which can be closely described as Ajax inside an AJAX which is iterated by a loop.If you are familiar with ajax then skip the next paragraph.

For those of you who are not familiar with ajax, let me try to briefly explain ajax. Ajax stands for Asynchonous Javascript. As it's name suggestes, it is used to send to and retrieve data from a server asynchronously (in the background) without interfering with the display and behavior ofthe existing page. See how to use ajax with jquery [here is the link for jquery docs](http://api.jquery.com/jquery.ajax/)

I was runnig a javascript which called a function(element-class-clicked) in loop for all the elements present on Dom.Also a refrence has to be passed to this function. As the code demands to call two ajax calls one after the other was done, so with the help of call backs we did that.The ajaxOptsAjaxFirst call was internally calling the ajaxOptsAjaxSecond and all of this was done inside namespaces. Code with issues:

`
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
	$('.element_class').on('click', NAMESPACE.SubpartNamespace.toggle_fn);   

  	$('.'+some_class+'').click(function(){ 
	     a=$('.element_class');
	     $.each(a,function(c){
	        if($(a[c]).parent().siblings('canvas').size()<=0){
	            $(a[c]).trigger('click'); //trigger click in this case will aslo call element_class_clicked(this_item) with same args
	        }else{
	            var this_item = this.parentElement.parentElement;
	            NAMESPACE.SubpartNamespace.element_class_clicked(this_item);
	        }    
	     }); 
    });
};
`
Here the problem was that sometimes on slower servers or clients on slower networks before we have recieved the response 
for ajaxOptsAjaxSecond, the code would proceed ahead and thus give wrong results as it would was loose the context for clicked-element 
for that paticular response

I tried to use many solutions and hacks, like setimeout , many other solutions from stackoverflow and diferent blogs but none of them seemed
worked for our problem Then I solved it with by having simple abstraction for the ajaxOptsAjaxFirst the solution is mentioned below:

`
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
`
What it does is actually creates stack for the seperate-ajax-fn and thus saves the context for arguments.Thus this solution will
even work for slower networks as it will wait for the response for (ajax link1).The most important thing here is that we eliminated the
use settimeouts.Moreoever with just restucturing of code in Namespace we got the desiered results

Hope this helps someone out there wandering for a solution in sea of answers.If you have any other approach which you think can work,
preferabbly which is better or easier to implement than that mentioned above , please do share.Thanks for reading the post. Happy coding... 
