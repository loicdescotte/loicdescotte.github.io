---
layout: post
title: JavaScript for Scala Developers
tags:
 - Scala
 - JavaScript
 - ECMAScript
 - Typescript
---

Disclaimer : this post is not about [Scala.js](https://www.scala-js.org/), but about the JavaScript language (+ Typescript).

As you may know, the last published JavaScript standard is ECMAScript 2016 (or ECMAScript 7, or ES7). Since ES6 (the previous standard, published in 2015), JS has a lot new syntactic possibilities. Many of this features will make Scala developers feel at home, at least a lot more than previous JS versions.
Important note : it is possible to target browsers that don't fully support ES6 or ES7 using the [Babel compiler](https://babeljs.io/).

## Basic examples

### Define a variable

Scala :
```scala
var x = 1
```

JS :
```javascript
let x = 1
```
Note : `let` is aimed at replacing JS `var`, as it defines [block scoped](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/block) variables.

### Define a constant (a value)

Scala :
```scala
val x = 1
```

JS :
```javascript
const x = 1
```

Both of this values are not reassignable. In JS you will have an "invalid assignment" error if you try to redefine a const.

Note : like `let`, `const` is block scoped.

### Functions

```javascript

// old school JS functions
const f = function(x) {
  return x +1
};

// Or arrow functions
const f = (x => x +1);
```

The second syntax is closer to Scala, and is nice when you need to pass a function argument to another function.


### Classes

Since ES6, it is possible to define classes in JavaScript.

```javascript
class Point {
       constructor(x, y) {
           this.x = x;
           this.y = y;
       }
       toString() {
           return '(' + this.x + ', ' + this.y + ')';
       }
   }

const p = new Point(1,2);
```

### Default parameters

```javascript
function multiply(a, b = 1) {
  return a * b;
}

multiply(5, 2); // 10
multiply(5, 1); // 5
multiply(5);    // 5
```

### Generators

Generators are functions that can produce a sequence of values :

```javascript
function* idMaker(index = 0, max=10){
    yield index;
    if(index < max)
      yield* idMaker(index+1);
}
let gen = idMaker();

console.log(gen.next().value); // 0
console.log(gen.next().value); // 1
console.log(gen.next().value); // 2
```

Generator functions are prefixed with a `*`.
`yield` may remind you of Scala for comprehension. This keyword is used to produce an element in the generator.
`yield*` is used to append elements from another generator.

### Destructuring

```javascript

//define x and y coordinates from a point :
let { x, y } = new Point(1,2);
```

Note : unlike Scala, JS asks you to use the same parameter names in the left expression as in your object.

#### Spread operator

This operator can be used to structure and destructure arrays.

```javascript
let [...values] = idMaker();
console.log(values); //0 to 10

let [head, ...tail] = values;
console.log(head); //0
console.log(tail); //1 to 10
```

### Rest operator

```javascript
let f = function(a, b, ...rest) {
  // ...
}
```

This operator works like Scala varargs.
`parameters` is retrieved as an Array object.

Usage :

```javascript
f("a","b", 1 ,2 ,3) //1, 2, 3 will be in the rest array
```

### Modules and imports

In lib.js :
```javascript
export function f(x) {
    return x+1;
};

// other functions ...
```

In your JS file :

```javascript
import { f, g } from 'lib';

const a = f(1);
```

## Promises

JS Promises are close to Scala Futures :

```javascript
let result = Promise.resolve("Success")
.then((value) =>"result :" + value);
//Promise { <state>: "fulfilled", <value>: "result :Success" }

//combine 2 Promises :
let result2 = Promise.resolve("Success")
 .then((value) => Promise.resolve("result " + value + " Success 2"));
//Promise { <state>: "fulfilled", <value>: "result Success Success 2" }

//print result :
Promise.resolve("Success")
.then((value) => Promise.resolve("result " + value + " Success 3"))
.then((value) => console.log(value));
```


## Where are my types ?

One of the biggest difference with Scala is the absence of types.
TypeScript is a typed superset of JavaScript, that can be compiled and transformed (transpiled) to JS.
It is the language that has been used to write the Angular framework (since Angular 2), and it is the recommended language to develop Angular applications as well.

Let's see how it looks :

```javascript
let result: Promise<string> = Promise.resolve("Success")
.then((value) =>"result :" + value);

let result2: Promise<string>  = Promise.resolve("Success")
.then((value) => Promise.resolve("result " + value + " Success 2"));
```

You can see that I've added some type definitions to my result variable. The type notation is similar to Scala type notation. `let result: Promise<string>` tells the compiler that this variable must be a Promise of string. If it's not the case, it will produce a compilation error.

Note: You can notice that there is an automatic flatten of JS Promises, as my `result2` is a `Promise<string>` and not a `Promise<Promise<string>>`. This is not specific to the TypeScript language (pure JS promises are also flatten).

### Type aliases with TypeScript

TypeScript type aliases look like Scala aliases :

```javascript
type Name = string;
type NameResolver = () => string;
```

## Immutable data

We have const to define values, but it is possible to get further with [Immutable.js](https://facebook.github.io/immutable-js/) collections.

[React](https://facebook.github.io/react/), [Redux](http://redux.js.org/) and [Angular](https://angular.io/) are frameworks that rely a lot on immutability.

### ES proposal : rest/spread properties

[This proposal](https://github.com/sebmarkbage/ecmascript-rest-spread) would be very convenient to copy objects.

Object copy in Scala (using case classes) :
```scala
case class Person(name: String, age: Int)

val bob = Person("bob", 30)
val john = bob.copy(name="john")
```
JS :
```javascript
const bob = {name: "bob", age: 30};
const obj1 = {...bob, name: "john", address: "25 5th street NYC"}; // copy bob with updated name and address added
```

And it can already be used [with Babel](https://babeljs.io/docs/plugins/transform-object-rest-spread/)!

## More advanced examples

### Extensions methods

With JS you can add methods on any class via its prototype (the concept behind JS object oriented programming):

```javascript

let add = (x, y) => x.sum(y);

//extension
Number.prototype.sum = function (x) { return this + x };

console.log(add(1, 2)); //3
```

With TypeScript you can declare that a function needs an object with a specific method (or several methods), using an interface :

```javascript

interface CanSum {
    sum(x: CanSum): Number;
}

let add = (x: CanSum, y: CanSum) => x.sum(y);

//extension
Number.prototype.sum = function (x) { return this + x };

console.log(add(1, 2)); //3
```

Note : interfaces are implicitly implemented if an object defines all methods of the interface (it is called structural subtyping).

With extension methods and structural subtyping, you can emulate Scala implicit classes and be close to type classes, but without the recursive power of implicit resolution of type class instances. For example, if I need a class P to be serializable to a format X, I can add a `serializeToX` method on it instead of defining an implicit XSerializer[P], and P could be seen as implementing a `CanBeSerialized` interface. But, if P contains other types that need to be serialized, serializers won't be discovered implicitly and recursively. You will have to define methods on each type.

## Other functional goodness

If you miss functional abstractions like Option, Either and even Free monad, take a look at [Monet.js](https://cwmyers.github.io/monet.js/), an amazing functional JS library.
As types are really important for this kind of library (Maybe/Option monad for instance is not very useful without types), you can use Monet with its [TypeScript bindings](https://github.com/cwmyers/monet.js/pull/48).
