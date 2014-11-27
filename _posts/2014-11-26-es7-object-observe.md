---
layout: post
title: Playing with ES7 Object.observe
tags:
 - ES7
 - JavaScript
 - Object.observe
---

I'm really not a JavaScript expert but today I wanted to try the new Object.observe API that will be part of ECMAScript 7 and that is already supported in Chrome.  
In my example I want to observe an array of names, and update an HTML list according to the events occurring on this array (e.g. add or delete).

{% highlight javascript %}
var names = ['joe', 'bob']
Object.observe(names, function(changes){
    changes.forEach(function(change) {        
        var index = change.name;
        if(change.type == "add"){
           var elem = names[index];
           $('#list ul').append('<li>...</li>') //add element in markup
        }
        //...
    });  
}); 
{% endhighlight %}

This is very simple and handy, even for a backend developper like me to automatically udpate the view from the model.  
Of course, it is also possible to have 2 ways binding, for example we can add an element in the array using an input field. Then the list will be automatically updated, thanks to the array observer.

{% highlight javascript %}
$("#ok").click(function(){
    names.push($("#in").val());
});
{% endhighlight %}

[See this JsFiddle for the full example](http://jsfiddle.net/j2wdns09/)
