---
layout: post
title: No generics in Golang, but...
category: golang
---

Everyone knows that Go is a great language but that it does not support generics (yet?).

However in a language that supports first-class functions, generic types can be very handy. 
Fortunately, the [gen](http://clipperhouse.github.io/gen/) package helps to fill this gap.
Gen is a kind of macro tool that generate new structs from your models, with collections-like functions to filter, sort, group, iterate over your data.

You just need to add a `+gen` annotation comment on your models :

{% highlight golang %} 
// +gen
type Message struct {
	author, text string
}
{% endhighlight %}

Then, after running `go get github.com/clipperhouse/gen`, you will be able to generate some Go code with the `gen` command. 
Finally, the new generated types can be used as follow, for example to filter a list of messages : 

{% highlight golang %} 
func main() {

	messages := make(Messages, 3)
	messages[0] = Message{"loic", "hi"}
	messages[1] = Message{"loic", "hello"}
	messages[2] = Message{"bob", "hi"}

	myMessages := messages.Where(func(m Message) bool {
		return m.author == "loic"
	})

	for _, message := range myMessages {
		fmt.Printf("author : %s - message : %s \n", message.author, message.text)
	}

}
{% endhighlight %}

It only remains to integrate the generation step in your build tool.
