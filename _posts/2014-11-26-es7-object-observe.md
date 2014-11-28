---
layout: post
title: Playing with ES7 Object.observe
tags:
 - ES7
 - JavaScript
 - Object.observe
---

I'm really not a JavaScript expert but today I wanted to try the new Object.observe API that will be part of ECMAScript 7 and that is already supported in Chrome.  
In my example I want to observe an array of names, and update an HTML list according to the events occurring on this array (e.g. add or delete). To achieve this, the appropriate function is Array.observe.

{% highlight javascript %}
Array.observe(users, function(changes){
    changes.forEach(function(change) {  
        if(change.type == "splice"){
            if(change.addedCount){
                var index = change.index;
                var user = users[index];
                $('#list ul').append('<li>...</li>') //add element in markup
            }
            else{
                var id = change.removed[0].id;
                $('li[id='+id+']').remove();
            }
        }
    });  
}); 
{% endhighlight %}

This is very simple and handy, even for a backend developper like me to automatically udpate the view from the model.  
Of course, it is also possible to have 2 ways binding, for example we can add an element in the array using an input field. Then the list will be automatically updated, thanks to the array observer.

{% highlight javascript %}
$("#ok").click(function(){
    lastId +=1;
    var name = $("#in").val();
    var user = {name: name,id: lastId};
    users.push(user);
});
{% endhighlight %}

We can also delete elements in the HTML list : 

{% highlight javascript %}
$("#list").on("click", ".del", function(event) {   
    var id = parseInt($(this).parent().attr('id'),10);
    var index = users.map(function(x) {return x.id; }).indexOf(id);
    users.splice(index, 1);
});
{% endhighlight %}

[See this JsFiddle for the full example](http://jsfiddle.net/7abdtp8r/)
