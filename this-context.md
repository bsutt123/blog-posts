# Understanding `this` in React

When I first started learning React, I was always appalled that for some reason React didn't understand that I had to use `this` everywhere. I had to bind `this` when I called a function, I had to use `this.state`, `this.props`, `this.handleClick`; you name it, you had to consider `this` and what it meant in the context of whatever you did.

At first, I thought this was because React had a bad design. In reality, its just a function of the way that javascript has designed classes, and for React to get rid of it they would have to both do a ton of work, and also remove flexibility from the framework. So here is my attempt at explaining `this`, but hopefully with the context of how to use it within React.

## Preface: Don't just read this post

I stand on the shoulders of giants in terms of my understanding of javascript, scope, and `this`. I highly recommend that in addition to playing with these concepts yourselves in REPLs and your code base, explore the learning options out there. I personally have learned a lot of this material on [Frontend Masters](https://frontendmasters.com/) with a particular shoutout for [Kyle Simpson](https://github.com/getify) and [Will Sentance](http://willsentance.com/) for making absolutely phenomenal courses that were not only fun to watch and participate in but were immensely helpful in solidifying foundation knowledge of javascript.

## Why does `this` exist in the first place?

So as you might expect, before you can under `this` in React, you have to understand why `this` exists in vanilla javascript as well. 

Javascript has what is called "static" or "lexical" scope for its variables. This means that when javascript gets compiled, it bases what variables a block of code has access to based on where it is defined in the code. This makes sense, because "lexical" just means "written", so it matters very much where a variable is defined.

In Javascript, you don't just have access to variables in your functions local lexical scope, you also have access to variables up the "lexical scope chain". Thats just a fancy way of saying that if you define a function `fn2` within a function `fn1`, then `fn2` will be able to use any variable defined in it own local variable environment _and_ anything in `fn1`'s varibale environment.

```javascript
function fn1() {
  var foo = 'foo';
  var bar = 'bar';
  function fn2() {
   var bar = 'bar';
   var bam = 'bam';
  }
}
```

In this contrived example, `fn2` would be able to access all 4 variables at run time, but `fn1` would only be able to access `foo` and `bar`.

So this is great if you know what variable you want to use at compile time, but what about if you don't? What if you don't know what Object is going to be called, or you aren't sure exactly what the function will be attached to. Maybe you want to write a method that will reference data on the object that the method is defined on. In this case you need to _dynamically_ determine the scope at execution time, and for that, you are going to need `this`.

## What is 'this'

Even MDN, the holy bible of javascript definitions, doesn't provide a good definition for what `this` is, because it is so dependent on what and how you are using it. This is my best attempt at defining what `this` is...

> `this` in javascript will by default reference the object which invoked the function

That doesn't sound so hard, except when you consider that sometimes you can use `bind` to make it bound to a new function, you can use arrow functions that won't define their own this context, you can use `call` or `apply` to change how `this` interacts with your code. Ultimately, it can be quite tricky to figure out what `this` actually is.

## How do I use 'this'

The simplest way to use `this` is to reference some kind of property on an object inside a of _function_ defined on that object. Lets consider the an example Student named Brady...

```javascript
var Brady = {
  name: 'Brady',
  grades: ['A','B',...],
  changeName: function(newName) {
    this.name = newName;
  },
}
```

in this example, our `changeName` property is a function that changes the name of the Student object to be a new name.

```javascript
Brady.name; //return Brady
Brady.changeName("Bill");
Brady.name; //return Bill 
```
Great! I have a Brady Object, but that would mean that I need to manually create the all my students, which sounds like a huge pain. But when we do the same thing all the time in javascript, we get to abstract that as a function! That means that we can make a function that generates Student Objects. Amazing!

## How does this all relate to classes

So for a long time, javascript didn't have classes. But because javascript is a prototypal language (or objects can inherit methods and properties from other objects), you could make something that looked very much like a class using the objects and function. Lets see how we might make a Student using these principals...

```javascript
var studentFunctions = {
  changeName: function(newName) {
    this.name = newName;
  },
}

function studentCreator(name, grades) {
  var newStudent = Object.create(studentFunctions);
  newStudent.name = name;
  newStudent.grades = grades;
  return newStudent;
}

var brady = studentCreator('Brady', ['A','B']);
var bill = studentCreator('Bill', []);
...

```

We have done it! We have defined a function that "creates" students. It attaches the functions we want on every student to a new Object with `Object.create`. It then attaches the properties that will define that particular student, and returns the newly created student from the function. 

But here is where I think that Javascript went a little off the rails. I think that even though this might take a bit more code to write, what is happening is obvious. When you call `studentCreator`, you are making a new object with the properties of studentFunctions, and assigning the inputs to properties on that object, and then returning it. But developers are by nature lazy, so they wanted to automate this. Enter the `new` keyword, perhaps the most misunderstood word in all of javascript.

`new` will do 4 things...

1. Create a brand new Object
2. Create a link between that new Objects prototype and the prototype of the function that called it
3. Execute the function and pass the newly created Object into the function as the `this` context
4. If the function doesn't return anything, returns `this` from the function

That lets you take our above code and make it into something that looks like this...

```javascript
function Student(name, grades) {
  this.name = name;
  this.grades = grades;
}

Student.prototype.changeName = function(newName) {
  this.name = newName;
}

var brady = new Student('Brady',['A','B'])
var bill = new Student('Bill', [])

```

Fewer lines of code? Absolutely.
Easier to understand? I don't think so.

But that is essentially how javascript creates classes, which are defined by properties and methods on an Object. When you are using `this` inside of these classes, you are ultimately trying to reference the `this` that is that object. You want to get to the variable `state` you put on the object and get it with `this.state`. You want to access the `handleChange` you defined as a property as `this.handleChange`. And most importantly _inside_ those functions you _need_ `this` to reference that class Object so that you can do all the wonderful things you want to do with your React Components

### But I'm using classes, not Objects!

Well I have some sad news for you. In Javascript, `class` is mostly just syntactic sugar for using the exact same notation that I specified above. You could also write a student class like this...

```javascript
class Student {
  constructor(name, grades) {
    this.name = name;
    this.grades = grades;
  }
  
  changeName(newName) {
    this.name = newName;
  }
}

```

classes have a couple extra bits (like the `super` keyword) but for most intents and purposes, they are the exact same thing. This syntax is way easier to write, and has a ton of advantages. But its largest disadvantage is that its very easy to create a class in javascript and have _no_ idea how it works under the hood, which leads to misunderstanding and bugs down the line.

## When are you gonna talk about React already!

I'm getting there! The problem is that to under `this` in React, you have to understand `this` in plain old javascript. 

A long time ago, React added functionality to use the native ES6 class to make a React component. For better or for worse they chose to not [auto-bind this](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#autobinding), and instead implemented classes the same as ES6 would.

So largely, it is on you to bind the right this context to your functions so that they can access the `this` that has all the properties like `setState`, `state`, and anything else you define on the class object. If you _don't_ then when you go update the state, or change anything on `this`, your function will fail as `this` will be undefined.

## How to bind `this` to your functions

One of the things that I don't like about React is just how _many_ different ways there are to bind `this`. I think that it leads to confusion about what is happening and where the `this` binding is actually occuring. I'm going to try and detail all the ways that I know how to bind this, and hopefully that should give you a good idea of how you might see it happening.

But just as important as examples are understanding the _tools_ that are used to bind `this`, and that means that we need to talk about regular functions, arrow functions and the `bind` keyword...

### Aside: Arrow Functions

ES6 made a cool new syntax for functions that looks something like this.

```javascript
var myCoolFunction = () => {
...
}
```
Its called an arrow function. In this case, that arrow function mostly equivalent to this function expression.
```javascript
var myCoolFunction = function() {
...
}
```
You have probably seen arrow functions all around town, and all the hip cool kids are telling you that its so last season to use the `function` keyword. I'm not here to discuss the merits of each one, but I do want you the critical difference between these two.

> Regular function declarations and expressions naturally make a `this` context that references the functions `this` by default. Arrow functions _do_ _not_ create a `this` context inside the function and instead inherit their `this` context from the functions parents lexical scope.

I know that sounds confusing, but it basically boils down to the fact that if you define an arrow function, `this` won't refer to the object calling the function, but instead to the `this` where that function was _defined_. This can be good or bad, depending on what you want to do, but for our purposes, arrows functions will be very useful in React because it means that instead of having to `bind(this)` to a function, we can instead let the function inherit its `this` from the parent (which is the class)

### `bind` versus Arrow functions

So are `bind(this)` and an arrow function doing the same thing? 

Yes and No.

Functionally, it ends up being equivalent, because ultimately it means that `this` is defined by what `this` is in the parent. In reality, `bind(this)` actually takes `this` and defines it inside the function, while an arrow function doesn't _have_ a `this` inside it, so javascript goes up to the next level (remember the lexical scope chain?) and uses the `this` of the parent instead.

## Binding this in React

Here is a brief summary of how I have seen people commonly bind this...

My personal choice when I can is to actually bind this inside the constructor of the function. I like it because it means that I am being very explicit about what is happening. I am taking this function that I am defining inside the constructor and I am making sure that I know _exactly_ what `this` will be...
```javascript
class Button extends React.Component {
  constructor() {
    ...
    this.toggle = this.toggle.bind(this);
  }

  toggle() {
  ...Some dope logic using this
  }
  
  render() {
    ...type up some sweet sweet JSX
    <RandomThing toggleProp={this.toggle} />
  } 
}
```
I would argue that this is the only _true_ way to "bind" `this` to your function. But using our good friend the arrow function, we can make sure that `this` is what we want it to be...
```javascript
class Button extends React.Component {
  constructor() {
    ...
  }
  
  toggle = () => {
    ...Some dope logic using this
  }
  
  render() {
    ...type up some sweet sweet JSX
    <RandomThing toggleProp={this.toggle}
  }
}
```
What am I doing here? I am defining a property `toggle` as an anonymous arrow function, so when I call `this.toggle` inside of my component, it gets to inherit its `this` context from the parent and I'll be able to access `this.state` and other necessities. Yippee!

Also possible but perhaps my least favorite way, you can also use an anonymous arrow function inside of the `{}` you define for the (likely) prop. For example...
```javascript
class Button extends React.Component {
  constructor()  {
    ...
  }
  
  toggle() {
    ...Some dope logic using this
  }
  
  render() {
    ...type up some sweet sweet JSX
    <RandomThing toggleProp={(...args) => this.toggle(...args)}
  }
}
```
This is my least favorite, because now you have defined the prop as an anonymous arrow function that when executes calls _another_ function, and passes in the props (or not) and keeps track of the event object if necessary. I think that this method feels very convoluted, but you will see it in others people's code, and there are times that I use it myself when it makes the code easier to reason about and understand.

You shouldn't depend on any single version of `this` binding in React. I think that each one of these can have advantages and disadvantages, and while you might find that you settle into one over the other, they all will have their time to shine. Your goal is to make sure that you code is readable in addition to functional, so always be considering how much your future readers (and self!) will have to understand at one time to comprehend your code.

## What now?

So thats my best introduction to how I handle binding `this` inside of React Components. With that being said, there isn't anything that can replace practice and playing with the code yourself. But I think that having a good fundamental understanding of scope, classes and `this` in javascript can only help you on your journey to being a better React Developer!

## I'm not perfect

I am trying to condense knowledge that I have learned from hours of videos and practice into a 5-10 minute read. I will make mistakes. I might call something by the wrong name, or misunderstand why something is the way it is. Just let me know if the comments, and I'll get it fixed as soon as I can.
