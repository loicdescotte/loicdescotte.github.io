---
layout: post
title: JavaScript for Scala Developers
tags:
 - Scala
 - JavaScript
 - ECMAScript
 - Typescript
---

Disclaimer : this post is not about ScalaJS, but about the JavaScript (+ Typescript) language.
As you may know, the last published Javascript standard version is ECMAScript 2016 (or ECMAScript 7, or ES7) standard. Since ES6 (the previous standard, published in 2015), Js has a lot new syntaxic possibilities. Many of this features will make feel Scala developers at home, at least a lot more than previous JS versions.


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
Note : `let` is aimed to replace JS `var`, as it defines block scoped variables (link).

### Define a constant (a value)

Scala :

```scala
val x = 1
```

JS :
```javascript
const x = 1
```

Both of this values are not reassignable. In JS you will have an "invalid assignment" error if you try to redefine const.

Note : like `let`, `const` is block scoped.

### functions

```javascript

// old school JS function
const f = function(x) {
  return x +1
};

// Or arrow functions
const f = (x => x +1);
```

The second syntax is closer to Scala syntax, and is nice when you need to pass a function argument to another function.


### Classes

Since ES6, it is possible to defines classes in Javascript.

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

### Destructuring

```javascript

//define x and y coordinates from a point :
let { x, y } = new Point(1,2);
```

Note : unlike Scala, JS asks you to use the same parameter names in the left expression as in your object


Note 2 : you can also destructure arrays and iterables

### imports

In lib.js :
```javascript
//------ li.js ------
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

### for-in

```javascript
const object = {a:1, b:2, c:3};

for (var prop in obj) {
  console.log("obj." + prop + " = " + obj[prop]);
}
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

### varargs

```javascript
function(a, b, ...parameters) {
  // ...
}
```

`parameters` is an array

usage :

## Promises and futures


```javascript
//kind of map
let result = Promise.resolve("Success")
.then((value) =>"result :" + value);
//Promise { <state>: "fulfilled", <value>: "result :Success" }

//kind of flatmap
let result2 = Promise.resolve("Success")
 .then((value) => Promise.resolve("result " + value + " Success 2"));
//Promise { <state>: "fulfilled", <value>: "result Success Success 2" }

//print result :
Promise.resolve("Success")
.then((value) => Promise.resolve("result " + value + " Success 3"))
.then((value) => console.log(value));
```


## Where are my types ?

Typescript intro
Default angular 2 language

```javascript
let result: Promise<string> = Promise.resolve("Success")
.then((value) =>"result :" + value);

//kind of flatmap
let result2: Promise<string>  = Promise.resolve("Success")
.then((value) => Promise.resolve("result " + value + " Success 2"));
```


You can notice that there is an autoflatten (explain as the this is still Promise<string> .


Typescript compatible with JS syntax (sur ensemble)

Note : Babel for both ES 2015 and Typescript

## Immutable data

We have const to define values, but

Immutable js

React and Redux, Angular 2

### ES proposal : rest/spread properties

[This proposal](https://github.com/sebmarkbage/ecmascript-rest-spread) would be very conveniant to copy objects.

```scala
case class Person(name: String, age: Int)

val bob = Person("bob", 30)
val john = bob.copy(name="john")
```

```javascript
const bob = {name: "bob", age: 30};
var obj1 = {...bob, name: "john", address: "25 5th street NYC"}; // copy bob with name updated and address added
```

Ant it can already be used with Babel!

## More advanced examples

### extensions methods

```javascript

let sum = (x, y) => x.sum(y);

//extension
Number.prototype.sum = function (x) { return this + x };

console.log(sum(1, 2)); //3
```

With typescript (explain) :

```javascript

interface CanSum {
    sum(x: CanSum): Number;
}

let sum = (x: CanSum, y: CanSum) => x.sum(y);

//extension
Number.prototype.sum = function (x) { return this + x };

console.log(sum(1, 2)); //3
```

(kind of) typeclasses

/*not recursive*/
extension method + structural/duck typing

 type alias


 imutable collections
