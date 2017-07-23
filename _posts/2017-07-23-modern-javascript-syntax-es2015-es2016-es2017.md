---
layout: post
title: Modern JavaScript syntax (ES2015, ES2016, ES2017)
date: 2017-07-23 22:49:18 +0200
categories: javascript
description: |
  Newest versions of ECMAScript has become quite popular, but I noticed that
  only a few know all the features it provides. In this post, I'm going to show
  you the most important functionalities from ECMAScript. I'm skipping
  generators and for of loops on purpose because they aren't handy in our daily
  job.
last_modified_at: 2017-07-23 22:49:18 +0200
header-img: assets/images/javascript-background.jpg
---

Newest versions of ECMAScript has become quite popular, but I noticed that only
a few know all the features it provides. In this post, I'm going to show you
the most important functionalities from ECMAScript. I'm skipping generators and
for of loops on purpose because they aren't handy in our daily job.

## let and const
In classic JavaScript, we use `var` keyword to declare a local variable in the
scope of the enclosing function. Let me show you an example:

```javascript
var a = 1

if (true) {
  var b = 2
}

while (true) {
  var c = 3
  break
}

console.log('a =', a) // a = 1
console.log('b =', b) // b = 2
console.log('c =', c) // c = 3
```

As you can see, we can print all the variables even if they're declared in an
`if` or a `while` block. This behavior is unique to JavaScript's `var`. Other
programming languages limit the scope of local variables to a block, not a
function. Here comes `let` keyword. It's a more sane version of `var`. Let's
replace all the `var` declarations with `let`:

```javascript
let a = 1

if (true) {
  let b = 2
}

while (true) {
  let c = 3
  break
}

console.log('a =', a) // a = 1
// console.log('b =', b) COMPILE TIME ERROR
// console.log('c =', c) COMPILE TIME ERROR
```

Now, the variables' scope is limited to the `if` and the `while` block. `const`
is a more strict version of `let` because it doesn't allow to change the value
of the variable. Here's a small example which explains the difference:

```javascript
let a = 1
console.log('a =', a) // a = 1
a = 2
console.log('a =', a) // a = 2

const b = 1
console.log('b =', b) // b = 1
// b = 2 COMPILE TIME ERROR
```

## Exponentiation operator
The old way to calculate the power of a number is to use `Math.pow` function.
`Math` is a built-in object that contains various utilities for mathematical
operations and `pow` is its function. ES2016 brings us a more elegant way to
perform the calculation - exponentiation operator. Instead of writing
`Math.pow(2, 2)` we can write `2 ** 2`. Convenient, isn't it?

## Computed properties
Have you ever wanted to create an object with a dynamic property and return it?
I've met the problem multiple times and here's what I did:

```javascript
const object = {}
object[dynamicName] = someValue
return object
```

![mother of god meme](/assets/images/mother-of-god-meme.jpg)

It's terrible! We have to create a local variable and use ugly indexing
operator (`[]`) operator to set the property. Fortunately, we can use computed
property to clean it up:

```javascript
return { [dynamicName]: someValue }
```

Much better! We've got a pretty one liner.

## Template literals
String construction is a pretty common programming task, and various languages
handle it differently. C has `sprintf` function, Python has `format` function,
Ruby has string interpolation, and JavaScript has... string concatenation.
There are numerous tricks to make it a bit more acceptable like using `join`
method, but it's not how it should be done. ES2016 fills this loophole and
defines template literals. It works the same as Ruby's string interpolation.

```javascript
const user = { name: 'Peter', age: 24 }
const str1 = 'Hello. My name is ' + user.name + ". I'm " + user.age + " years old" // awful!
const str2 = ['Hello. My name is ', user.name, ". I'm ", user.age, 'years old.'] // a bit better but still hacky
const str3 = `Hello, My name is ${user.name}. I'm ${user.age} years old` // I like it!
```

Cleaner, isn't it?

## Variadic functions
Let's suppose we want to write a function with two or more arguments in
JavaScript:

```javascript
function foo(a, b) {
  console.log(a, b, Array.prototype.slice.call(arguments))
}

