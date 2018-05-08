# Async Abstractions in Vanilla JS

In todays frontend web development world, you often will run into an issue thats been solved by someone else with a module. Whether its dealing with asyncronous javascript behavior or package css images into sprites, the ubiquitious phrase seems to be "there's a module for that".

But just as often as it is easy to include a module, I often find in valuable to learn to use vanilla JavaScript to see how it might be possible to create the same functionality. Not only is it possible to reduce your runtime dependencies, but you also get a chance to learn about how to identify and create abstractions, perhaps even abstractions that don't exist yet. 

This post is going to describe my journey in creating two different async abstractions, a "interrupted request" and a "controlled request".

#### Caveat!

This code will use some of the latest features that are native in all modern browsers but likely don't exist in older implementations, so for a larger browser compatibility, you will likely need to use polyfills or transpiling to have the right coverage. But if you are working inside of a project using babel already, then you likely can include these functions and not worry about how they get transpiled away.

#### Extra Code!

If you see a function that doesn't exist, scroll to the end where I have included any extra functions that I might use in the code snippets that weren't defined up front.

## Motivation

I don't know about you guys, but I _hate_ including implementation details of an async request in the body of my main code. It feels wrong to expose so many details in the function, and because of that, I will often wrap my async requests in the new and exciting `async function`.

```javascript
async function makeRequest(url, params) {
  var fullUrl = url+paramterize(params);
  var newRequest = new Request(fullUrl);
  var response = await fetch(newRequest);
  var result = await response.json();
  return result;
}
```

Now, instead of having to include the details of a `fetch` in my functions, I can simply make a the request by including my url and parameters as the inputs to my function, and I will receive the result of the `json` representation as a promise. This may seem trivial, but now that we have abstracted away the details of making the request, we can start to build upon that abstraction and give it more power.

## First Example: Interruptable Request

Often times, we will make requests to a database or a server, but before that request comes back we get _another_ request for more up to date information. If this is an expensive request then we almost certainly want to abort the request. But tracking that as a state of our application gets bothersome very quickly. Fortunately, closure can come to the rescue. Our goal is to make a function that returns a function that can make a request, but also remember if it is currently making a request, and abort if necessary. Lets dive into this process in steps.

First we want to create a function...

```javascript
function interruptableRequestGen() {
}
```

This function should have a single "request" that it might make across any time it is invoked...

```javascript
function interruptableRequestGen() {
  var currentRequest = null;
  var controller = new AbortController();
  var signal = controller.signal;
}
```

I also created the controller that can abort the request, which you can read about more on MDN [here](https://developer.mozilla.org/en-US/docs/Web/API/AbortController/abort)

Now that we have created a closure around these variables, we can comfortably use them inside the function and not be concerned that outside execution contexts might tamper with our state in unknown ways...

```javascript
function interruptableRequestGen() {
  var currentRequest = null;
  var controller = new AbortController();
  var signal = controller.signal;
  return async function makeRequest(url, params) {
    if (currentRequest) {
      controller.abort();
    }
    
    var fullUrl = url + paramterize(params);
    currentRequest = new Request(fullUrl, { signal });
    var response = await fetch(currentRequest);
    var result = await response.json();
    
    currentRequest = null;
    return result;
  }
}
```

with that we have created an "interruptable request generator". Using it in our actually function, first we would invoke interruableRequest to create the function, the use it however we like...

```javascript
var makeInterruptableRequest = interruableRequestGen();
...
await makeInterruptableRequest(url, params) 
```

But we can live with certainty knowing that whenever we call `makeInterruptableRequest` with a `url` and `params` if there is a current request being made, it will abort before continuing, freeing up resources to be used both by the client and the server!

## Controlled Request

Sometimes, when you make a request, you might want to _ignore_ all subsequent requests to the same place until you have had a chance to deal with the first one. Perhaps you need to perform some complex calculation, or append something to the DOM, and until that is done, all requests should be ignored. In my own code, this kind of request is what I called a "controlled request" because the function that makes the request to the server or database gives up control to the end user on when it is safe to make another request.

Lets start by creating a function that will generate the requester, and create closure around a variable that tracks if it is responding to requests currently

```javascript
function controlledRequestGen() {
  var allowRequests = true;
}
```

now we can actually generate the requester in a similar fashion to the first request, but we don't need to worry about aborting anymore (because we always want to let a request finish)

```javascript
function controlledRequestGen() {
  var allowRequests = true;
  return async function makeRequest(url, params) {
    if (allowRequests) {
      allowRequests = false;
      var fullUrl = url + paramterize(params);
      var request = new Request(fullUrl);
      var response = await fetch(request);
      var result = response.json();
      
      return result;
    } else {
      return false;
    }
  }
}
```

Our function will either allow the request and make the call, or it will return false. But wait! we didn't actually give our user the functionality to tell our function it was ok to make another request! lets fix that now...

```javascript
function controlledRequestGen() {
  var allowRequests = true;
  return async function makeRequest(url, params) {
    if (allowRequests) {
      allowRequests = false;
      var fullUrl = url + paramterize(params);
      var request = new Request(fullUrl);
      var response = await fetch(request);
      var result = response.json();
      
      return {
        result,
        allowMoreRequests: function() {
          allowRequests = true;
        }
      }
    } else {
      return false;
    }
  }
}
```

This would work, but it would also give the user a function that they could _always_ call to allow more requests. That is probably alright, but we can do one better using our favoriate javascript programming principal, closure.

```javascript
function controlledRequestGen() {
  var allowRequests = true;
  
  var allow = function() {
    var called = false;
    return function() {
      if (!called) {
        allowRequests = true;
        called = true;
      }
  }_
  return async function makeRequest(url, params) {
    if (allowRequests) {
      allowRequests = false;
      var fullUrl = url + paramterize(params);
      var request = new Request(fullUrl);
      var response = await fetch(request);
      var result = response.json();
      
      return {
        result,
        allowMoreRequests: allow(),
      }
    } else {
      return false;
    }
  }
}
```

With that, we have done it! We have made a requester that will ignore all requests until the user has invoked the function to change its internal state. Because the closure can't be directly accessed, we know that its safe from tampering, and because the `allowMoreRequests` function can only be invoked once to change that state, everything will work as planned.

Creating this abstraction took a lot of work and understanding, but _using_ it is actually quite simple. All we have to do is make a use the generator to make the requester...

```javascript
var makeControlledRequest = controlledRequestGen();
...
var response = await makeControlledRequest(url, param);

if (response) {
  const { result, allowMoreRequests } = response;
  //Do some logic with the result
  
  allowMoreRequests();
}
```

we have taken a difficult async request, and abstracted it into just a few lines of readable, synchronous code.


## Whats the point!

The point is that everyday, we are using abstractions when we make code. In this case, it was 2 particular abstractions in relation to making async requests, but as you move through your codebases, you may find lots of abstractions that you are using with relation to database management, state control, or really anything involved with complex code.

Identifying, extracting and using these abstractions make us better coders, and makes it easier for others to come back and better read and understand the code we wrote today. And time making the codebase easier to read and understand is never time wasted.

So next time you find yourself include a complex bit of imperative code in your functions, take a step back and try and identify if you are using any core concepts that could be abstracted away from the details. It might just help you write better code.

## Extra functions!

I used a `paramaterize` function. If you are passing in parameters as and Object with key/value pairs with strings, to encode them into query params string you can do...

```javascript
function parameterize(params) {
  var keys = Object.keys(params);
  var paramStrings = keys.map( function(key) {
    return `${key}=${params[key]}`;
  });
  
  return "?"+paramStrings.join("&");
}
```
