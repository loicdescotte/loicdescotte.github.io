---
layout: post
title: Creating user interfaces with Scala.js and Binding.scala
tags:
 - Scala
 - Binding.scala
 - Scala.js
 - JavaScript
---

Binding.scala is a UI library for Scala.js, that is quite similar to React.js.

Let's see how to use this library to build a simple list of elements, with a search box and add/remove features.

Binding.scala allows to mix HTML with Scala code to dynamically generate content on the browser side. Let's create a first version of an item list component :

```scala
@dom def itemList(items: Vars[String]) = {
  <section>
    <ol>{
      for (item <- items) yield
        <li>
          { item }         
        </li>
    }</ol>
  </section>
}
```

`Vars` and `Var` types can encapsulate data and states. When a change is done in this structures, the UI is updated.

Now let's add a delete button to remove items :

```scala
@dom def itemList(items: Vars[String]) = {
  <section>
    <ol>{
      for (item <- items) yield
        <li>
          { item }         
          <button onclick={ event: Event => items.get -= item }>x</button>
        </li>
    }</ol>
  </section>
}
```

You can notice that Vars defines some operators (like `-=`) to update the state.

It's also quite easy to add a search box :

```scala
@dom def itemList(items: Vars[String]) = {

  val search: Var[String] = Var("")
  val searchInput: Input = {
    <input type="text" oninput={event: Event => search := dom.currentTarget[Input].value}/>
  }

  <section>
    <div> Search: { searchInput } </div>
    <ol>{
      //bind search value to item list
      for (item <- items if (item.startsWith(search.bind))) yield
        <li>
          { item }
          <button onclick={ event: Event => items.get -= item }>x</button>
        </li>
    }</ol>
    {add(items).bind}
  </section>
}
```

Using a simple `if` in items rendering, we can bind a filter to the search box value.

Now we would like to add new items in our list. When the UI becomes more complex, it's better to split it into several sub-components, so we will create a distinct `add` component :

```scala
@dom def add(items: Vars[String]) = {

 val addInput: Input = <input type="text"/>

 val addHandler = { event: Event =>
    if (addInput.value != "" && !items.get.contains(addInput.value)) {
      items.get += addInput.value
      addInput.value = ""
    }
  }

  <div>{ addInput } <button onclick={ addHandler }>Add</button></div>
}
```

To ensure communication between our two components, we use the `items` Vars as a parameter.
Now we can include the `add` component into our `itemList` component :

```scala
@dom def itemList(items: Vars[String]) = {

  val search: Var[String] = Var("")
  val searchInput: Input = {
    <input type="text" oninput={event: Event => search := dom.currentTarget[Input].value}/>
  }

  <section>
    <div> Search: { searchInput } </div>
    <ol>{
      for (item <- items if (item.startsWith(search.bind))) yield
        <li>
          { item }
          <button onclick={ event: Event => items.get -= item }>x</button>
        </li>
    }</ol>
    {add(items).bind}
  </section>
}
```

Finally we can initialize our components and build our application in a simple HTML page :

```scala
@dom def render() = {
  val items = Vars("bob", "joe", "john")
  <div>
    <h3>List of elements</h3>
    { itemList(items).bind }
  </div>
}

dom.render(document.body, render)
```

In conclusion, I think it's pretty easy to build and assemble reactive UI components with Binding.scala. It's also nice (and more powerful) to be able to use Scala to iterate over data, format and filter data, rather than HTML tags (à la Angular).

You can run the full example [here](https://scalafiddle.io/sf/9f4Tp47/1).

You can also compare this example with [this similar application](https://codepen.io/loicd/pen/RKJryq) written in JavaScript with React.js. The main difference is that React is using a virtual dom while Binding.scala is not.
The more direct consequence is the use of Var and Vars types to handle states and data lifecycle in Binding.scala, while React only allows to update the state of a component to update the (virtual) dom.