foo() // undefined undefined []
foo(1) // 1 undefined [1]
foo(1, 2) // 1 2 [1, 2]
foo(1, 2, 3) // 1 2 [1, 2, 3]
```

This is messy. Not only we use hacky `arguments` object, but also parameters
appear in both an argument variable and in `arguments` object. We can fix it by
providing an additional argument to `slice` method, but we have to keep in mind
that the argument is dependant on the number of positional arguments which can
lead to additional bugs. That's not how a modern programming language should
behave. ES2015 fixes that by introducing "rest parameters". Let's take a look
at it:

```javascript
function foo(a, b, ...c) {
  console.log(a, b, c)
}

foo() // undefined undefined []
foo(1) // 1 undefined []
foo(1, 2) // 1 2 []
foo(1, 2, 3) // 1 2 [3]
```

Nice and clean!

## Default parameters

Similarly to variadic functions, default parameters are a disaster in
JavaScript. To be honest, it's just one big hack.

```javascript
function foo(a, b) {
  if (typeof b === 'undefined') {
    b = 2
  }

  console.log('a =', a)
  console.log('b =', b)
}
```

The code is very cryptic, and it doesn't scale very well (what if we have three
arguments with a default value). Thankfully, ES2015 handles it very well.

```javascript
function bar(a, b = 2) {
  console.log('a =', a)
  console.log('b =', b)
}
```

![mother of god meme](/assets/images/i-like-it-a-lot-meme.gif)

## Destructuring

Sometimes, we use an object's property multiple times in a scope. Writing
`object.property` each time we want to access it isn't very DRY. To save a
couple of bytes we can use a local variable, but ECMAScript's technical
committee decided that it can be done better. That's where destructuring comes:

```javascript
const foo = { a: 1, b: 2, c: 3 }
const a = object.a
const b = object.b
const c = object.c // it's definitely shorter, but still lengthy

const { a, b, c } = object // wow!
```

It's handy and saves us much typing. However, using destructuring whenever you
want to access the property is overkill. It doesn't make the code cleaner; it
pollutes it with additional local variables. I'd suggest using destructuring
only if you use a property at least three times.

## async and await

Promises significantly improved asynchronous functions handling. Instead of an
ugly functions nesting (callback hell), we can chain promises using `then` and
gently handle errors using `catch`.

```javascript
function foo() {
  return fetch('/data.json')
    .then(response => response.json())
    .then(data => {
      if(condition(data)) {
        return fetch('/other-data.json')
      } else {
        return Promise.resolve(something)
      }
    })
    .catch(error => { handleError(error) })
}
```

Better than callback hell, but still doesn't look very natural. The code is
segmented into `then` and `catch` blocks which might be confusing for many
programmers. Can't we write asynchronous code  line by line exactly as we write
synchronous code? Actually, we can. We can use `async` and `await`. Let's
rewrite the promise code using `async` and `await`:

```javascript
async function bar() {
  try {
      const response = await fetch('/data.json')
      const data = await response.json()
      if (condition(data)) {
        return fetch('/other-data.json')
      } else {
        return Promise.resolve(something)
      }
  }
  catch(error) {
    handleError(error)
  }
}
```

Wow! Asynchronous code is written like synchronous. This is magic! The `async`
`await` syntax is so beautiful that there's no reason not to use it.

## Other syntax improvements

Besides the ECMAScript specification, there're other language improvements like
JSX or Flow.

JSX is React's extension to simplify writing components in JavaScript files.
Its syntax is a little-changed HTML:

```javascript
function Component(props) {
  return (
    <div className="component">
      <h1 className="component__heading">{props.heading}</h1>
    </div>
  )
}
```

Nothing particularly difficult, but makes life much easier.

Flow is Facebook's static type checker for JavaScript. It allows programmers to
annotate their code with type definitions and catch errors at the compile time.
I haven't used it yet, but the project is exciting, and I'll try it in the
nearest future. You can found more information here.

As you have probably noticed, I didn't write a single word about classes and
arrow functions. These two features are much more complicated than the listed
one and I'd rather write more detailed posts about them to cover all the
functionalities. If you'd like to try these features by yourself, you can clone
my repository and play with it. Don't forget to leave a star. :)
